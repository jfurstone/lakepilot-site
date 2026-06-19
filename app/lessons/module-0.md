# Module 0 — Setup & Orientation (lesson-by-lesson, v0.1)

**Objective:** go from "I have a Databricks login" to "I can talk to my workspace from code" —
the foundation every later module assumes. By the end you've authenticated, built the local
toolchain, and proved the connection.
**Prereqs:** a Databricks workspace login (Free Edition is fine — see 0.3) and a terminal.
**Sandbox:** your terminal + a text editor for the local setup; the Databricks UI for the tour.
**Default tutor model:** `claude-haiku-4-5`.

> Format per lesson: **Concept** (why) → **Watch** (optional) → **Walkthrough** (run it) →
> **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.

---

### Lesson 0.1 — What "agent" actually means on Databricks
**Concept.** An **agent is not a chatbot.** A chatbot talks; an agent **reasons, uses tools, and
loops** — it decides *to act*, runs a tool, reads the result, and decides what to do next. On
Databricks specifically, "agent" means the **Mosaic AI Agent Framework / Agent Bricks** stack: the
LLM is *served inside your workspace*, the tools are *governed Unity Catalog objects*, and you
*deploy* the result as a serving endpoint. The whole course is: build one → test it → deploy it →
orchestrate several. Knowing that shape now keeps every later piece from feeling random.

**📺 Watch (optional, then DO):** [AI Agents on Databricks in 5 Minutes](https://www.databricks.com/resources/demos/videos/ai-agents-on-mosaic-ai-in-5-minutes) (~5m)

**Walkthrough.** No code — a thought experiment. Open any plain chat model and ask *"what is 17
factorial?"* It will often **guess wrong** (LLMs are bad at exact arithmetic). An agent answers the
same question by *calling a tool* — running `import math` / Python — and returns the exact number.
That gap (guess vs. compute) is the entire reason agents exist.

**Challenge.** Write down one task you'd want an agent to do that a plain chatbot *couldn't* — and
name the one tool it would need (run code? query a table? call an API?). **STOP** and tell me your
task + its tool.

**Check.** (1) Name the three things that make an agent more than a chatbot. (2) On Databricks, where
does the LLM your agent calls actually run?

**Source.** [What are GenAI apps / agents on Databricks](https://docs.databricks.com/aws/en/generative-ai/guide/introduction-generative-ai-apps)

---

### Lesson 0.2 — The platform map (so you know where to click)
**Concept.** Six pieces of Databricks show up across this whole course. Learn what each *does* now
and you'll never be lost hunting for a menu:
| Piece | What it does for agents |
|---|---|
| **Unity Catalog** | governs your data **and** your tools (functions live here) |
| **Foundation Model API** | serves the **LLMs** your agent calls |
| **Model Serving** | hosts your **deployed agent** as a REST endpoint |
| **Genie** | natural-language **→ SQL** over your tables |
| **Databricks Apps** | hosts a **web UI** for your agent |
| **MLflow** | **tracing, evaluation, and the model registry** |

**Walkthrough.** A 2-minute tour — open the left nav and just *find* these, no clicking deeper:
**Catalog** (your data + functions), **SQL Warehouses** (compute for SQL/Genie), **Serving**
(deployed endpoints), **Playground** (chat with models), **Agents** (Agent Bricks), **Genie Spaces**,
**Experiments** (MLflow traces).

**Challenge.** In your workspace's left nav, find and name the page for: (a) your tables, (b) deployed
agents, (c) experiment traces. **STOP** and give me the three nav labels.

**Check.** (1) Which piece *serves* the LLM your agent calls? (2) Which piece *governs* both your tools
and your data?

**Source.** [Mosaic AI Agent Framework overview](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 0.3 — Free Edition vs a paid workspace (what you need)
**Concept.** You can do **almost all of this course on Databricks Free Edition** — it includes
serverless compute, Unity Catalog, Model Serving, Foundation Models, and Genie. A paid workspace
adds scale, private networking, and some model/compute options you won't need to *learn*. The
minimum the course actually requires: **(1)** a workspace you can log into, **(2)** serverless or a
SQL warehouse for compute, and **(3)** access to at least one **Foundation Model serving endpoint**
(e.g. a Claude or Llama endpoint). If you've got those three, you're set.

**Walkthrough.** Confirm your three essentials: sign in → can you open **Catalog**? → is there a
**SQL Warehouse** (the "Serverless Starter Warehouse" counts)? → under **Serving** or **Playground**,
do you see foundation model endpoints like `databricks-claude-3-7-sonnet`?

**Challenge.** Confirm a SQL warehouse exists (Serverless Starter is fine) and you can open Catalog.
**STOP** and tell me which edition you're on (Free or a named paid workspace) and one model endpoint
you can see.

**Check.** (1) What three capabilities does the course minimally need? (2) True/false: you need a paid
workspace to deploy an agent to a serving endpoint. (Why?)

**Source.** [Databricks Free Edition](https://docs.databricks.com/aws/en/getting-started/free-edition)

---

### Lesson 0.4 — Authentication (the part that bites everyone)
**Concept.** Inside a notebook, auth is automatic. The moment you talk to Databricks **from outside**
— a local script, the tutor sandbox — you need two things: your **workspace URL** (the *host*) and a
**Personal Access Token (PAT)**. And the trap that costs people an afternoon: **PATs are scoped.** A
token needs the right scopes for what you're doing — **`model-serving`** to call LLMs, **`unity-catalog`**
for tools/data, **`workspace`** for notebooks. Use a token missing a scope and the API fails with a
blunt `required scopes: model-serving`. The standard place to store host + token is **`~/.databrickscfg`**,
which the SDK and MLflow read automatically.

**Walkthrough.**
1. **Host** = the base of your workspace URL: `https://dbc-xxxxxxxx-xxxx.cloud.databricks.com` (drop any
   `/editor/...` path).
2. **Make a PAT:** Settings → Developer → **Access tokens** → Manage → **Generate new token**. Set a
   *short* lifetime. ⚠️ The picker **groups** scopes: **"BI Tools"** = a broad all-APIs token; **"Other
   APIs"** = pick specific scopes. For agent work you want **`model-serving` + `unity-catalog`** (add
   `workspace` if you'll export notebooks). A real token **starts with `dapi`**.
3. **Write the config file** at `~/.databrickscfg`:
   ```ini
   [DEFAULT]
   host  = https://dbc-xxxxxxxx-xxxx.cloud.databricks.com
   token = dapiXXXXXXXXXXXXXXXXXXXXXXXX
   ```
   Lock its permissions and **never commit it** (add to `.gitignore`).

**Challenge.** Generate a short-lifetime PAT, write your `~/.databrickscfg`, and confirm the file
exists with a `host` line and a `token` that starts with `dapi`. **STOP** and tell me the host line
and that a token is present — **do not paste the token itself**.

**Check.** (1) You call an LLM and get `403 ... required scopes: model-serving` — what's wrong and
where do you fix it? (2) Where does the SDK read your host + token from by default?

**Source.** [Authenticate with personal access tokens](https://docs.databricks.com/aws/en/dev-tools/auth/pat)

---

### Lesson 0.5 — The local toolchain (Python 3.11 venv + the SDK)
**Concept.** Two gotchas trip up *everyone* on day one. **First:** the Databricks agent packages
(`mlflow`, `databricks-agents`, `databricks-openai`) don't yet ship wheels for the newest Python
(3.14), so build a **Python 3.11 virtual environment** — not the system default. **Second:**
`%pip install ...` is a **notebook magic**, not a shell command — paste it in your terminal and you
get `'%pip' is not recognized`. Locally you use plain `pip` / `python -m pip`. Get these two right and
the rest of setup is painless.

**Walkthrough.**
```bash
# 1. Make a Python 3.11 venv (Windows py launcher shown; use uv or python3.11 elsewhere)
py -3.11 -m venv .venv
.\.venv\Scripts\activate                     # macOS/Linux: source .venv/bin/activate

# 2. Install the agent toolchain (NOTE: plain pip, not %pip)
python -m pip install -U mlflow databricks-sdk databricks-agents databricks-openai
```
One import gotcha to memorize: **`databricks-agents` imports as `from databricks import agents`** —
*not* `import databricks_agents`.

**Challenge.** Create a 3.11 venv, install the four packages, then run:
```bash
python -c "import mlflow, databricks_openai; from databricks import agents; print('mlflow', mlflow.__version__)"
```
**STOP** and tell me the mlflow version it printed (and confirm no import errors).

**Check.** (1) Why build a 3.11 venv instead of using the system's Python 3.14? (2) `%pip install` failed
in your terminal — why, and what's the local equivalent?

**Source.** [Install the Databricks SDK for Python](https://docs.databricks.com/aws/en/dev-tools/sdk-python) · [Mosaic AI Agent Framework requirements](https://docs.databricks.com/aws/en/generative-ai/agent-framework/build-agents)

---

### Lesson 0.6 — First connection (prove the whole setup works)
**Concept.** Before you touch an agent, run the "hello world" of Databricks-from-code: **authenticate
and identify yourself.** If `current_user.me()` returns *your* email, then host + token + scopes are
all wired correctly. Then list warehouses to confirm you can reach compute. This one snippet is the
gate — pass it and every later module just works; fail it and you fix auth (0.4), not your agent.

**Walkthrough.**
```python
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()            # reads ~/.databrickscfg automatically
me = w.current_user.me()
print("Connected as:", me.user_name)

for wh in w.warehouses.list():
    print("  warehouse:", wh.name, "—", wh.state)
```
Reading the failures: an **`Unauthenticated`** / "credential was not sent or unsupported" error → a bad
token (confirm it starts with `dapi` and isn't expired). A **`403` on a *specific* API** (e.g. listing
catalogs) → a **missing scope** (back to 0.4), not a broken token.

**Challenge.** Run the snippet. Confirm it prints your identity and at least one warehouse. **STOP** and
tell me the email it connected as and one warehouse name + state.

**Check.** (1) What does a successful `current_user.me()` prove about your setup? (2) You connect fine but
`w.catalogs.list()` returns `403` — what's missing, and is that a *token* problem or a *scope* problem?

**Source.** [Databricks SDK for Python — authentication](https://docs.databricks.com/aws/en/dev-tools/sdk-python)

---

## Module 0 wrap
By here the learner can: explain what an agent *is* on Databricks, navigate the platform's six core
pieces, confirm their workspace meets the bar, **authenticate from code with a scoped PAT in
`~/.databrickscfg`**, build a Python 3.11 toolchain, and **prove the connection** with
`current_user.me()`. Everything is now in place. Next: **Module 1 — Your First Agent**, where you call a
served LLM, give it a tool, and watch the agent loop run end to end.
