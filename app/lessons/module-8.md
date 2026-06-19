# Module 8 — Capstone: Build & Ship Your Own (lesson-by-lesson, v0.1)

**Objective:** build, evaluate, deploy, and hand off **one real agent end to end** — your own — reusing
every technique from Modules 1–7. By the end you own a governed, traced, deployed agent a real person can use.
**Prereqs:** Modules 1–7 (LLM calls + UC-function tools, ResponsesAgent authoring, Genie/RAG/MCP tools,
Agent Evaluation, deployment to Model Serving + Databricks Apps, the Supervisor pattern, governance).
**Sandbox:** a Databricks notebook for build/eval/deploy; SQL Editor for UC functions/grants; Agent Bricks UI for the supervisor.
**Default tutor model:** `claude-haiku-4-5` (bump the *agent's* own model to Sonnet only if eval says you need it — Lesson 8.4 / 7.4).

> Format per lesson: **Concept** (why) → **Watch** (optional) → **Walkthrough** (build it, referencing the
> earlier lesson it reuses) → **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.
>
> **The running project (pick once, carry it through all 6 lessons).** Default suggestion: a **Procurement Ops
> Assistant** — answers "what's our stock and who supplies it?", extracts line items from a vendor quote, and
> drafts a reorder email. If procurement isn't your world, swap in a **Personal Ops Assistant** (calendar/expenses/
> notes) — but pick *one* and don't change horses. Every lesson below builds the *same* agent one layer at a time.

---

### Lesson 8.1 — Design from first principles: what job, what tools, what data
**Concept.** Most agents fail before a line of code — they're scoped as "an assistant that helps with procurement"
(a vibe, not a job). A shippable agent starts from a **single concrete job** with a **done condition**, then works
backward to the *minimum* tools and data needed to do it. This is the supervisor's "classify → route → synthesize"
shape (6.1) read in reverse: name the routes first, and each route tells you exactly one tool to build.

**Walkthrough.** Write a one-page **spec** (a notebook markdown cell is fine) with four fields:

```text
JOB        : Help a buyer reorder stock — answer stock/supplier questions, pull line items
             from a quote PDF, and draft the reorder email. Done = a reorder email ready to send.
USERS      : One real buyer (you'll name them in 8.6). They type plain English, not SQL.
ROUTES     : (a) "what do we have / who supplies it?"   -> data question  -> Genie
             (b) "pull the items out of this quote"      -> extraction     -> UC function tool
             (c) "draft the reorder email"               -> generation     -> UC function tool
DATA       : a `stock` table + a `suppliers` table (you'll make these in 8.2); quote text pasted in.
NON-GOALS  : no placing real orders, no sending real email, no payment. (Stays a draft — see 7.x safety.)
```

Three routes → three tools. That mapping *is* your build list for 8.2, and it's the same inventory a supervisor
coordinates (6.2): a Genie Space + two UC functions. Keep it to three — a 12-tool capstone is how people never ship.

**Challenge.** Fill in all four fields for *your* chosen project (procurement default, or your own). Force yourself
to write the **Done =** clause and at least one **NON-GOAL**. **STOP** and paste me your JOB line and your three ROUTES.

**Check.** (1) Why does naming the *routes* first hand you the tool list for free? (2) What does a missing **Done =**
clause cost you later — in 8.4 specifically? (Hint: you can't evaluate "helpfulness," only "did it produce the email.")

**Source.** Module 6.1–6.2 (supervisor pattern + what it coordinates); the procurement worked example in 6.8.

---

### Lesson 8.2 — Build the tools (UC functions / Genie / MCP)
**Concept.** Tools are the only thing that makes an agent *do* rather than *talk* (1.2). You'll now build the three
from your 8.1 routes as **governed Unity Catalog objects** — exactly the pattern from 1.7 (author your own UDF) and
3.2 (SQL + Python UC-function tools), plus a Genie Space from 3.3. Build each tool *and test it standalone* before any
agent sees it; a tool you haven't called by hand is a tool you can't trust.

**Walkthrough.** Pick a catalog/schema you control (e.g. `workspace.procurement`).

**1 — Data + the Genie route (3.3).** Create the tables, then build a Genie Space over them.
```sql
CREATE SCHEMA IF NOT EXISTS workspace.procurement;
CREATE OR REPLACE TABLE workspace.procurement.stock AS
  SELECT * FROM VALUES ('SKU-100','Copier paper A4',12,40),('SKU-200','Toner black',3,15)
  AS t(sku, item, on_hand, reorder_point);
CREATE OR REPLACE TABLE workspace.procurement.suppliers AS
  SELECT * FROM VALUES ('SKU-100','Acme Paper','orders@acme.test',3),('SKU-200','InkCo','sales@inkco.test',5)
  AS t(sku, supplier, email, lead_days);
```
Then in the UI: **New ▸ Genie Space**, add both tables, and test a question — *"which SKUs are at or below their
reorder point?"* (as you did in 3.3). This is route (a).

**2 — The extraction tool (UC Python function, as in 1.7 / 3.2).** Route (b):
```sql
CREATE OR REPLACE FUNCTION workspace.procurement.extract_line_items(quote_text STRING)
RETURNS STRING LANGUAGE PYTHON AS $$
import re, json
items = []
for line in quote_text.splitlines():
    m = re.search(r'(SKU-\d+).*?qty\s*(\d+)', line, re.I)
    if m:
        items.append({"sku": m.group(1).upper(), "qty": int(m.group(2))})
return json.dumps(items)
$$;
```

**3 — The email-draft tool (UC SQL function, as in 3.2).** Route (c):
```sql
CREATE OR REPLACE FUNCTION workspace.procurement.draft_reorder_email(supplier STRING, sku STRING, qty INT)
RETURNS STRING
RETURN concat('To: ', supplier, '\nSubject: Reorder ', sku,
              '\n\nHi,\nPlease send ', cast(qty AS STRING), ' units of ', sku,
              '. Confirm price and lead time.\nThanks.');
```

Smoke-test each one *by hand* before wiring (the 8.2 discipline):
```sql
SELECT workspace.procurement.extract_line_items('Line1 SKU-200 qty 12');   -- [{"sku":"SKU-200","qty":12}]
SELECT workspace.procurement.draft_reorder_email('InkCo','SKU-200',12);     -- a ready email body
```
*(MCP option, from 3.7–3.8: if your real data lives behind an MCP server — a vendor catalog, an internal API —
register that server instead of a UC function for that one route. Same agent-facing shape; pick UC functions here
unless you already have an MCP endpoint.)*

**Challenge.** Build all three (Genie Space + two UC functions) in *your* catalog and run the two `SELECT` smoke
tests. **STOP** and paste the JSON the extraction function returned for one test line.

**Check.** (1) Why test each tool standalone *before* attaching it to the agent? (2) When would you reach for an
**MCP server** (3.7) instead of authoring a UC function for a route?

**Source.** Lessons 1.7 (author your own UDF), 3.2 (UC-function tools), 3.3 (Genie Space), 3.7–3.8 (MCP).

---

### Lesson 8.3 — Wire the supervisor + instructions
**Concept.** You have three working tools; now you need **one brain that routes to them** (6.1). The supervisor
classifies the user's message, picks the right tool/sub-agent, runs it, and synthesizes the answer — and the single
biggest quality lever isn't the model, it's the **Instructions** (the routing system prompt) and each tool's
**Description** (6.4). Vague descriptions make the supervisor guess; precise ones make routing deterministic.

**Walkthrough.** In **Agent Bricks ▸ New ▸ Agent ▸ Supervisor** (6.3), attach the three resources from 8.2: the
**Genie Space**, `extract_line_items`, and `draft_reorder_email`. Then write the two things that matter:

**Instructions (the routing prompt, 6.4):**
```text
You are a procurement assistant. Route each request:
- Stock levels, what's on hand, who supplies a SKU, what's below reorder point -> the Genie Space.
- "pull / extract the items from this quote" (user pasted quote text) -> extract_line_items.
- "draft / write the reorder email" -> draft_reorder_email, using the supplier from the Genie Space.
Never place an order or send email. Always return the email as a DRAFT for the human to send.
When asked to reorder, chain: Genie (find supplier for the SKU) -> draft_reorder_email.
```

**Tool Descriptions (6.4)** — one precise line each, e.g. *`extract_line_items`: "Parse pasted vendor-quote text
into a JSON list of {sku, qty}. Use when the user pastes a quote and wants the items."*

That `Never place an order... return a DRAFT` line is your **non-goal from 8.1** wired in as a guardrail — the
deterministic-rule instinct from 4.4. Build the agent, then in the AI Playground (6.5) try one message per route to
confirm each lands on the right tool.

*(Prefer code over the UI? The same three tools attach to a `ResponsesAgent` (2.1–2.3) with `UCFunctionToolkit`
(1.3) exactly as in Module 2 — the supervisor UI is the fast path; the ResponsesAgent is the portable one. Either
produces a loggable agent for 8.5.)*

**Challenge.** Wire the supervisor, paste the Instructions above (adapted to your project), and in the Playground
send one message that should hit *each* route. **STOP** and tell me which route mis-fired first (there's almost
always one) — that's your 8.4 starting point.

**Check.** (1) Why is a tool's **Description** a routing decision, not documentation? (2) Where did your 8.1
**NON-GOAL** end up living in the running agent, and why there?

**Source.** Lessons 6.3 (create a Supervisor), 6.4 (Instructions + Descriptions), 6.5 (Playground); 2.1–2.3 for the
code path; 4.4 for the guardrail instinct.

---

### Lesson 8.4 — Evaluate against your own dataset
**Concept.** "It worked when I tried it" is not shipped. Evaluation turns your agent from a demo into something you
can *change without fear* — every prompt tweak gets scored against the same questions (4.1–4.2). The catch: generic
benchmarks won't tell you if *your* agent does *your* job. You need **your own dataset**, built straight from the
routes you defined in 8.1 (4.3) — and your 8.1 **Done =** clause becomes the thing you actually measure.

**Walkthrough.** Build a tiny eval set — one row per route plus an edge case (4.3). A handful of good rows beats a
hundred vague ones.
```python
import mlflow, pandas as pd
eval_df = pd.DataFrame([
  {"request": "Which SKUs are at or below reorder point?",        "expected_route": "genie"},
  {"request": "Pull the items from: Line1 SKU-200 qty 12",        "expected_route": "extract"},
  {"request": "Draft the reorder email for SKU-200, 12 units",    "expected_route": "draft"},
  {"request": "Place the order for me",                           "expected_route": "refuse"},  # NON-GOAL guardrail
])

results = mlflow.evaluate(
    model="endpoints:/<your-deployed-or-logged-agent>",   # or a predict_fn wrapping your agent
    data=eval_df,
    model_type="databricks-agent",                        # Agent Evaluation LLM judges (4.2)
)
print(results.metrics)
```
Open the eval run's traces (the same trace UI from 1.6) and read *why* a row failed — wrong tool chosen, empty Genie
result, the refuse-case that didn't refuse. Fix it where it lives: a routing miss → tighten the **Instructions/
Description** (8.3 / 6.4); a bad tool output → fix the **UC function** (8.2); a weak final answer → consider bumping
the agent's model (7.4). Re-run. That loop — change one thing, re-score — *is* the job.

**Challenge.** Build your 4-row eval set (one per route + your guardrail row), run `mlflow.evaluate`, and iterate
until the guardrail/refuse row passes. **STOP** and report which row failed first and the one change that fixed it.

**Check.** (1) Why must the eval set come from *your* 8.1 routes rather than a generic benchmark? (2) A row fails —
how do you decide whether to fix the **instructions**, the **tool**, or the **model**? (Trace first — 1.6/4.1.)

**Source.** Lessons 4.1–4.3 (eval + datasets), 4.4 (guardrails), 1.6 (traces), 7.4 (model choice as a fix).

---

### Lesson 8.5 — Deploy (endpoint + Databricks App)
**Concept.** A logged agent helps no one until it's *serving*. Deployment is the same two-step from Module 5: register
the agent to Unity Catalog (5.1), then deploy it to a Model Serving endpoint (5.3) with auto-auth resources (5.2) so
it can reach its own tools in production — and finally front it with a Databricks App (5.5) so a human gets a chat box,
not a curl command. The trap that bites everyone: the deployed agent runs as a *different identity* than you, so it
needs `EXECUTE` granted on every tool (1.8 / 5.2).

**Walkthrough.** From a notebook (the 5.x path):
```python
import mlflow
from databricks import agents
from databricks.agents import deploy
from mlflow.models.resources import DatabricksServingEndpoint, DatabricksFunction, DatabricksGenieSpace

mlflow.set_registry_uri("databricks-uc")   # register to UC, not the workspace registry (5.1)

# Declare what the agent must reach so deploy can auto-provision auth (5.2):
resources = [
    DatabricksGenieSpace(genie_space_id="<your-space-id>"),
    DatabricksFunction(function_name="workspace.procurement.extract_line_items"),
    DatabricksFunction(function_name="workspace.procurement.draft_reorder_email"),
]
# (log_model with these resources as in 2.3/5.1, register, then:)
agents.deploy(model_name="workspace.procurement.reorder_agent", model_version="1", scale_to_zero=True)
```
Grant the deployed identity `EXECUTE` on both functions and access to the Genie Space (1.8) — skip this and you'll
get exactly the `UNRESOLVED_ROUTINE` mask from 1.8 in production. Chat with it in the **AI Playground**, hit "Get
code" to confirm the endpoint answers (5.4), then host the **Databricks App** chat UI (5.5) so your 8.6 user has a
real front door. Keep `scale_to_zero=True` so an idle capstone costs ~nothing (5.7 / 7.4).

**Challenge.** Register and deploy your agent, grant the deployed identity `EXECUTE` on your tools, and open it once
in the AI Playground. **STOP** and paste the endpoint name and confirm one real answer came back through the endpoint
(not just the notebook).

**Check.** (1) Why does the *deployed* agent need `EXECUTE` granted even though you, the author, already have it
(callback to 1.8)? (2) What does declaring `resources=[...]` at deploy time actually buy you (5.2)?

**Source.** Lessons 5.1 (register to UC), 5.2 (resources/auto-auth), 5.3 (deploy), 5.4 (Playground), 5.5 (Databricks
App), 1.8 (the EXECUTE-grant trap), 5.7/7.4 (scale-to-zero cost).

---

### Lesson 8.6 — Hand it to a real user and watch the trace
**Concept.** The capstone isn't done when *you* can use it — it's done when *someone else* can, and you can *see what
happened*. Production monitoring (7.3) is just tracing (1.6) at scale: every real request lands in an inference table
you can read, so a confused user becomes a fixable trace instead of a shrug. This is the loop that keeps the agent
alive after launch — watch real traces, harvest the failures into new eval rows (4.5), iterate.

**Walkthrough.** Name the real user from your 8.1 spec and give them the **Databricks App** URL (5.5). Ask them to do
*one real task* end to end — e.g. "find what's below reorder point, then draft me the email for it" — in their own
words, not yours. While they use it:
- Open the **inference / payload-logging table** for the endpoint (7.3) and watch requests arrive.
- Click into a request's **trace** (1.6): which route fired, which tool ran, what Genie returned, the final draft.
- When something's off (wrong supplier, awkward email, a route miss), copy that real request into your 8.4
  **eval set** as a new row (4.5) — your dataset now grows from reality, not your imagination — and re-run 8.4's loop.

That's the whole production lifecycle in one sitting: **real user → trace → new eval row → fix → re-deploy.** You've
now reused every module: tools (1/3), authoring (2), Genie/MCP (3), eval (4), deploy (5), supervisor (6), governance
+ monitoring (7).

*(📺 Optional, and a great capstone artifact — record your **own** 2–3 minute demo video of the finished agent doing
the real task, the way the course's video layer suggests for signature moments. It's the single best thing to show
when you "sell it" below.)*

**Challenge.** Hand the App to your real user for one task, then open the trace of *their* request. **STOP** and tell
me: which route fired, and one thing you'd change based on what you saw in *their* trace (not yours).

**Check.** (1) Why is a real user's trace worth more than ten of your own test runs? (2) How does a production failure
become a permanent improvement rather than a one-off fix (callback to 4.5)?

**Source.** Lessons 7.3 (monitor/inference tables), 1.6 (traces), 4.5 (iterate from real examples), 5.5 (the App as
the front door).

---

## Module 8 wrap — Course wrap: ship it, sell it, keep it current
You built **one real agent end to end**: scoped to a single job (8.1), backed by three governed tools (8.2),
routed by a supervisor with tight instructions (8.3), scored against your own dataset (8.4), deployed behind a
real chat UI (8.5), and handed to a real user whose traces you watched (8.6). That's the entire Databricks
agent stack — Modules 1–7 — exercised on something that's actually yours.

**What to do next:**
- **Ship it.** It's already live behind a Databricks App with scale-to-zero (5.5/5.7). Run the **go-to-prod
  checklist** (7.5) once more — grants (1.8), secrets/scopes (7.2), monitoring on (7.3), cost set (7.4) — and tell
  your one user it's theirs to keep using.
- **Sell it.** Record the 2–3 minute demo from 8.6 and write a one-paragraph "what job does this do" (your 8.1 JOB
  line, basically). A traced, evaluated, deployed agent with a demo video is a portfolio piece, not a tutorial exercise.
- **Keep it current.** Leave the 8.6 loop running: real traces (7.3) → new eval rows (4.5) → re-deploy. An agent
  that doesn't ingest its own failures rots; one that does gets quietly better every week.

You now know how to build, evaluate, deploy, and *maintain* an AI agent on Databricks — and, when a built-in piece
won't cooperate, how to build the piece yourself (1.7). That last skill is the one that makes you dangerous. Go build
the next one.
