# Module 2 — Authoring Agents Properly (lesson-by-lesson, v0.1)

**Objective:** wrap your Module 1 agent in MLflow's `ResponsesAgent` interface, give it
streaming, package it as a logged-from-code `agent.py`, and meet the two frameworks
(LangGraph, OpenAI Agents SDK) Databricks ships templates for — so the agent is now
loggable, evaluable, and deployable instead of a one-off notebook cell.
**Prereqs:** Module 1 (a working tool-using agent loop, `mlflow.openai.autolog()`, UC-function tools).
**Sandbox:** a Databricks notebook is easiest (you'll use `%%writefile` and `dbutils`); the local 3.11 venv works for the pure-Python parts.
**Default tutor model:** `claude-haiku-4-5`.

> 📺 **Video note:** Module 2 has **no standalone official video** — the ResponsesAgent/LangGraph
> authoring flow is a known **"record-your-own" gap clip** (see CURRICULUM "Gaps to fill later").
> So lessons below omit Watch links by design; don't go hunting for one.

> Format per lesson: **Concept** (why) → **Watch** (only if a real video applies) →
> **Walkthrough** (run it) → **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.

---

### Lesson 2.1 — The `ResponsesAgent` interface (why a standard shape)
**Concept.** In Module 1 your agent was a loose `run_agent()` function — fine for a demo, useless for production, because Databricks' logging, tracing, evaluation, and deployment machinery don't know how to call it. MLflow's **`ResponsesAgent`** is the standard contract that fixes this: you wrap *any* framework (raw OpenAI loop, LangGraph, LlamaIndex) in one class, and in return you get automatic compatibility with the **AI Playground**, **Agent Evaluation**, **MLflow Tracing**, and one-command deployment. Databricks explicitly recommends it: "`ResponsesAgent` lets you build agents with any third-party framework, then integrate it with Databricks AI features." Think of it as the USB-C port for agents.

**Walkthrough.** The contract is two methods over typed request/response objects:
```python
from mlflow.pyfunc import ResponsesAgent
from mlflow.types.responses import (
    ResponsesAgentRequest,
    ResponsesAgentResponse,
    ResponsesAgentStreamEvent,
)
from typing import Generator

class MyAgent(ResponsesAgent):
    def predict(self, request: ResponsesAgentRequest) -> ResponsesAgentResponse:
        ...
    def predict_stream(
        self, request: ResponsesAgentRequest
    ) -> Generator[ResponsesAgentStreamEvent, None, None]:
        ...
```
The base class also hands you **output-item helpers** so you don't hand-build JSON: `self.create_text_output_item(text, id)`, `self.create_function_call_item(id, call_id, name, arguments)`, `self.create_function_call_output_item(call_id, output)`, and (streaming only) `self.create_text_delta(delta, item_id)`. Requires `pydantic>=2`.

**Challenge.** In a notebook, run the import block above and define an empty `class MyAgent(ResponsesAgent): pass`. Does it import cleanly, or does it complain that abstract methods are unimplemented? **STOP** and tell me what happened when you instantiated `MyAgent()`.

**Check.** (1) Name two Databricks capabilities you get "for free" once your agent is a `ResponsesAgent`. (2) Why is a *standard interface* worth more than your hand-rolled `run_agent()` even though both produce the same answer?

**Source.** [Author an AI agent](https://docs.databricks.com/aws/en/generative-ai/agent-framework/author-agent) · [MLflow ResponsesAgent](https://mlflow.org/docs/latest/genai/flavors/responses-agent-intro/)

---

### Lesson 2.2 — `predict` vs `predict_stream`
**Concept.** `predict` returns the **whole answer at once** — one `ResponsesAgentResponse` with a list of output items. `predict_stream` **yields events as they happen** — text deltas, tool calls, tool results — so a UI can render token-by-token. Same logic, two surfaces: batch and streaming. The AI Playground and a chat UI want streaming; an eval job is happy with `predict`. You write the agent once and expose both.

**Walkthrough.** A complete (if trivial) agent showing both, plus tracing:
```python
import mlflow
from mlflow.entities.span import SpanType
from mlflow.pyfunc import ResponsesAgent
from mlflow.types.responses import (
    ResponsesAgentRequest, ResponsesAgentResponse, ResponsesAgentStreamEvent,
)
from typing import Generator

class SimpleResponsesAgent(ResponsesAgent):
    @mlflow.trace(span_type=SpanType.AGENT)
    def predict(self, request: ResponsesAgentRequest) -> ResponsesAgentResponse:
        return ResponsesAgentResponse(
            output=[self.create_text_output_item(text="Hello world!", id="msg_1")]
        )

    @mlflow.trace(span_type=SpanType.AGENT)
    def predict_stream(
        self, request: ResponsesAgentRequest
    ) -> Generator[ResponsesAgentStreamEvent, None, None]:
        # stream deltas — ALL share the same item_id...
        for chunk in ["Hello", " world", "!"]:
            yield ResponsesAgentStreamEvent(**self.create_text_delta(delta=chunk, item_id="msg_1"))
        # ...then close with a matching done event carrying the full item
        yield ResponsesAgentStreamEvent(
            type="response.output_item.done",
            item=self.create_text_output_item(text="Hello world!", id="msg_1"),
        )
```
**The streaming rule that bites people:** every `create_text_delta` and the final `response.output_item.done` item must share the **same `item_id`** (`"msg_1"` here), or the client can't stitch the deltas into one message.

**Challenge.** Instantiate it and call both:
```python
agent = SimpleResponsesAgent()
req = ResponsesAgentRequest(input=[{"role": "user", "content": "hi"}])
print(agent.predict(req).output[0])
for ev in agent.predict_stream(req):
    print(ev.type)
```
**STOP** and tell me the sequence of event `type`s `predict_stream` emitted.

**Check.** (1) When would you reach for `predict_stream` over `predict`? (2) What breaks if your text deltas and your `response.output_item.done` use *different* `item_id`s?

> Note: `ResponsesAgentRequest(input=[...])` — the request field is `input` (verified: field is `input` on mlflow 3.14).

**Source.** [MLflow ResponsesAgent — streaming](https://mlflow.org/docs/latest/genai/serving/responses-agent)

---

### Lesson 2.3 — Packaging: `%%writefile` + `mlflow.models.set_model`
**Concept.** A class defined in a notebook cell can't be deployed — MLflow needs your agent as a **file it can re-import** in a fresh serving process. The pattern is **"models from code"**: write your agent to `agent.py`, end that file with **`mlflow.models.set_model(AGENT)`** to mark *which* object is the model, then `log_model(python_model="agent.py", ...)`. No pickling of live objects, no "it worked in my notebook" surprises — the file *is* the contract.

**Walkthrough.** First cell writes the file:
```python
%%writefile agent.py
import mlflow
from mlflow.pyfunc import ResponsesAgent
from mlflow.types.responses import ResponsesAgentRequest, ResponsesAgentResponse

class SimpleResponsesAgent(ResponsesAgent):
    def predict(self, request: ResponsesAgentRequest) -> ResponsesAgentResponse:
        return ResponsesAgentResponse(
            output=[self.create_text_output_item(text="Hello world!", id="msg_1")]
        )

AGENT = SimpleResponsesAgent()
mlflow.models.set_model(AGENT)   # <-- tells MLflow THIS is the model
```
Next cell logs it from that file:
```python
import mlflow
with mlflow.start_run():
    logged_agent_info = mlflow.pyfunc.log_model(
        python_model="agent.py",      # the FILE, not the class
        name="agent",
        input_example={"input": [{"role": "user", "content": "hi"}]},
    )
print(logged_agent_info.model_uri)
```
MLflow infers the signature and stamps the task metadata `{"task": "agent/v1/responses"}` automatically. The returned `model_uri` (e.g. `runs:/<run_id>/agent`) is what Module 5 registers to Unity Catalog and deploys.

**Challenge.** `%%writefile agent.py` the SimpleResponsesAgent, then `log_model` it. Reload it with `mlflow.pyfunc.load_model(logged_agent_info.model_uri)` and call `.predict({"input": [{"role":"user","content":"hi"}]})`. **STOP** and tell me the model_uri and whether the reloaded model answered "Hello world!".

**Check.** (1) Why can't MLflow just deploy the class object straight from your notebook cell? (2) What single line in `agent.py` tells MLflow which object to serve, and what happens if you forget it?

**Source.** [MLflow ResponsesAgent — logging](https://mlflow.org/docs/latest/genai/serving/responses-agent) · [Author an AI agent](https://docs.databricks.com/aws/en/generative-ai/agent-framework/author-agent)

---

### Lesson 2.4 — Authoring frameworks overview (pick your engine)
**Concept.** `ResponsesAgent` is the *wrapper*; inside it you can run whatever **engine** you like. Databricks documents four: **PyFunc** (raw Python — your Module 1 OpenAI loop, maximum control), **LangGraph** (graph of nodes/edges, great for branching and explicit state — Databricks' default sample), the **OpenAI Agents SDK** (lightweight `Agent` + `@function_tool`, the App template), and **LlamaIndex** (retrieval-first / RAG-heavy apps). They all reduce to the same logged `ResponsesAgent`, so your choice is about *ergonomics*, not lock-in — you can re-wrap later.

| Framework | Best when | Looks like |
|---|---|---|
| PyFunc (raw) | full control, minimal deps | your Module 1 `while` loop |
| LangGraph | branching, explicit state machine | nodes + edges graph |
| OpenAI Agents SDK | quick tool-using assistant | `Agent(...)` + `@function_tool` |
| LlamaIndex | retrieval / RAG | index + query engine |

**Walkthrough.** No new run — just locate the menu. In your workspace, click **+ New ▸ Notebook**, then **File ▸ Add sample** (or browse the **Marketplace / sample gallery**) and note that the agent samples are labeled by framework. The two you'll build next (2.5 LangGraph, 2.6 OpenAI Agents SDK) are the ones Databricks ships ready-made.

**Challenge.** Of the four, which fits *your* Module 1 square-root agent with the least rewrite, and which would you pick if you were about to add Vector Search retrieval (Module 3.5)? **STOP** and give me your two picks with one reason each.

**Check.** (1) True/false: choosing LangGraph means you can't use the Databricks deploy path. (Why?) (2) What do all four frameworks have in common at the *interface* layer?

**Source.** [Build agents on Databricks](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 2.5 — Build with LangGraph (Databricks' default sample)
**Concept.** LangGraph models the agent loop as an explicit **graph**: a node for the LLM call, a node for the tools, and a conditional edge "if the model asked for a tool, go run it, else finish." It's the same loop you wrote by hand in Lesson 1.4 — but the branching is declared, not buried in `if` statements, which pays off the moment you add a second tool or a retry path. Databricks ships this as a default notebook sample, so you wrap LangGraph's compiled graph in a `ResponsesAgent` and log it exactly like 2.3.

**Walkthrough.** Shape (the Databricks LangGraph sample fills in the bodies):
```python
%%writefile agent.py
import mlflow
from databricks_langchain import ChatDatabricks
from langgraph.prebuilt import create_react_agent
from databricks_langchain import UCFunctionToolkit
from mlflow.pyfunc import ResponsesAgent
from mlflow.types.responses import ResponsesAgentRequest, ResponsesAgentResponse

llm = ChatDatabricks(endpoint="databricks-claude-3-7-sonnet")
tools = UCFunctionToolkit(function_names=["workspace.default.python_exec"]).tools
graph = create_react_agent(llm, tools)          # the LLM↔tool loop, as a graph

class LangGraphResponsesAgent(ResponsesAgent):
    def predict(self, request: ResponsesAgentRequest) -> ResponsesAgentResponse:
        messages = [m.model_dump() for m in request.input]
        result = graph.invoke({"messages": messages})
        final = result["messages"][-1].content
        return ResponsesAgentResponse(
            output=[self.create_text_output_item(text=final, id="msg_1")]
        )

AGENT = LangGraphResponsesAgent()
mlflow.models.set_model(AGENT)
```
Then log it with the same `mlflow.pyfunc.log_model(python_model="agent.py", name="agent")` from 2.3 and `load_model(...).predict(...)`.

**Challenge.** Open the workspace **LangGraph agent sample notebook** (the ready-made one), run its setup + define-agent + log_model cells unchanged, and ask it the Module 1 smoke test "what is the square root of 429?". Confirm it routes through the tool. **STOP** and tell me the answer it returned and whether a tool call showed up in the trace.

**Check.** (1) In graph terms, what is the conditional edge deciding? (2) The wrapping `ResponsesAgent` is thin — what's the actual reasoning engine underneath, and where does the Databricks-served LLM plug in?

> ⚠️ verify: exact import paths (`databricks_langchain` vs `databricks_langchain.UCFunctionToolkit`) and `create_react_agent`'s return-message shape can shift across versions — trust the workspace sample's imports over this sketch; treat the snippet as structure, not gospel.

**Source.** [Build agents on Databricks](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 2.6 — Build with the OpenAI Agents SDK template (New ▸ App ▸ Agents)
**Concept.** The OpenAI Agents SDK is the lightest path to a tool-using agent: declare an `Agent` with instructions, a Databricks-served model, and `@function_tool`-decorated Python functions — the SDK runs the loop. Databricks packages this as a **one-click App template** that bundles a chat UI, an async FastAPI **MLflow AgentServer** with built-in tracing, and evaluation code, all deployable with Asset Bundles. This is the fastest "from zero to a chat URL".

**Walkthrough.**
1. In the workspace, click **+ New ▸ App**, choose **Agents ▸ Custom Agent (OpenAI SDK)**.
2. When prompted, create the MLflow experiment (the template suggests `openai-agents-template`) and finish setup — this installs the `agent-openai-agents-sdk` template.
3. The agent code looks like:
   ```python
   from agents import Agent, function_tool

   @function_tool
   def get_current_time() -> str:
       """Get the current date and time."""
       from datetime import datetime
       return datetime.now().isoformat()

   agent = Agent(
       name="My agent",
       instructions="You are a helpful assistant.",
       model="databricks-claude-sonnet-4-5",
       tools=[get_current_time],
   )
   ```
4. Open the app URL to chat with it. To deploy from the CLI instead, clone `https://github.com/databricks/app-templates.git` and run:
   ```bash
   databricks bundle validate
   databricks bundle deploy
   databricks bundle run agent_openai_agents_sdk
   ```
   Permissions (which serving endpoints/UC functions the app may reach) are declared in `databricks.yml` under `resources.apps`; per-user calls need `user_api_scopes`.

**Challenge.** Create the App from the template, open its chat URL, and ask "what time is it?" — confirm the `get_current_time` tool fired (check the trace in the bundled MLflow AgentServer). **STOP** and tell me the timestamp it returned and whether you saw a tool span.

**Check.** (1) What three things does the App template bundle *besides* the agent code? (2) Where do you declare which serving endpoint and UC functions the deployed app is allowed to call?

**Source.** [Author an agent as a Databricks App](https://docs.databricks.com/aws/en/generative-ai/agent-framework/author-agent-db-app) · [Agent quickstart](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-quickstart)

---

### Lesson 2.7 — Kernel/session gotchas (where your variables go to die)
**Concept.** Notebook state is **mutable and sticky**, and that's the source of most "but it worked a minute ago" agent bugs. Three traps: (1) installing a library with `%pip install` doesn't take effect until you **restart Python** — `dbutils.library.restartPython()` — which *wipes every variable in the session*; (2) **"Run all above"** vs **"Run all"** run different cell sets, so a cell that depends on output below it silently uses stale or missing state; (3) re-running your `%%writefile agent.py` cell rewrites the file but does **not** re-log the model — the deployed artifact is whatever you last `log_model`'d, not what's in the cell now.

**Walkthrough.** The safe sequence after any `%pip install`:
```python
%pip install -U mlflow databricks-langchain databricks-openai
dbutils.library.restartPython()   # NOTE: clears `agent`, `client`, every var you defined
```
After that restart, **re-run your setup cells** (imports, client, `agent.py` writefile, `log_model`) before testing — nothing from before the restart survives. Reach for **"Run all above"** to rebuild state up to your current cell; use **"Run all"** only from a clean kernel. And remember: editing the agent class in a cell changes nothing about the *logged* model until you re-run `%%writefile` **and** `log_model`.

**Challenge.** Define `x = 42`, then run `dbutils.library.restartPython()`, then try to `print(x)`. Confirm it's gone (NameError). **STOP** and tell me the error — that NameError is the single most common "my agent broke" cause, and now you'll recognize it instantly.

**Check.** (1) Why does `restartPython()` force you to re-run your setup cells? (2) You edit your `predict` method in the notebook but the deployed agent still gives the old answer — name the two cells you forgot to re-run. (3) "Run all above" vs "Run all": which rebuilds state *up to* your cursor?

**Source.** [Agent quickstart](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-quickstart) · [Build agents on Databricks](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

## Module 2 wrap
By here the learner can: wrap any agent engine in MLflow's **`ResponsesAgent`** interface,
implement both **`predict`** and streaming **`predict_stream`**, **package** it as a
`%%writefile agent.py` + `mlflow.models.set_model` + `log_model` artifact, choose among
**PyFunc / LangGraph / OpenAI Agents SDK / LlamaIndex**, build the two Databricks-shipped
samples (LangGraph notebook, OpenAI Agents SDK App), and survive the notebook
kernel/session traps that masquerade as agent bugs. Next: **Module 3 — Tools & Data** —
give the agent real power with UC-function tools, **Genie Spaces**, Vector Search retrieval,
and **MCP**.
