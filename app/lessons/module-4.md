# Module 4 — Evaluation & Quality (lesson-by-lesson, v0.1)

**Objective:** stop guessing whether your agent is "good." By the end you can read a trace
to debug behavior, score an agent with LLM judges *and* deterministic code checks, build a
reusable evaluation dataset, bolt a hard rule-based **Guardian** gate in front of shipped
results, and close the loop by turning human feedback into ground truth.
**Prereqs:** Module 1 (a working tool-using agent + `mlflow.openai.autolog()` traces),
Module 2 (a packaged `ResponsesAgent`), `mlflow[databricks]>=3.1` installed (Module 0.5).
**Sandbox:** a Databricks notebook is easiest; the MLflow Trace + Evaluation UIs appear under
cell output and in the **Experiments** page. Evaluation datasets live in **Unity Catalog**.
(Experiment/labeling paths below use your **Databricks workspace** identity `you@example.com` —
not your Gmail — since `/Users/<email>/…` must be a workspace user.)
**Default tutor model:** `claude-haiku-4-5`.

> Format per lesson: **Concept** (why) → **Watch** (optional) → **Walkthrough** (run it) →
> **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.
>
> **API era note:** Verified on mlflow 3.14 — `mlflow.genai` is the correct path in the reference
> env, with **`mlflow.genai.evaluate(...)`** as the entry point and **scorer** objects from
> `mlflow.genai.scorers`. MLflow-2 workspaces use the older
> `mlflow.evaluate(model_type="databricks-agent")` API — the concepts transfer but the import paths differ.

---

### Lesson 4.1 — Why eval matters; MLflow Tracing deep dive
**Concept.** A printed answer tells you *what* the agent said, never *why*. "Looks right" is
not a quality bar — it doesn't scale past a handful of hand-checks, and it silently regresses
the moment you change a prompt, a model, or a tool. The fix is **measurement**, and
measurement starts with the **trace**: the structured record of every step the agent took
(LLM calls, tool calls, retrieval, arguments, outputs, latency, tokens). In Module 1 you saw
a trace as a debugger. Here it becomes the **substrate for evaluation** — judges and scorers
in 4.2 read the *trace*, not just the final string, so they can grade the retrieval step, the
tool arguments, and the answer independently.

**📺 Watch (optional, then DO):** [Build, Deploy & Assess a RAG App — Agent Framework + Agent Evaluation (product tour)](https://www.databricks.com/resources/demos/tours/data-science-and-ai/mosaic-ai-agent-framework-evaluation) (~10m) — the trace + eval UIs in one pass.

**Walkthrough.**
```python
import mlflow

# 1. Point MLflow at an experiment so traces have a home.
mlflow.set_experiment("/Users/you@example.com/lakepilot-eval")

# 2. Turn on automatic tracing for your agent's LLM calls (Module 1.6).
mlflow.openai.autolog()       # if you author with the OpenAI/Databricks client
# mlflow.langchain.autolog()  # if you built with LangGraph (Module 2.5)

# 3. Run the agent once, then pull the trace back programmatically.
#    (run_agent is your helper from Module 1.)
answer = run_agent("What is the square root of 429?")

traces = mlflow.search_traces(max_results=1)   # returns a pandas DataFrame
print(traces[["trace_id", "request", "response", "execution_time_ms"]])
```
Open the **MLflow Trace UI** under the cell (or **Experiments ▸ Traces**). Expand the tree:
`predict` → the LLM call → the tool call → the result. Every span carries inputs, outputs,
latency, and token counts.

You can also **manually trace** any function you want graded as its own span:
```python
@mlflow.trace(span_type="TOOL")
def lookup_inventory(sku: str) -> dict:
    ...
```

**Challenge.** Run your agent on one question, then `mlflow.search_traces(max_results=1)` and
print `execution_time_ms` plus the `trace_id`. Open that trace in the UI and expand it.
**STOP** and tell me the latency and how many spans the trace has.

**Check.** (1) Why is a trace, not the printed answer, the right input to an automated judge?
(2) Name two things a span records that you'd need to debug a *wrong tool argument*.

**Source.** [Evaluate and monitor AI agents](https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/)

---

### Lesson 4.2 — Agent Evaluation: LLM judges and metrics
**Concept.** You can't hand-grade hundreds of answers, and exact-string matching breaks the
moment wording differs. **LLM judges** solve this: a strong model reads the trace and the
(optional) ground truth and returns a **pass/fail score plus a written rationale** for *why* —
on dimensions like **correctness, relevance, groundedness, and safety**. Databricks ships
these as **built-in scorers** in `mlflow.genai.scorers`, run by `mlflow.genai.evaluate(...)`.
Crucially, some judges (Safety, RelevanceToQuery, Groundedness) need **no ground-truth labels**
— they grade the answer against the question/context alone — while Correctness needs an
`expected_response` or `expected_facts`.

**📺 Watch (optional):** the judges + scorecard appear in the [Agent Evaluation product tour](https://www.databricks.com/resources/demos/tours/data-science-and-ai/mosaic-ai-agent-framework-evaluation).

**Walkthrough.**
```python
import mlflow
from mlflow.genai.scorers import (
    Correctness, RelevanceToQuery, Safety, Guidelines,
    # retrieval-aware judges (use when your agent does RAG, Module 3.5):
    RetrievalGroundedness, RetrievalRelevance, RetrievalSufficiency,
)

# A tiny eval set: 'inputs' is what the agent receives; 'expectations' is ground truth.
eval_data = [
    {
        "inputs": {"question": "What is the square root of 429?"},
        "expectations": {"expected_response": "Approximately 20.712"},
    },
    {
        "inputs": {"question": "What is 17 factorial?"},
        "expectations": {"expected_facts": ["355687428096000"]},
    },
]

# predict_fn is your agent. Its parameter name(s) must match the 'inputs' keys.
def predict_fn(question: str) -> str:
    return run_agent(question)

results = mlflow.genai.evaluate(
    data=eval_data,
    predict_fn=predict_fn,
    scorers=[
        Correctness(),         # needs expected_response / expected_facts
        RelevanceToQuery(),    # no ground truth needed
        Safety(),              # no ground truth needed
        Guidelines(            # a custom LLM judge written in plain English
            name="concise_and_numeric",
            guidelines="The answer must state the numeric result and stay under 40 words.",
        ),
    ],
)
print(results.metrics)   # aggregated pass rates per scorer
```
Open the run in **Experiments**: each row shows per-question pass/fail **and the judge's
rationale**. That rationale is the whole point — it tells you *why* a row failed so you can
fix the agent, not just stare at a number.

> ⚠️ **verify:** the judge model differs by workspace. You can pin one, e.g.
> `Correctness(model="databricks:/databricks-claude-3-7-sonnet")` — confirm the endpoint name
> in your workspace's serving list (Module 1.1) before relying on it.

> Note: `RelevanceToQuery`, `RetrievalRelevance`, `Correctness`, `Guidelines`, `Safety`, and
> `ToolCallCorrectness` are all confirmed present in `mlflow.genai.scorers` (verified against mlflow 3.14).

**Challenge.** Run `mlflow.genai.evaluate` on the two-row set above with `Correctness()`,
`RelevanceToQuery()`, and `Safety()`. Open the run, find a row, and read the **Correctness
rationale**. **STOP** and paste the rationale plus the aggregate pass rate.

**Check.** (1) Which of these judges require a ground-truth label and which don't —
`Correctness`, `Safety`, `RelevanceToQuery`? (2) Why is the judge's *rationale* more useful
to you than its pass/fail bit?

**Source.** [Built-in scorers & LLM judges](https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/concepts/scorers) · [Built-in AI judges reference](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-evaluation/llm-judge-reference)

---

### Lesson 4.3 — Building an evaluation dataset
**Concept.** Ad-hoc inline lists (4.2) are fine for a smoke test, but real evaluation needs a
**versioned, governed, reusable** dataset — so every prompt tweak is scored against the *same*
yardstick and you can prove a change helped. On Databricks, an **MLflow evaluation dataset**
is a first-class object **stored as a Unity Catalog table**. The schema is two columns:
**`inputs`** (required — what the agent receives) and **`expectations`** (optional ground
truth, with *reserved keys* the judges look for: `expected_response` and `expected_facts` feed
**Correctness**, `guidelines` feeds **Guidelines**, `expected_retrieved_context` feeds the
retrieval scorers). The highest-value trick: **seed the dataset from real production traces**
instead of inventing questions.

**Walkthrough.**
```python
import mlflow.genai.datasets as datasets

UC_TABLE = "workspace.default.lakepilot_eval_set"

# 1. Create (or get) a UC-backed evaluation dataset. You need CREATE TABLE on the schema.
ds = datasets.create_dataset(UC_TABLE)   # use get_dataset(UC_TABLE) to reopen later

# 2. Add hand-authored records.
ds.merge_records([
    {
        "inputs": {"question": "What is the square root of 429?"},
        "expectations": {"expected_response": "Approximately 20.712"},
    },
    {
        "inputs": {"question": "What is 17 factorial?"},
        "expectations": {"expected_facts": ["355687428096000"]},
    },
])

# 3. Better: harvest real questions from production traces and merge them in.
traces_df = mlflow.search_traces(max_results=50)   # your logged agent runs
ds.merge_records(traces_df)   # inputs/outputs are pulled from each trace

# 4. Evaluate against the governed dataset (note: pass the dataset object as `data`).
import mlflow
from mlflow.genai.scorers import Correctness, Safety
results = mlflow.genai.evaluate(
    data=ds,
    predict_fn=lambda question: run_agent(question),
    scorers=[Correctness(), Safety()],
)
```
Records carry `inputs`/`expectations` plus system-managed metadata (`dataset_record_id`,
`source`, `create_time`). Records sourced from a trace keep **lineage** back to that trace, so
a failing eval row links straight to the original run in the Trace UI.

> ⚠️ **verify:** `merge_records` accepting a `search_traces` DataFrame directly (vs. needing a
> list of dicts) is version-dependent. If it errors, convert rows to
> `[{"inputs": {...}, "expectations": {...}}, ...]` first. Confirm against your installed
> `mlflow.genai.datasets` signature.

**Challenge.** Create a UC-backed dataset at `workspace.default.lakepilot_eval_set`, merge in
the two hand-authored records, then query `SELECT * FROM workspace.default.lakepilot_eval_set`
in a SQL cell to confirm the rows landed. **STOP** and tell me the row count and the column
names you see.

**Check.** (1) Why store the eval set in Unity Catalog instead of a Python list in the
notebook? (2) Which reserved `expectations` key would you set so the **Correctness** judge can
grade an answer against a list of must-include facts?

**Source.** [Evaluation datasets](https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/concepts/eval-datasets) · [Build, evaluate, deploy a retrieval agent](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-framework-notebook)

---

### Lesson 4.4 — Quality gates: a deterministic "Guardian" rule layer
**Concept.** LLM judges are *probabilistic* — superb for fuzzy quality ("is this grounded?"),
but you would never let a *judge* decide whether to ship an answer that leaks a credit-card
number or a forbidden phrase. Some rules must be **deterministic, fast, and free**: a credit
card pattern is present or it isn't; the response is over 500 words or it isn't; a required
disclaimer is present or it isn't. That's a **Guardian** — a pure-code check that runs as part
of evaluation (and, in production, *before results ship*). In MLflow you express it as a
**custom code-based scorer** via the `@scorer` decorator. It receives `inputs`, `outputs`,
`expectations`, and the full `trace`, and returns a primitive (`bool`/`int`/`float`/`str`) or a
rich **`Feedback(value=..., rationale=...)`**. No model call, so it's deterministic and runs in
the same `mlflow.genai.evaluate` pass alongside the judges — the contrast with 4.2 is the whole
lesson: **judges weigh, Guardians enforce.**

**Walkthrough.**
```python
import re
from mlflow.genai.scorers import scorer
from mlflow.entities import Feedback

# Guardian 1 — hard PII gate (regex, deterministic). Fails on any card-like number.
CARD_RE = re.compile(r"\b(?:\d[ -]*?){13,16}\b")

@scorer
def no_pii_leak(outputs: str) -> Feedback:
    hit = CARD_RE.search(outputs or "")
    return Feedback(
        value=(hit is None),                         # True = passes the gate
        rationale=("No card-like number found."
                   if hit is None
                   else f"BLOCKED: card-like pattern '{hit.group()}'"),
    )

# Guardian 2 — length budget (pure code, numeric pass/fail).
@scorer
def within_length_budget(outputs: str) -> bool:
    return len((outputs or "").split()) <= 500

# Guardian 3 — required-disclaimer gate, only when policy demands it.
@scorer
def has_disclaimer(outputs: str, expectations: dict) -> bool:
    if not (expectations or {}).get("requires_disclaimer"):
        return True                                   # gate not applicable -> pass
    return "not financial advice" in (outputs or "").lower()

# Run Guardians alongside the LLM judges in ONE evaluate pass.
import mlflow
from mlflow.genai.scorers import Safety, RelevanceToQuery

results = mlflow.genai.evaluate(
    data=ds,
    predict_fn=lambda question: run_agent(question),
    scorers=[Safety(), RelevanceToQuery(),          # probabilistic judges
             no_pii_leak, within_length_budget, has_disclaimer],  # deterministic Guardians
)
```
**Production use** (preview of Module 5/7): the same functions become a pre-ship gate inside
your `ResponsesAgent.predict` — if `no_pii_leak` returns `False`, you replace the answer with a
safe refusal *before returning it*, rather than merely scoring it after the fact.

```python
# Sketch of the in-line gate (conceptual):
def guard(answer: str) -> str:
    if CARD_RE.search(answer):
        return "I can't share that — it contained sensitive data."
    return answer
```

**Challenge.** Add a fourth Guardian `forbids_competitor_mention(outputs)` that fails if the
text contains any name in `["AcmeRival", "FooCorp"]` (case-insensitive), returning a
`Feedback` with a rationale. Run `mlflow.genai.evaluate` with it in the `scorers` list against
your dataset. **STOP** and tell me which rows it flagged and the rationale on one of them.

**Check.** (1) Give one quality dimension you'd trust a **Guardian** (code) for and one you'd
trust an **LLM judge** for — and say why each fits. (2) Why run Guardians inside
`mlflow.genai.evaluate` *and* inline in `predict` — what does each placement buy you?

**Source.** [Custom code-based scorers](https://mlflow.org/docs/latest/genai/eval-monitor/scorers/custom/) · [Scorers & judges concepts](https://docs.databricks.com/aws/en/mlflow3/genai/eval-monitor/concepts/scorers)

---

### Lesson 4.5 — Iterating from human feedback / labeled examples
**Concept.** Judges and Guardians get you far, but *what counts as a good answer* is often a
human call — and the cheapest ground truth is your own production traffic reviewed by an
expert. Databricks closes the loop with **labeling sessions** and the **Review App**: you
collect real traces, send them to domain experts who label them against a **label schema**
(e.g. "Poor/Fair/Good/Excellent," or "what facts *should* this answer contain"), then **sync
those labels into an evaluation dataset** (4.3) — which the **Correctness** judge then uses as
ground truth (4.2). That's the full virtuous loop: **trace → human label → eval dataset →
re-evaluate after a fix → measure the lift.**

**📺 Watch (optional):** the Review-App + feedback loop is demoed in the [Agent Evaluation product tour](https://www.databricks.com/resources/demos/tours/data-science-and-ai/mosaic-ai-agent-framework-evaluation).

**Walkthrough.**
```python
import mlflow
import mlflow.genai.labeling as labeling
import mlflow.genai.label_schemas as schemas

mlflow.set_experiment("/Users/you@example.com/lakepilot-eval")

# 1. Define what reviewers will judge. Custom categorical schema + a built-in one.
schemas.create_label_schema(
    name="response_quality",
    type="feedback",
    title="Rate the response quality",
    input=schemas.InputCategorical(options=["Poor", "Fair", "Good", "Excellent"]),
    overwrite=True,
)

# 2. Create a labeling session and assign your expert(s).
session = labeling.create_labeling_session(
    name="lakepilot_review_2026_06",
    assigned_users=["you@example.com"],
    label_schemas=["response_quality", schemas.EXPECTED_FACTS],  # EXPECTED_FACTS is built in
)

# 3. Add real traces for review (filter to recent agent runs).
traces_df = mlflow.search_traces(max_results=20)
session.add_traces(traces_df)

# 4. Experts open the Review App (link in the session / Experiments UI), label each trace.

# 5. Pull labels back, then sync the EXPECTED_FACTS labels into an eval dataset.
labeled = mlflow.search_traces(run_id=session.mlflow_run_id)
print(labeled[["trace_id", "assessments"]])
session.sync(to_dataset="workspace.default.lakepilot_eval_set")   # human labels -> ground truth

# 6. Re-evaluate against the now human-labeled dataset to measure your fix.
import mlflow.genai.datasets as datasets
from mlflow.genai.scorers import Correctness
ds = datasets.get_dataset("workspace.default.lakepilot_eval_set")
results = mlflow.genai.evaluate(
    data=ds, predict_fn=lambda question: run_agent(question), scorers=[Correctness()],
)
```
Assigned users automatically get WRITE on the experiment. Keep sessions to ~25–100 traces with
clear, date-stamped names.

> ⚠️ **verify:** `session.sync(to_dataset=...)` argument name and whether it appends vs.
> overwrites varies across MLflow 3 minors. Confirm with `help(session.sync)` before pointing
> it at a dataset others depend on.

**Challenge.** Create the `response_quality` label schema and a labeling session, add ~5
traces with `session.add_traces(...)`, then open the **Review App** and label one trace.
**STOP** and tell me the session's `mlflow_run_id` and whether your label shows up in
`mlflow.search_traces(run_id=...)` under `assessments`.

**Check.** (1) Trace through the full loop in order: where does a *human* label enter, and
which *judge* consumes it later? (2) Why are human labels worth the effort when you already
have automated judges — what can a person catch that `Correctness`/`Safety` cannot?

**Source.** [Create & manage labeling sessions](https://docs.databricks.com/aws/en/mlflow3/genai/human-feedback/concepts/labeling-sessions) · [Labeling schemas](https://docs.databricks.com/aws/en/mlflow3/genai/human-feedback/concepts/labeling-schemas)

---

## Module 4 wrap
By here the learner can: read a trace as the substrate for evaluation, score an agent with
built-in **LLM judges** (Correctness, Relevance, Groundedness, Safety) *and* **Guidelines**
judges, build a **Unity-Catalog-backed evaluation dataset** (hand-authored or harvested from
traces), enforce hard rules with deterministic **Guardian** code-scorers via `@scorer`, and
run the **human-feedback loop** (labeling sessions → Review App → synced ground truth →
re-evaluate). You now have a number, a reason, and a gate — not a vibe. Next:
**Module 5 — Deployment** — log & register the agent to Unity Catalog, deploy to Model Serving,
and chat with it in the AI Playground, carrying these scorers forward as production monitors.
