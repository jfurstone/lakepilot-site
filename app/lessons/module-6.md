# Module 6 — Multi-Agent Orchestration (Agent Bricks Supervisor) (lesson-by-lesson, v0.1)

**Objective:** build one **Supervisor Agent** that classifies a question, routes it to the right
specialist (a Genie Space, a Knowledge Assistant, a custom agent, a UC function…), and synthesizes
one answer — then test it, improve it from labeled examples, and drive it programmatically via the
Beta Supervisor API. This is the **orchestrator** module: everything you built in Modules 1–5
becomes a *sub-agent* a supervisor can call.

**Prereqs:** Modules 1–5 (a served LLM, UC-function tools, a Genie Space from 3.3–3.4, and ideally
one deployed agent from Module 5). Agent Bricks must be enabled in your workspace; the **Supervisor
API** is **Beta** and needs an admin to turn it on under **Previews**.
**Sandbox:** the **Agents** UI (left sidebar ▸ Agents) for most lessons; a notebook / the local 3.11
venv for the API lesson (6.6).
**Default tutor model:** `claude-haiku-4-5`.

> Format per lesson: **Concept** (why) → **Watch** (optional) → **Walkthrough** (run it) →
> **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.

> ⚠️ The Supervisor API (Beta) surface in 6.6 (import name, `responses.create` params, tool dict
> shapes) is summarized from the Beta doc and **may have shifted** — every API field below is marked
> `> ⚠️ verify` and should be confirmed live against your workspace before relying on it.

---

### Lesson 6.1 — The supervisor pattern: classify → route → synthesize
**Concept.** A single agent with twelve tools gets confused — it picks wrong tools, blends contexts,
and is impossible to debug. The fix is **hierarchy**: a *supervisor* that owns no domain knowledge of
its own but is excellent at three things — **classify** the incoming request, **route** it to the one
specialist best suited (or a few in sequence), and **synthesize** their outputs into a single coherent
answer. Each specialist stays small, well-described, and independently testable. This is the same
"manager + team" shape that makes human orgs scale, and it's exactly what Agent Bricks' **Multi-Agent
Supervisor** automates: *you* register the specialists and describe them; *Databricks* runs the
classify→route→synthesize loop for you.

**📺 Watch (optional, then DO):** [Agent Bricks Multi-Agent Supervisor Demo](https://www.youtube.com/watch?v=OUJgLfov3Po) (~a few min) — watch the supervisor pick a sub-agent live.

**Walkthrough.** No code yet — build the mental model. Take three questions you'd ask *your* business:
- "How many units of SKU-44 are in stock?" → a **data** question (Genie over a table)
- "What's our return policy for opened items?" → a **knowledge** question (a doc/Knowledge Assistant)
- "Draft a reorder email to the supplier." → an **action** (a UC-function tool)

A supervisor's whole job is to look at each one, decide *which* of those three it is, call the right
specialist, and hand you back one answer — without you naming the tool.

**Challenge.** Write down 3 questions a coworker might ask your team, and label each as **data /
knowledge / action**. **STOP** and tell me your three questions and which specialist each should route to.

**Check.** (1) What three jobs does a supervisor do that a flat single-agent doesn't? (2) Why does
splitting into small specialists make the system *easier* to debug than one agent with all the tools?

**Source.** [Supervisor Agent (multi-agent)](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

### Lesson 6.2 — What a supervisor can coordinate
**Concept.** A supervisor's power is the *breadth* of things it can route to. On Databricks a single
supervisor can coordinate a mix of these specialist types (each needs the calling identity to hold the
listed permission — governance never disappears, it just moves up a level):

| Sub-agent / tool | What it's for | Permission needed |
|---|---|---|
| **Genie Space** | NL → SQL over your tables (stock, suppliers, sales) | `CAN VIEW` on the space |
| **Published dashboard** | answer from an existing AI/BI dashboard | `CAN VIEW` |
| **Knowledge Assistant endpoint** | RAG over your docs (policies, manuals) | `CAN QUERY` |
| **Model serving endpoint** | any served model / agent (e.g. your Module 5 deploy) | `CAN QUERY` |
| **Unity Catalog function** | run code / an API / draft text (the "action" tools) | `EXECUTE` |
| **UC tables & volumes** | read structured / file data directly | `SELECT` + schema nav |
| **AI Search index** (Delta Sync only) | vector retrieval | catalog/schema + `SELECT` |
| **Web search** | live public-web lookups (built-in) | none; per-query user approval |
| **MCP servers** (external + custom) | tools exposed over MCP (Module 3.7–3.8) | connection/app perms |
| **Custom agents** | your LangGraph/OpenAI-SDK agent on a Databricks App | `CAN_USE` on the app |
| **Nested Supervisor Agents** | a *whole other supervisor* as one sub-agent | `CAN QUERY` |

> Note the ceiling: **~20 agents per supervisor**. That's a design constraint, not a bug — past ~20
> the classify step gets noisy. When you need more, you **nest** supervisors (Lesson 6.7).

**Walkthrough.** Inventory what you already have from earlier modules: your Genie Space (3.3–3.4),
any UC-function tool (1.7 `python_exec`, or an email-drafting function), and the deployed agent from
Module 5. Those are your candidate sub-agents.

**Challenge.** List which of the types above you can actually wire up *today* in your workspace (you
have it built/permissioned). Aim for at least two. **STOP** and tell me which ones, and the one you're
missing that you'd most want.

**Check.** (1) What's the approximate per-supervisor agent limit, and what do you do when you exceed
it? (2) If a sub-agent is a UC function, what privilege must the supervisor's identity hold to call it?

**Source.** [Supervisor Agent — supported sub-agents & limits](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

### Lesson 6.3 — Create a Supervisor Agent (Agent Bricks UI)
**Concept.** Creating a supervisor is a *UI flow*, not code — Agent Bricks gives you a builder where
you assemble specialists and Databricks wires up the orchestration, evaluation, and serving endpoint
behind it. You'll have a callable agent in minutes; the *quality* work (Lessons 6.4–6.5) is where the
time goes.

**📺 Watch (optional):** [Build a Multi-Agent GenAI System in Minutes](https://www.databricks.com/resources/demos/videos/build-multi-agent-genai-system) (databricks) — and the bookstore example in [DAIWT Agent Bricks demo](https://www.databricks.com/resources/demos/videos/daiwt-multi-agent-bricks-demo) (Genie + Knowledge Assistant).

**Walkthrough.**
1. Left sidebar ▸ **Agents** ▸ **Create Agent** ▸ **Supervisor Agent**.
2. You land on the builder; on the left is the **Tools and sub-agents** panel.
3. Click to add a sub-agent — pick a type (e.g. **Genie Space**), then select your actual space.
   > ⚠️ verify: exact add-button label and whether the cap shown is "up to 20" or "up to 30 items".
4. Give it a clear **name** and **description** (the description is what the supervisor reads to decide
   when to route there — Lesson 6.4 is all about writing these well).
5. Add a second specialist (e.g. a **Unity Catalog function** with `EXECUTE`, or your Module-5
   **Model serving endpoint**).
6. Fill the supervisor's own **Instructions** (routing system prompt) and **Description** (what this
   supervisor is for).
7. Use the right-side **chat pane** to send a test question. Save.

**Challenge.** Create a supervisor named e.g. `ops-helper` with **two** sub-agents: your Genie Space +
one UC function (or your deployed agent). Give each a one-line description. Send one test question in
the chat pane. **STOP** and tell me which sub-agent it routed to and whether that was correct.

**Check.** (1) Which panel do you add specialists in, and what does the supervisor actually *read* to
decide routing? (2) Name the two text fields you fill on the supervisor itself (hint: one routes, one
summarizes its purpose).

**Source.** [Create a Supervisor Agent — UI steps](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

### Lesson 6.4 — Writing routing Instructions + sub-agent Descriptions
**Concept.** A supervisor routes on **words**, not magic. It reads (a) each sub-agent's **Description**
and (b) the supervisor's **Instructions**, then matches the user's question against them. Vague
descriptions ("handles data") cause mis-routes; sharp, *example-bearing* descriptions ("Answers
questions about **current inventory levels, supplier lead times, and stock by SKU** from the
warehouse tables") route reliably. The doc is explicit: the supervisor "uses the information in the
description to help it coordinate agents." So **the Description is the routing logic** — treat it like
a function's docstring that another LLM will read.

**Walkthrough.** Rewrite your two sub-agent descriptions to be *capability + trigger* shaped:
- Genie Space → `"Use for quantitative questions about inventory and suppliers: stock counts by SKU, supplier lead times, reorder thresholds. Input: a plain-English data question. Not for policy or email."`
- UC function (`draft_email`) → `"Use ONLY to draft an outgoing email once the data is known. Input: recipient + purpose + key facts. Does not look anything up."`

Then write the supervisor **Instructions** to set routing policy and tone, e.g.:
```
You are an operations assistant. Classify each request as DATA, KNOWLEDGE, or ACTION.
- DATA (stock, suppliers, numbers) -> the Genie Space.
- ACTION (write/draft/send) -> the email tool, but only AFTER you have the needed facts.
Never invent numbers; if a sub-agent can answer, route to it instead of guessing.
Synthesize sub-agent outputs into one concise answer. If the user lacks access to a
needed sub-agent, say so rather than attempting it.
```
Each "Not for…" / "ONLY…" clause is a *guardrail* that prevents a neighboring specialist from
stealing the call.

**Challenge.** Tighten **both** descriptions to the "capability + trigger + boundary" pattern and add
one routing rule to Instructions. Re-ask a question that previously mis-routed. **STOP** and tell me
the before/after routing.

**Check.** (1) Why is a sub-agent's *Description* effectively its routing logic? (2) What does adding a
"Not for X" boundary clause buy you when two specialists overlap?

**Source.** [Configure sub-agents & Instructions](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

### Lesson 6.5 — Test & improve: AI Playground, labeled Examples, expert feedback
**Concept.** First-draft routing is never final. Agent Bricks gives you a tight improvement loop:
chat in the **AI Playground**, capture the questions that route wrong as **Examples**, attach
**Guidelines** to each, optionally **share a link** so a domain expert can add feedback, then re-test.
This is the supervisor analog of Module 4's evaluation — but driven from real questions, not a static
dataset. The point is to convert "it routed wrong once" into a *labeled example* the system learns from.

**📺 Watch (optional):** [How to Improve Quality of Multi-Agent Systems with Agent Bricks](https://www.databricks.com/resources/demos/videos/how-to-improve-quality-of-multi-agent-systems-with-agent-bricks) (databricks).

**Walkthrough.**
1. On the agent page click **Open in Playground** (or the right-pane chat).
2. Ask 5–6 varied questions spanning all your specialists. Note any mis-routes or weak syntheses.
3. Open the **Examples** tab ▸ **+ Add** a question you want to label.
4. Click the question and add **Guidelines** describing the *desired* behavior/answer.
   > ⚠️ verify: exact label — "Guidelines" vs "expected response" in your build.
5. **Share the agent link** with a colleague who knows the domain so they can add feedback.
6. **Test again** and confirm the previously-wrong question now behaves.

**Challenge.** Find one question that routes wrong or answers weakly, add it as an **Example** with a
Guideline for what it *should* do, re-test, and confirm the change. **STOP** and tell me the question
and whether the Guideline fixed it.

**Check.** (1) Where do you turn a one-off bad answer into something the supervisor improves against?
(2) Why gather *expert* feedback via a shared link rather than judging routing quality yourself?

**Source.** [Improve the supervisor — Examples & feedback](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

### Lesson 6.6 — The Supervisor API (Beta): drive it programmatically
**Concept.** The UI is great for building; the **API** is how you embed a supervisor in an app,
script it, run it in CI, or kick off long jobs. It's **Beta** — an admin must enable it on the
**Previews** page — and it's shaped like an **OpenResponses** endpoint: you submit a request with
`tools`/`instructions`, and **Databricks runs the whole agent loop server-side** (it iterates, calls
the sub-agents, and synthesizes) before returning. You are *not* hand-rolling the loop from Module 1
here — the supervisor *is* the loop.

**Walkthrough (Beta — verify every field live).**
```python
from databricks_openai import DatabricksOpenAI          # ⚠️ verify import name
client = DatabricksOpenAI(use_ai_gateway=True)            # ⚠️ verify constructor + arg

resp = client.responses.create(                          # ⚠️ verify method
    model="databricks-claude-sonnet-4-5",                # supervisor reasoning model
    instructions="Classify DATA/KNOWLEDGE/ACTION and route; synthesize one answer.",
    input=[{"type": "message", "role": "user",
            "content": "How many units of SKU-44 are in stock?"}],  # ⚠️ verify input shape
    tools=[                                               # ⚠️ verify tool dict shapes
        {"type": "genie_space", "name": "Inventory",
         "description": "Stock counts and supplier lead times by SKU.",
         "genie_space": {"space_id": "<your-space-id>"}},
        {"type": "uc_function", "name": "draft_email",
         "description": "Draft a reorder email once facts are known.",
         "uc_function": {"name": "workspace.default.draft_email"}},
    ],
    stream=False,
)
print(resp.output_text)                                  # ⚠️ verify result accessor
```
**Long-running / async** — pass `background=True` (Beta caps a background run at **~30 min**; results
retained ~30 days), then **poll**:
```python
resp = client.responses.create(..., background=True)      # ⚠️ verify
while resp.status in {"queued", "in_progress"}:           # ⚠️ verify status values
    resp = client.responses.retrieve(resp.id)             # ⚠️ verify retrieve()
```
Beta caveats to know: you **can't** combine `stream=True` with `background=True`; **on-behalf-of-user
(OBO)** auth is **not** supported for the Supervisor API in Databricks Apps; background MCP tool calls
require explicit user approval. If you'd rather not author tool dicts by hand, an alternative is to
build the supervisor in the UI (6.3) and just call its **serving endpoint** ("Get code" gives you
curl/Python).

**Challenge.** Enable the Beta if needed, then run **either** a UI-built supervisor's endpoint ("Get
code") **or** the `responses.create` snippet above against one real sub-agent. **STOP** and tell me
what came back — and flag any field above that didn't match your workspace's actual API.

**Check.** (1) In the Supervisor API, who runs the classify→route→synthesize loop — your code or
Databricks? (2) Name two Beta limitations (hint: streaming+background, OBO, run length, retention).

**Source.** [Supervisor API (Beta)](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/supervisor-api) — *Beta; confirm signatures live.*

---

### Lesson 6.7 — Nested supervisors & long-running tasks; the ≤20-agent limit
**Concept.** Two limits shape real designs. First, **~20 agents per supervisor** — to go bigger you
**nest**: a sub-agent can itself be a whole Supervisor Agent (it needs `CAN QUERY` like any served
endpoint). So a top "Operations" supervisor might route to a "Procurement" supervisor *and* a
"Finance" supervisor, each owning their own ≤20 specialists — a tree, not a flat list. Second,
**long-running tasks**: some routes (multi-step research, a big extraction job) take minutes, so the
API supports **background** execution (~30-min cap) with polling. Nesting also *organizes* routing:
the top supervisor only has to classify into a few broad domains, and each child does the fine-grained
routing — which keeps every classify step small and accurate.

**Walkthrough.** Sketch (don't necessarily build) a 2-level tree for your domain:
```
Operations Supervisor
├── Procurement Supervisor   (Genie: stock/suppliers, doc-extraction agent, draft_email)
└── Support Supervisor       (Knowledge Assistant: policies, web search)
```
Each child stays under 20 sub-agents; the parent routes on broad domain only. If you have a genuinely
slow specialist, note where you'd switch that call to `background=True` + poll.

**Challenge.** Draw your own 2-level supervisor tree (parent + ≥2 child supervisors, each with its
specialists). Mark which leaf, if any, is slow enough to warrant background execution. **STOP** and
share the tree.

**Check.** (1) When you hit the ~20-agent ceiling, what's the structural fix, and what permission does
a nested supervisor need? (2) Why does nesting make the *top-level* routing more reliable rather than
just bigger?

**Source.** [Nested supervisors, limits & long-running](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

### Lesson 6.8 — Worked example: a procurement supervisor
**Concept.** Time to assemble the centerpiece. A **procurement supervisor** is the canonical
three-specialist system — it shows **data + knowledge/extraction + action** working together under one
brain:
- a **Genie Space** over your stock & supplier tables (the **DATA** specialist),
- a **document-extraction agent** that pulls fields from supplier PDFs/quotes (a custom agent or
  Knowledge Assistant — the **KNOWLEDGE** specialist),
- an **email-drafting UC function** that writes the reorder note (the **ACTION** specialist).

The supervisor's job: a user says *"We're low on SKU-44 — sort it,"* and it **classifies** (data →
then action), **routes** (ask Genie for the stock level + the supplier's lead time, extract terms from
the latest quote if needed), and **synthesizes** (drafts the reorder email with the real numbers
filled in). One sentence in, one ready-to-send draft out.

**📺 Watch (optional):** [Agent Bricks Overview](https://www.databricks.com/resources/demos/videos/agent-bricks-overview) (databricks) for the end-to-end shape.

**Walkthrough.**
1. **Genie** — reuse your Module 3.3–3.4 space over `inventory` / `suppliers` (`CAN VIEW`).
   Description: *"Stock by SKU, reorder thresholds, supplier lead times."*
2. **Extraction** — a custom agent or Knowledge Assistant over supplier quote docs (`CAN QUERY`).
   Description: *"Extract price, MOQ, and lead time from a supplier quote. Input: a quote/doc reference."*
3. **Email** — a UC function `workspace.default.draft_email(recipient, subject, body_facts)` (`EXECUTE`).
   Description: *"Draft (not send) a reorder email once SKU, quantity, and supplier are known."*
4. Create a **Supervisor Agent** (6.3), add all three, write Instructions:
   ```
   Goal: keep stock above reorder thresholds with minimal user effort.
   1. For any "we're low / reorder / how much stock" request, ASK THE GENIE SPACE for the
      current level and the supplier's lead time first.
   2. If supplier terms (price/MOQ) are needed and not known, use the extraction agent.
   3. Only once SKU + quantity + supplier are known, use draft_email to produce a draft.
   Never send; always return the draft for human review. Never invent stock numbers.
   ```
5. Test in the Playground with *"We're low on SKU-44 — sort it."* Watch it route Genie → (extract) →
   draft_email. Capture any mis-route as an **Example** (6.5).

**Challenge.** Build this three-specialist procurement supervisor (or the closest version your
workspace allows — substitute a `python_exec`-style UC function for the email tool if needed). Run the
"we're low on SKU-44" prompt and inspect the routing in the trace. **STOP** and report: did it hit
Genie first, then the action tool, and produce a sensible draft?

**Check.** (1) Map each of the three specialists to DATA / KNOWLEDGE / ACTION. (2) Why does the
Instruction force "ask Genie *first*, draft *only once* facts are known" — what failure does that
ordering prevent?

**Source.** [Supervisor Agent (multi-agent)](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor) · permissions per [Module 1.8 grants](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

## Module 6 wrap
By here the learner can: explain the classify→route→synthesize pattern; inventory the ~11 specialist
types a supervisor can coordinate (and their permissions); **build a Supervisor Agent in the Agent
Bricks UI**; write sub-agent Descriptions and supervisor Instructions that route reliably; improve it
from labeled Examples + expert feedback; drive it through the **Beta Supervisor API** (including
background/long-running runs); nest supervisors past the ~20-agent limit; and assemble a full
**procurement supervisor** (Genie + extraction + email). Next: **Module 7 — Governance & Production**,
where this supervisor (and its sub-agents) get the Unity Catalog permissions, token hygiene,
monitoring, and cost controls needed to ship it safely.
