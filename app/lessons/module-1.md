# Module 1 — Your First Agent (lesson-by-lesson, v0.1)

**Objective:** build and test a working tool-using agent end to end, and walk away able to
make your *own* tool when a built-in one fails.
**Prereqs:** Module 0 (workspace auth, a notebook or the local 3.11 venv, a running serverless/SQL warehouse).
**Sandbox:** a Databricks notebook is easiest; SQL snippets run in a SQL cell or the SQL Editor.
**Default tutor model:** `claude-haiku-4-5`.

> Format per lesson: **Concept** (why) → **Watch** (optional) → **Walkthrough** (run it) →
> **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.

---

### Lesson 1.1 — Call your first Databricks-served LLM
**Concept.** On Databricks you don't call Anthropic/OpenAI directly — you call models *served inside your workspace* through the Foundation Model API, using an OpenAI-compatible client (`DatabricksOpenAI`). Same `chat.completions` shape you may know, but auth, billing, and governance all run through Databricks. This matters because **an agent is just an LLM call in a loop with tools** — so the LLM call is the bedrock everything else sits on.

**📺 Watch (optional, then DO):** [AI Agents on Databricks in 5 Minutes](https://www.databricks.com/resources/demos/videos/ai-agents-on-mosaic-ai-in-5-minutes) (~5m)

**Walkthrough.**
```python
from databricks_openai import DatabricksOpenAI
client = DatabricksOpenAI()
r = client.chat.completions.create(
    model="databricks-claude-3-7-sonnet",
    messages=[{"role": "user", "content": "In one sentence, what is an AI agent?"}],
)
print(r.choices[0].message.content)
```
If this returns `403 ... required scopes: model-serving`, your token is missing the **model-serving** scope — flag it, we fix tokens in Lesson 1.8 / Module 0.4.

**Challenge.** Swap the model to `databricks-meta-llama-3-3-70b-instruct` and ask the same question. Notice the different voice. Then **STOP** and tell me which serving endpoints your workspace actually has available.

**Check.** (1) When you call `DatabricksOpenAI`, where does the model physically run — your laptop, Anthropic, or your Databricks workspace? (2) Why route through the Databricks-served model instead of calling Anthropic directly?

**Source.** [Get started with AI agents](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-quickstart)

---

### Lesson 1.2 — Tools 101: what an agent can actually DO
**Concept.** An LLM alone only *talks*. A **tool** lets it *act* — run code, query a table, hit an API. On Databricks, tools are **Unity Catalog functions**: governed, reusable, and callable by any agent. The simplest powerful one is the built-in code interpreter **`system.ai.python_exec`** — it runs Python and returns stdout, so the model can *compute* instead of guessing.

**Walkthrough.** In a SQL cell:
```sql
DESCRIBE FUNCTION EXTENDED system.ai.python_exec;
```
Note: input `code STRING`, returns `STRING` (the captured stdout).

**Challenge.** Run `SELECT system.ai.python_exec('print(2**10)');`. Did you get `1024`? (If you get `UNRESOLVED_ROUTINE` instead — hold that thought, it's Lesson 1.7.) **STOP** and tell me what came back.

**Check.** (1) What's the real difference between the LLM answering "what's 2^10?" from memory vs. calling `python_exec`? (2) Why is a UC function a good home for a tool? (Think reuse + governance.)

**Source.** [AI agent tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 1.3 — Wire a tool to the LLM
**Concept.** The model can't use a tool it can't *see*. You describe the tool to it as a JSON schema (name, parameters, description); the model then *chooses* whether to call it. `UCFunctionToolkit` generates that schema from a UC function automatically.

**Walkthrough.**
```python
from databricks_openai import UCFunctionToolkit, DatabricksFunctionClient
client = DatabricksFunctionClient()
builtin_tools = UCFunctionToolkit(function_names=["system.ai.python_exec"], client=client).tools
for t in builtin_tools:
    t["function"].pop("strict", None)   # compatibility tweak
print(builtin_tools[0]["function"]["name"])   # -> system__ai__python_exec
```
**Notice the name transform:** dots become double underscores (`system.ai.python_exec` → `system__ai__python_exec`). Remember this — it bites in Lesson 1.7.

**Challenge.** Print the full tool schema and find where the `code` parameter and its description live. **STOP** and paste that parameter description.

**Check.** (1) Why does the model need a schema rather than just the function itself? (2) What does the tool *name* become, and why will that matter when we route tool calls?

**Source.** [Build agents on Databricks](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 1.4 — The agent loop (the whole secret)
**Concept.** An "agent" is this loop, nothing more:
1. Send the prompt **+ tools** to the model.
2. Model replies with either a final answer **or** a tool call.
3. If a tool call → *you* run the tool and feed the result back.
4. Model uses the result to produce the final answer.
Frameworks (LangGraph, ResponsesAgent) dress this up, but **this loop is the engine.**

**Walkthrough (raw loop, chat-completions style).**
```python
openai_client = DatabricksOpenAI()
LLM = "databricks-claude-3-7-sonnet"
messages = [{"role": "user", "content": "What is the square root of 429? Use the tool."}]
resp = openai_client.chat.completions.create(model=LLM, messages=messages, tools=builtin_tools)
msg = resp.choices[0].message
print(msg.tool_calls)   # the model's decision to call the tool (and the code it wrote)
# next step: run each tool_call via DatabricksFunctionClient().execute_function(...),
# append the result as a tool message, and call create() again for the final answer.
```
(The streamed, production-shaped version lives in Module 2 — here you see the bare mechanics.)

**Challenge.** Print `resp.choices[0].message.tool_calls`. Did the model decide to call the tool, and what `code` did it generate? **STOP** and paste it.

**Check.** (1) Who decides whether a tool gets called — your code or the model? (2) Name the 4 stages of the loop.

**Source.** [Build agents on Databricks](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 1.5 — Run it end to end
**Concept.** Now run a real agent and watch it reason → call the tool → answer. The classic smoke test is **"what's the square root of 429?"** — a question the model *shouldn't* answer from memory, which forces a genuine tool call.

**Walkthrough.** Open the **"Build your first AI agent"** quickstart notebook (Workspace sample) and run its setup + test cells, or use your `run_agent` helper. Expect the output to contain a `function_call` to the code interpreter with `import math; print(math.sqrt(429))`, then a tool result ≈ **20.712**.

**Challenge.** Change the question to **"what is 17 factorial?"** and run again. Confirm the agent *computes* it (and doesn't hallucinate a number). **STOP** and tell me the value it returned.

**Check.** (1) Why choose a question the model can't answer from memory for a tool test? (2) Two distinct things show up in the output — the model's tool *call* and the tool's *result*. Which is which?

**Source.** [Get started with AI agents](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-quickstart)

---

### Lesson 1.6 — See inside it: MLflow Tracing
**Concept.** When an agent misbehaves, a printed answer won't tell you *why*. A **trace** shows what it actually did — which model, which tool, what arguments, what came back, how long, how many tokens. `mlflow.openai.autolog()` captures all of it. This is both your debugger **and** the foundation for evaluation (Module 4).

**📺 Watch (optional):** the trace UI appears in [AI Agents in 5 Minutes](https://www.databricks.com/resources/demos/videos/ai-agents-on-mosaic-ai-in-5-minutes).

**Walkthrough.**
```python
import mlflow
mlflow.openai.autolog()
# ...run the agent again...
```
Open the **MLflow Trace UI** under the cell output (or via Experiments). Expand the tree: `predict` → the LLM call → the tool call → the output.

**Challenge.** In the trace, find: (a) the model used, (b) the tool name, (c) the exact `code` argument the model wrote, (d) the latency and token count. **STOP** and report the latency.

**Check.** (1) What can a trace tell you that the printed answer can't? (2) Why is tracing the foundation for *evaluating* an agent later?

**Source.** [Build agents on Databricks](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 1.7 — When the built-in tool won't run (the real-world lesson)
**Concept.** Sometimes `system.ai.python_exec` exists in the catalog but its execution backend isn't live in your workspace, and a tool call fails with `[UNRESOLVED_ROUTINE] Cannot resolve routine system.ai.python_exec`. Don't panic — the fix teaches the single most useful agent skill: **author your own tool** as a Unity Catalog Python UDF in a catalog you control, and point the agent at it.

**Walkthrough.**
1. Diagnose: `SELECT system.ai.python_exec('print(1)');` → if `UNRESOLVED_ROUTINE`, it's a backend/enablement gap, not your code.
2. Build your own equivalent:
   ```sql
   CREATE OR REPLACE FUNCTION workspace.default.python_exec(code STRING)
   RETURNS STRING LANGUAGE PYTHON AS $$
   import sys
   from io import StringIO
   buf, old = StringIO(), sys.stdout
   sys.stdout = buf
   try:
       exec(code, {})
   except Exception as e:
       sys.stdout = old
       return f"Error: {type(e).__name__}: {e}"
   finally:
       sys.stdout = old
   return buf.getvalue()
   $$;
   ```
3. Test it: `SELECT workspace.default.python_exec('import math; print(math.sqrt(429))');` → `20.712...`
4. Point the agent at it: set `function_names=["workspace.default.python_exec"]` **and** update the tool-name check in `call_tool` to `workspace__default__python_exec` (dots → double underscores, from Lesson 1.3).

**Challenge.** Create the function, test it, and confirm it returns the square root. **STOP** and tell me it worked.

**Check.** (1) Does `UNRESOLVED_ROUTINE` mean the function is missing, you lack permission, or the backend isn't live — and how would you tell them apart? (2) When you rename the tool, what *else* must change in the agent's `call_tool` logic, and why?

**Source.** experienced live on 2026-06-17; concept per [AI agent tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 1.8 — Permissions & grants (where most "agent broke" mysteries actually live)
**Concept.** Tools are governed objects. To call one, the calling identity needs three things: **`USE CATALOG`** + **`USE SCHEMA`** + **`EXECUTE`** on the function. A huge share of "the agent can't call its tool" failures are really *permission* failures wearing an `UNRESOLVED_ROUTINE` mask — Unity Catalog hides objects you can't access.

**Walkthrough.**
```sql
SHOW GRANTS ON FUNCTION workspace.default.python_exec;
GRANT EXECUTE ON FUNCTION workspace.default.python_exec TO `account users`;
-- if a principal still can't reach it:
GRANT USE CATALOG ON CATALOG workspace TO `account users`;
GRANT USE SCHEMA ON SCHEMA workspace.default TO `account users`;
```

**Challenge.** Run `SHOW GRANTS` on your function — who has `EXECUTE`? Grant it to `account users`, then re-run `SHOW GRANTS` to confirm. **STOP** and paste the grants.

**Check.** (1) Why does a *deployed* agent need `EXECUTE` granted even though you, the author, already have it? (2) What three privileges does calling a UC-function tool require?

**Source.** experienced live on 2026-06-17; [Supervisor Agent permissions](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

## Module 1 wrap
By here the learner can: call a served LLM, understand tools as UC functions, wire a tool to the model, run the full agent loop, read a trace, **build their own tool when a built-in fails**, and reason about the permissions behind it. Next: **Module 2 — authoring the agent properly** (ResponsesAgent, packaging, LangGraph/OpenAI SDK) so it can be logged, evaluated, and deployed.
