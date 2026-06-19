# Module 5 — Deployment (lesson-by-lesson, v0.1)

**Objective:** take the agent you authored in Module 2 and *ship it* — log & register it to Unity
Catalog, deploy it to Model Serving with `agents.deploy()`, chat with it in the AI Playground,
front it with a real Databricks Apps chat UI, **read deploy failures when they happen**, and
then turn it all off so it stops costing money.
**Prereqs:** Module 1 (a working tool-using agent) + Module 2 (a `ResponsesAgent` you can log).
You need a Unity Catalog catalog/schema you can write to, and `mlflow>=3.1.3` + `databricks-agents>=1.1.0`.
**Sandbox:** a Databricks notebook is easiest (deploy from a notebook so MLflow tracing works); the Apps
lesson uses the Databricks CLI / your terminal.
**Default tutor model:** `claude-haiku-4-5`.

> Format per lesson: **Concept** (why) → **Watch** (optional) → **Walkthrough** (run it) →
> **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.

> ⚠️ Databricks now recommends **Databricks Apps** as the default deployment path for *new* agents
> (full control over code + server). Model Serving via `agents.deploy()` is still fully supported and
> is the simplest "endpoint in one call" path — so we teach **both**: Model Serving first (5.3), Apps
> after (5.5). Pick whichever your project needs.

---

### Lesson 5.1 — Log & register to Unity Catalog (the model registry)
**Concept.** Before anything can *serve* your agent, the agent has to become a **registered model** — a
versioned, governed artifact living in Unity Catalog. Deployment doesn't ship a notebook; it ships a
*logged model version*. Two steps: **log** (MLflow packages your agent code + its dependencies into a
run) then **register** (promote that logged model into UC as `catalog.schema.name`, version N). The
one line that makes UC the registry instead of the old workspace registry is
`mlflow.set_registry_uri("databricks-uc")` — forget it and your model lands in the wrong place.

**Walkthrough.**
```python
import mlflow
from mlflow.models.resources import DatabricksServingEndpoint

mlflow.set_registry_uri("databricks-uc")   # <-- registry = Unity Catalog

# 1. LOG: package the agent code you wrote with %%writefile in Module 2 (agent.py)
input_example = {"messages": [{"role": "user", "content": "What is the square root of 429?"}]}
with mlflow.start_run():
    logged_agent_info = mlflow.pyfunc.log_model(
        python_model="agent.py",                 # the file holding mlflow.models.set_model(AGENT)
        name="agent",                            # name= is valid in mlflow 3.14 (legacy artifact_path= also works)
        input_example=input_example,
        resources=[                              # more on this in 5.2
            DatabricksServingEndpoint(endpoint_name="databricks-claude-3-7-sonnet"),
        ],
    )

# 2. REGISTER: promote the logged model into Unity Catalog
UC_MODEL_NAME = "workspace.default.lakepilot"   # catalog.schema.name — use a UC location you own
uc_model_info = mlflow.register_model(
    model_uri=logged_agent_info.model_uri,
    name=UC_MODEL_NAME,
)
print(uc_model_info.name, "version", uc_model_info.version)   # e.g. workspace.default.lakepilot version 1
```
You now have a versioned model in UC. `uc_model_info.version` is the handle you'll deploy in 5.3.
> ⚠️ verify: arg name is `python_model="agent.py"` here (a path to your packaged agent file). Some
> templates pass `python_model=AGENT` (the object) instead — both appear in Databricks examples.

**Challenge.** Log and register your Module 2 agent to a UC location you own (e.g.
`workspace.default.lakepilot`). Open **Catalog ▸** your catalog ▸ schema ▸ **Models** and find it.
**STOP** and tell me the full three-level name and the version number it registered as.

**Check.** (1) What does `mlflow.set_registry_uri("databricks-uc")` change — and what breaks if you skip it?
(2) What's the difference between *logging* a model and *registering* it?

**Source.** [Log and register AI agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/log-agent)

---

### Lesson 5.2 — Resources & auto-auth (`DatabricksServingEndpoint`, `DatabricksFunction`)
**Concept.** A deployed agent runs as a **service principal**, not as you. So when it tries to call the
LLM endpoint or a UC-function tool, *your* credentials aren't there — and without help it gets a 403.
The fix is to **declare every resource the agent touches** in `resources=[...]` at log time. Databricks
reads that list and, at deploy, **provisions short-lived credentials** so the agent can reach exactly
those resources and nothing more (least privilege). This is the single most common reason a deploy
*succeeds* but the agent *fails on the first real query* — an undeclared resource.

**Walkthrough.** List one entry per thing your agent calls:
```python
from mlflow.models.resources import (
    DatabricksServingEndpoint,   # the LLM your agent calls
    DatabricksFunction,          # each UC-function tool
)

resources = [
    DatabricksServingEndpoint(endpoint_name="databricks-claude-3-7-sonnet"),
    DatabricksFunction(function_name="workspace.default.python_exec"),   # your tool from Lesson 1.7
    # add a DatabricksVectorSearchIndex(...) / DatabricksGenieSpace(...) here if the agent uses one
]
# pass resources=resources into mlflow.pyfunc.log_model(...) exactly as in 5.1
```
Tip: if your agent code declares resources via `mlflow.models.resources` internally, MLflow can often
**auto-infer** them — but for any external/UC tool, declaring them explicitly is the reliable path.

**Challenge.** Add a `DatabricksFunction(...)` for the tool your agent actually calls (from Module 1/3)
to the `resources` list, then re-log (this creates a new version). **STOP** and paste your final
`resources=[...]` list and tell me what each entry is for.

**Check.** (1) A deployed agent calls a UC function tool and gets a 403, but it worked in your notebook —
what's the most likely cause? (2) Why does declaring resources give the agent *less* access than just
running as you, and why is that good?

**Source.** [Authentication for AI agents (Model Serving)](https://docs.databricks.com/aws/en/generative-ai/agent-framework/agent-authentication-model-serving)

---

### Lesson 5.3 — Deploy to Model Serving — `agents.deploy(...)`, `scale_to_zero`
**Concept.** One function turns your registered model into a live REST endpoint:
`agents.deploy(model_name, version)`. Behind that single call Databricks **creates a serving endpoint**
(autoscaling + load balancing), **provisions the auth** from your 5.2 resources, **wires up MLflow
real-time tracing**, and **spins up a Review App** for stakeholder feedback. `scale_to_zero=True`
lets an idle endpoint scale to zero replicas — you stop paying for idle compute, at the cost of a cold
start (slower first query). Great for a tutor sandbox or a low-traffic internal tool.

**Walkthrough.**
```python
%pip install mlflow>=3.1.3 databricks-agents>=1.1.0
dbutils.library.restartPython()
```
```python
from databricks import agents

deployment = agents.deploy(
    UC_MODEL_NAME,                 # "workspace.default.lakepilot"
    uc_model_info.version,         # the version from 5.1, e.g. 1
    scale_to_zero=True,            # idle -> 0 replicas, no idle cost (cold start on next call)
)
print(deployment.query_endpoint)  # the REST URL you (or an app) POST to
```
Deployment can take **up to ~15 minutes**. Watch progress under **Serving** in the left nav — the
endpoint goes `Not Ready` → `Ready`. When ready you'll also get a **Review App** link for feedback.
The deploy signature is `agents.deploy(model_name, model_version, scale_to_zero=False, workload_size='Small', endpoint_name=None, deploy_feedback_model=False, ...)` (verified against databricks-agents 1.11.0).

**Challenge.** Deploy your registered model with `scale_to_zero=True`. Open **Serving**, watch it
go to **Ready**, and print `deployment.query_endpoint`. **STOP** and tell me the endpoint name and how
long it took to become Ready.

**Check.** (1) Name three things `agents.deploy()` sets up for you besides the raw endpoint.
(2) What's the trade-off you accept when you turn on scale-to-zero?

**Source.** [Deploy an agent (Model Serving)](https://docs.databricks.com/aws/en/generative-ai/agent-framework/deploy-agent)

---

### Lesson 5.4 — The AI Playground — chat + "Get code"
**Concept.** Once an endpoint is `Ready`, you don't need to write a client to try it — the **AI Playground**
is a chat UI wired to your serving endpoints. It's the fastest way to smoke-test a deployed agent, compare
it side-by-side against another model, and then **export working client code** with **Get code** so you
don't hand-write the request shape. This is your "does it actually work for a human?" checkpoint before
you build any UI.

**📺 Watch (optional, then DO):** [Databricks AI Playground with a custom model](https://www.youtube.com/watch?v=bxacSRGnn0I) (YouTube)

**Walkthrough.**
1. Left nav ▸ **AI/ML ▸ Playground**.
2. In the model dropdown, pick your deployed agent endpoint. For a tool-using agent, choose one labelled
   **Tools enabled**.
3. Ask the Module 1 smoke test: *"What is the square root of 429? Use the tool."* — confirm it calls the
   tool and returns ≈ **20.712** (not a hallucinated number).
4. Compare: top-right **+** adds a second pane; pick a different model and tick **Sync** to send the same
   prompt to both at once.
5. Export: **Get code** ▸ choose **Export to Databricks Apps** (a deployed chat app, leads into 5.5),
   **Create agent notebook** (a ResponsesAgent notebook), or a **curl / Python** snippet to call the
   endpoint directly.

**Challenge.** Chat with your deployed agent in the Playground, run the square-root test, then click
**Get code** and grab the Python/curl snippet. **STOP** and tell me whether it called the tool, and paste
the first line of the exported snippet.

**Check.** (1) Why is "Tools enabled" the model label you want for an agent endpoint? (2) What does **Get
code** save you from doing by hand?

**Source.** [Use AI Playground with an agent](https://docs.databricks.com/aws/en/generative-ai/agent-framework/ai-playground-agent)

---

### Lesson 5.5 — Host an agent on Databricks Apps (a real chat UI)
**Concept.** A serving endpoint is an API; **real users want a chat window**. **Databricks Apps** hosts a
web app *inside your workspace* — governed, authenticated, no separate hosting to manage. The agent
templates ship a **built-in chat UI** that talks to your agent, and you deploy it as a **Databricks Asset
Bundle** (`databricks.yml`) with the CLI. This is also Databricks' recommended path for *new* agents because
you control the server code, not just a black-box endpoint.

**Walkthrough.**
```bash
# 1. Start from an official agent app template (built-in chat UI included)
git clone https://github.com/databricks/app-templates.git
cd app-templates/agent-openai-agents-sdk
```
```yaml
# 2. databricks.yml — declare the app + the serving endpoint it may query
resources:
  apps:
    agent_openai_agents_sdk:
      name: 'agent-openai-agents-sdk'
      source_code_path: ./
      config:
        command: ['uv', 'run', 'start-app']
      resources:
        - name: 'llm'
          serving_endpoint:
            name: 'databricks-claude-sonnet-4-5'   # or YOUR deployed agent endpoint
            permission: 'CAN_QUERY'
```
```bash
# 3. Validate, deploy, and launch
databricks bundle validate
databricks bundle deploy
databricks bundle run agent_openai_agents_sdk
```
```bash
# (optional) run locally first at http://localhost:8000
uv run quickstart
uv run start-app
```
The app appears under **Compute ▸ Apps** with its own URL. Note: Databricks Apps requires **medium or
large** compute sizes.
> ⚠️ verify: the exact template folder name and `command`/`uv run` scripts come from the current
> `databricks/app-templates` repo — check the template's own README, as template names change.
> Shortcut: in the Playground (5.4), **Get code ▸ Export to Databricks Apps** scaffolds most of this for you.

**Challenge.** Either (a) export your agent to an App via the Playground's **Export to Databricks Apps**,
or (b) clone a template, point its `serving_endpoint` at *your* deployed endpoint, and
`databricks bundle deploy` it. Open the App URL and send one message. **STOP** and tell me the App URL and
whether the chat replied.

**Check.** (1) What does the App give a real user that the raw serving endpoint doesn't? (2) In
`databricks.yml`, why must you declare the `serving_endpoint` resource with `CAN_QUERY`?

**Source.** [Author an AI agent and deploy it on Databricks Apps](https://docs.databricks.com/aws/en/generative-ai/agent-framework/author-agent-db-app)

---

### Lesson 5.6 — Reading deploy failures (model server failed to load; update-failed states)
**Concept.** Deploys fail. The two you'll actually hit: the endpoint gets stuck **"Not ready (Update
failed)"**, and the build/serving logs say **"model server failed to load the model."** This almost never
means your agent logic is wrong — it means the *serving container couldn't reconstruct your model*:
missing/mismatched dependency, an undeclared resource (back to 5.2), or code that errors at import time.
The skill is reading the **build logs** and **service logs** on the Serving page instead of re-deploying
blindly and hoping.

**Walkthrough.** When an endpoint shows **Update failed**:
1. **Serving ▸** your endpoint ▸ **Events** tab — see *when* and *why* the update failed (often a build error).
2. Open **Build logs** — a pip/dependency resolution failure or a blocked package download (`pypi.org`
   unreachable) shows here. Fix: pin `pip_requirements` at log time so the serving env matches your notebook.
3. Open **Service logs / Model server logs** — "**failed to load the model**" + a Python traceback means the
   model errored while *loading* (e.g. an import that needs a package you didn't log, or code that runs at
   module import). Reproduce locally with
   `mlflow.models.predict(model_uri=logged_agent_info.model_uri, input_data=input_example)` before re-deploying.
4. **403 / permission** errors at *query* time (endpoint Ready but every call fails) → an **undeclared
   resource** in 5.2. Add the `DatabricksServingEndpoint` / `DatabricksFunction`, re-log, re-deploy.
5. Re-deploy the **fixed version**. Because updates are version-additive (5.3), the old version keeps serving
   until the new one is healthy — a failed update doesn't take a working endpoint down.

> Real-world note: a fresh deploy can sit at "Not ready" for 10–15 min and *then* flip to **Update failed** —
> that's normal timing, not a hang. Read the logs; don't spam re-deploys.

**Challenge.** Deliberately break it: re-log your agent **omitting** a tool it needs from `resources`,
deploy, and query it from the Playground. Capture the error. **STOP** and tell me what the error said and
which log (build vs service vs query-time 403) it showed up in.

**Check.** (1) "Model server failed to load the model" — is that more likely your *agent logic* or your
*environment/dependencies*? (2) Where do you look first when an endpoint shows **Update failed**, and what's
the one command that reproduces a load failure locally?

**Source.** [Deploy an agent (Model Serving)](https://docs.databricks.com/aws/en/generative-ai/agent-framework/deploy-agent) · also experienced live: deploys can hang at "Not ready" then fail.

---

### Lesson 5.7 — Cost & cleanup — scale-to-zero, DBUs, deleting endpoints
**Concept.** A live serving endpoint **bills in DBUs while it's up** — and unless you enabled scale-to-zero,
"up" means *all the time*, even with zero traffic. The two levers that control your bill: **scale-to-zero**
(idle → 0 replicas → ~no idle cost, with a cold-start penalty) and **deleting the endpoint** when you're
done (the only way to truly stop the meter for a sandbox). Leaving a "just testing" endpoint running over a
weekend is the classic surprise-bill. Treat cleanup as part of deploying, not an afterthought.

**Walkthrough.**
```python
from databricks import agents

# 1. See what's live (and billing)
print(agents.list_deployments())

# 2. Inspect a specific one
agent_model_name = UC_MODEL_NAME
agent_model_version = uc_model_info.version
print(agents.get_deployments(model_name=agent_model_name, model_version=agent_model_version))

# 3. STOP THE COST: delete the agent deployment / its endpoint
agents.delete_deployment(model_name=agent_model_name, model_version=agent_model_version)
```
Or from the UI: **Serving ▸** your endpoint ▸ **⋮ (kebab menu) ▸ Delete**.
- **Scale-to-zero** (set at deploy in 5.3) cuts *idle* cost but the endpoint still exists; first call after
  idle is slow (cold start).
- **Delete** removes the endpoint entirely — use this for sandbox/tutorial work so nothing bills overnight.
- The **registered UC model** is just metadata; it does **not** bill. You can delete the endpoint and keep
  the model version to redeploy later.

**Challenge.** Run `list_deployments()`, confirm your endpoint is there, then **delete it** (CLI/SDK or
Serving ▸ ⋮ ▸ Delete) and re-run `list_deployments()` to confirm it's gone. **STOP** and tell me the
endpoint is deleted so we know nothing is still billing.

**Check.** (1) What's the difference, cost-wise, between an endpoint with scale-to-zero **on** vs **deleting**
it? (2) After you delete the serving endpoint, does your *registered UC model* still cost anything — and can
you redeploy it later?

**Source.** [Deploy an agent (Model Serving) — retrieve and delete deployments](https://docs.databricks.com/aws/en/generative-ai/agent-framework/deploy-agent) · [Model Serving](https://docs.databricks.com/aws/en/machine-learning/model-serving/)

---

## Module 5 wrap
By here the learner can: log & register an agent to Unity Catalog, declare its resources for auto-auth,
deploy it to Model Serving in one call (with scale-to-zero), chat with it and export client code in the AI
Playground, front it with a real Databricks Apps chat UI, **diagnose a stuck/failed deploy from the logs**,
and **tear it all down so it stops costing money**. Next: **Module 6 — Multi-Agent Orchestration (Agent
Bricks Supervisor)** — one brain that classifies, routes to the specialist agents you just learned to ship,
and synthesizes their answers.
