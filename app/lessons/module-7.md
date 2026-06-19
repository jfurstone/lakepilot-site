# Module 7 — Governance & Production (lesson-by-lesson, v0.1)

**Objective:** take a *working* agent and make it *shippable* — lock down who can query vs.
manage it, stop leaking tokens, watch it in production, control what it costs, and run a
real go-to-prod checklist before users touch it.
**Prereqs:** Modules 1–5 (you can build, log, register, and **deploy** an agent to a serving
endpoint). Module 6 helps for the multi-agent permission table but isn't required.
**Sandbox:** SQL Editor / a notebook for grants; the **Serving** UI for endpoint permissions
and inference tables; the CLI (`databricks`) for permissions + token commands.
**Default tutor model:** `claude-haiku-4-5`.

> 📺 **Video note:** Module 7 has **no standalone official Databricks video** — governance/
> monitoring isn't covered as its own clip yet. This is a **"record-your-own" gap** (per the
> curriculum's video layer): when you build the capstone, capture a 3–5 min screen recording
> of granting `CAN QUERY`, rotating a token, and reading an inference table. No invented links.

> Format per lesson: **Concept** (why) → **Walkthrough** (run it) →
> **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.

---

### Lesson 7.1 — Unity Catalog governance for agents: Can Manage / Can Query
**Concept.** A deployed agent is a **model serving endpoint**, and an endpoint is a *securable
object* with its own access-control list — separate from the UC grants on the tools and tables
underneath it. The whole reason this matters: in dev *you* are the owner and can do everything,
so permission bugs stay invisible until a *different* identity (a teammate, an app, the agent's
own service principal) tries to use it. Endpoint ACLs have exactly four levels, and they stack:

| Ability | NO PERMISSIONS | CAN VIEW | CAN QUERY | CAN MANAGE |
|---|---|---|---|---|
| Get / List endpoint | | ✓ | ✓ | ✓ |
| **Query endpoint** | | | ✓ | ✓ |
| Update config / Delete / Modify permissions | | | | ✓ |

The rule of thumb: **end users get `CAN QUERY`** (chat with it, hit the API), **owners/operators
get `CAN MANAGE`** (edit config, delete, re-grant). Almost nobody should need more than `CAN QUERY`.

**Walkthrough.** Grant query access to a group on a deployed endpoint, two ways.

UI: **Serving → your endpoint → Permissions** (top-right) → add a user/group → pick **Can Query**.

CLI (Permissions API):
```bash
databricks permissions update serving-endpoints <endpoint-id> --json '{
  "access_control_list": [
    {"group_name": "data-scientists", "permission_level": "CAN_QUERY"}
  ]
}'
```
Read current ACLs back with:
```bash
databricks permissions get serving-endpoints <endpoint-id>
```

**Challenge.** On one of your deployed agent endpoints, grant **Can Query** to a group (or a
second user) and confirm with `permissions get` that the level reads `CAN_QUERY`. **STOP** and
paste the access-control entry you added.

**Check.** (1) A teammate can *see* your endpoint in the list but gets denied when they chat with
it — which permission level do they have, and which do they need? (2) Why is `CAN MANAGE` the
wrong default to hand out to everyone who needs to *use* the agent?

**Source.** [Serving endpoint ACLs](https://docs.databricks.com/aws/en/security/auth/access-control/) · [Create & manage serving endpoints](https://docs.databricks.com/aws/en/machine-learning/model-serving/create-manage-serving-endpoints)

---

### Lesson 7.2 — The two layers of access: endpoint ACL vs. the grants underneath
**Concept.** Granting `CAN QUERY` on the endpoint is only *half* the picture. The agent reaches
through to **subagents, tools, and data**, and **Databricks enforces the *caller's* access to
each of those** — "the supervisor has built-in access controls, so end users only access the
subagents and data they have access to. Without explicit access, the supervisor cannot return
helpful responses from a subagent." So an end user who can query the endpoint but lacks `EXECUTE`
on a tool function will get a *useless* answer, not an error you'd expect. This is the #1 source
of "it works for me but not for them" in production. The full grant map (from the Supervisor docs):

| What the agent reaches | What the caller must hold |
|---|---|
| Model serving endpoint (sub-agent) | `CAN QUERY` on the serving endpoint |
| UC function (tool) | `EXECUTE` on the function |
| UC table | `SELECT` + `USE CATALOG` + `USE SCHEMA` |
| UC volume | `READ VOLUME` + `USE CATALOG` + `USE SCHEMA` |
| Genie Space | access to the space **and** its underlying UC objects |
| Vector / AI Search index | `USE CATALOG` + `USE SCHEMA` + `SELECT` on the index |
| Published dashboard | `CAN VIEW` on the dashboard |
| External MCP server | `USE CONNECTION` on the UC connection |

Note the **endpoint creator** has a separate requirement: at deploy time they must already hold
`USE CATALOG` + `USE SCHEMA` + `EXECUTE` on each served model (and `EXECUTE` on any transitive
function dependencies) — otherwise creation fails with `PERMISSION_DENIED`.

**Walkthrough.** Pick one tool your agent calls and confirm the *caller* (not just you) can reach it:
```sql
SHOW GRANTS ON FUNCTION workspace.default.python_exec;
GRANT EXECUTE ON FUNCTION workspace.default.python_exec TO `data-scientists`;
GRANT USE CATALOG ON CATALOG workspace TO `data-scientists`;
GRANT USE SCHEMA  ON SCHEMA  workspace.default TO `data-scientists`;
```

**Challenge.** Map *your* agent: list every tool/table/Genie space it touches, and for one of
them run `SHOW GRANTS` to see who can actually reach it. **STOP** and tell me one object where the
intended end-user group is currently *missing* the required grant (or confirm all are covered).

**Check.** (1) An end user has `CAN QUERY` on the endpoint but gets vague, unhelpful answers
whenever the agent tries its database tool — what grant is probably missing, and on what object?
(2) Why isn't endpoint `CAN QUERY` enough on its own?

**Source.** [Supervisor Agent permissions](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor) · [Manage UC privileges](https://docs.databricks.com/aws/en/data-governance/unity-catalog/manage-privileges/)

---

### Lesson 7.3 — Tokens & secrets: scopes, rotation, never-commit hygiene
**Concept.** A **Personal Access Token (PAT)** is a bearer credential — anyone holding the string
*is* you. Three habits keep that from becoming an incident: (1) **scope it down** so a leaked
token can do less, (2) **rotate** it on a schedule, and (3) **never commit it** — put it in a
secret store, not in code or `.databrickscfg` that lands in git. Databricks supports **scoped
PATs** that restrict a token to specific API operations instead of full workspace access; documented
scope names include `sql`, `unity-catalog`, `scim`, `authentication`, and `model-serving`.
> Note: the **`model-serving`** scope is real — it appears verbatim in live 403 errors
> (`required scopes: model-serving` / `unity-catalog` / `workspace`). The token-creation UI
> *groups* scopes ("BI Tools" vs "Other APIs"), which is why they don't show up as a tidy set of
> checkboxes in the *Generate new token* picker.

Key facts that drive the habits: **`authentication`-scoped tokens can mint new tokens of any
scope** (treat as god-mode — avoid), Databricks **auto-revokes a PAT unused for 90 days**, and a
token's lifetime is set in days up to a workspace max (typically **730 days** — don't use the max).

**Walkthrough.**
1. Generate a **scoped** token: **Settings → Developer → Access tokens → Manage → Generate new
   token**, set a short **lifetime** (e.g. 30 days), and under **Other APIs** pick only the scopes
   the job needs (e.g. `unity-catalog`) — not a full-access token.
2. Store it as a **secret**, not in source:
   ```bash
   databricks secrets create-scope agent-prod
   databricks secrets put-secret agent-prod pat   # paste the token at the prompt
   ```
   Read it in code with `dbutils.secrets.get("agent-prod", "pat")` — the value is redacted in logs.
3. **Rotate:** generate a new token, update the secret, then **revoke the old one**:
   ```bash
   databricks tokens list
   databricks tokens revoke --token-id <old-token-id>
   ```

**Challenge.** Create a **scoped, short-lifetime** token, put it in a secret scope, and read it
back via `dbutils.secrets.get` (confirm it prints redacted). Then **STOP** and tell me which scopes
you selected and the lifetime you set — and whether `.gitignore` covers `.databrickscfg`.

**Check.** (1) Why is an `authentication`-scoped token especially dangerous if leaked? (2) Name two
things that make a leaked token *less* damaging — one about scope, one about time.

**Source.** [Authenticate with PATs](https://docs.databricks.com/aws/en/dev-tools/auth/pat) · [Secret management](https://docs.databricks.com/aws/en/security/secrets/)

---

### Lesson 7.4 — Monitor in production: inference tables & payload logging
**Concept.** Once real users hit the agent, "it returned an answer" isn't enough — you need to know
*what* it answered, how often it failed, how slow it was, and what it cost. **Inference tables**
do this automatically: every request/response to a serving endpoint is logged to a **Unity Catalog
Delta table**, so monitoring is just SQL over your own data. Agents deployed via `agents.deploy(...)`
get payload logging wired up; for plain endpoints you enable it explicitly. The default table is
`<catalog>.<schema>.<endpoint-name>_payload`, and the columns you'll actually query are:

| Column | Use |
|---|---|
| `databricks_request_id` | join/trace a single call |
| `timestamp_ms` | when (epoch ms) |
| `status_code` | HTTP result — **filter `!= 200` to find failures** |
| `request` / `response` | raw JSON in/out |
| `execution_time_ms` | latency per call |
| `request_metadata` | endpoint, model name, version |

Two operational truths to bake in: logging is **best-effort, ~within 1 hour** (not real-time —
don't build alerting that assumes instant rows), and large payloads can be dropped.
> ⚠️ verify: a **1 MiB** per-payload cap (request/response/trace) with
> `MAX_REQUEST_SIZE_EXCEEDED` / `MAX_RESPONSE_SIZE_EXCEEDED` in a `logging_error_codes` column is
> documented for the **AI-Gateway / agent** inference-table variant; the base model-serving
> inference-table page did not restate the exact byte limit. Confirm the limit + column name
> against your endpoint type before relying on it.

**Walkthrough.**
1. Enable on an endpoint: **Serving → endpoint → Edit configuration → Enable inference tables**,
   pick a **catalog + schema** (table name defaults to `<endpoint>_payload`).
2. Wait up to an hour after sending traffic, then query failures and latency:
   ```sql
   SELECT timestamp_ms, status_code, execution_time_ms, request
   FROM   workspace.default.my_agent_payload
   WHERE  status_code <> 200
   ORDER  BY timestamp_ms DESC
   LIMIT  20;

   SELECT percentile(execution_time_ms, 0.95) AS p95_ms, count(*) AS calls
   FROM   workspace.default.my_agent_payload;
   ```
3. For richer quality scoring on a *sample* of live traffic, Databricks also offers **production
   monitoring** that runs MLflow scorers (e.g. `Safety().register(...).start(...)`) and attaches
   results to traces — layer that on once payload logging is flowing.

**Challenge.** Enable inference tables on a deployed endpoint, send it a few requests, wait, then
run the `status_code <> 200` query. **STOP** and tell me how many non-200s you got (and the p95
latency if there's traffic). If the table's empty, confirm you waited the ~1-hour delivery window.

**Check.** (1) Why is filtering on `status_code` the fastest way to find production failures? (2) Why
is it a mistake to build a "page me the instant the agent errors" alert directly on the inference
table?

**Source.** [Inference tables for monitoring & debugging](https://docs.databricks.com/aws/en/machine-learning/model-serving/inference-tables) · [Monitor agents (production monitoring)](https://docs.databricks.com/aws/en/generative-ai/agent-evaluation/monitoring-agent-framework)

---

### Lesson 7.5 — Cost management: model choice, serving DBUs, scale-to-zero
**Concept.** Two dials drive an agent's bill, and they're independent. (1) **Which model** answers:
the Claude family is **Haiku → Sonnet → Opus** in ascending capability *and* price — Haiku is the
cheap default that handles routing/classification/most tool-use; reserve Sonnet for harder reasoning
and Opus for the genuinely hard minority. Picking Opus "to be safe" can multiply token cost for no
quality gain. (2) **The serving endpoint itself** consumes **DBUs** (Databricks Units — the metered
billing unit) for as long as it's *provisioned*, whether or not anyone's calling it. The fix for idle
cost is **scale-to-zero**: the endpoint spins compute down to zero when idle and back up on the next
request. The tradeoff is a **cold-start delay** on that first request — fine for internal/bursty
agents, not for latency-critical always-on traffic. The tutor itself reflects this: lessons default
to `claude-haiku-4-5` and only "bump to Sonnet when needed."
> ⚠️ verify: exact per-model token prices and per-endpoint DBU rates change — pull live numbers
> from the Databricks **Model Serving pricing** page before quoting figures; teach the *ordering*
> and the *levers*, not hard dollar amounts.

**Walkthrough.**
1. **Right-size the model.** In your agent code, default the LLM to Haiku and only escalate where
   eval (Module 4) shows Haiku underperforms:
   ```python
   LLM = "databricks-claude-haiku-4-5"   # cheap default; bump to sonnet only where eval demands
   ```
2. **Confirm scale-to-zero** at deploy. With `agents.deploy(...)` set:
   ```python
   from databricks import agents
   agents.deploy(model_name, model_version, scale_to_zero=True)
   ```
   Or in the **Serving** UI, check the endpoint's **Scale to zero** setting under compute config.
3. **Find idle spend:** an endpoint that's provisioned but shows little traffic in its inference
   table (Lesson 7.4) is a scale-to-zero candidate — or a delete candidate.

**Challenge.** For one deployed endpoint, confirm whether **scale-to-zero is on**, and state which
**model** it serves. Then make the cost call: is Haiku enough for its job, or does it genuinely need
Sonnet/Opus? **STOP** and give me your model + scale-to-zero decision with a one-line justification.

**Check.** (1) An internal agent gets ~20 requests a day at unpredictable times — scale-to-zero on or
off, and what's the user-visible cost? (2) Why is defaulting every agent to Opus a cost anti-pattern?

**Source.** [Deploy an agent (`agents.deploy`, scale-to-zero)](https://docs.databricks.com/aws/en/generative-ai/agent-framework/deploy-agent) · [Model Serving pricing / DBUs](https://www.databricks.com/product/pricing/model-serving)

---

### Lesson 7.6 — The go-to-prod checklist
**Concept.** "It works in my notebook" and "it's safe for real users" are different bars. This is the
gate to clear before you hand an endpoint to anyone — it just rolls up the four levers from this
module into a single pass/fail you can run every time you ship.

**Walkthrough — run the checklist against your agent:**

- **Identity.** Is the agent deployed under a **service principal** (not your personal PAT), so it
  doesn't break when you leave / rotate your token?
- **Endpoint ACL (7.1).** End users have **`CAN QUERY`**; only operators have **`CAN MANAGE`**. No
  one over-privileged.
- **Underlying grants (7.2).** Every tool/table/Genie space/index the agent touches is granted to the
  end-user group (`EXECUTE` / `SELECT` + `USE CATALOG`/`USE SCHEMA`, etc.) — verified with
  `SHOW GRANTS`, not assumed.
- **Secrets (7.3).** No token in code or committed `.databrickscfg`; credentials live in a **secret
  scope**; tokens are **scoped + short-lived** with a rotation owner.
- **Monitoring (7.4).** **Inference tables enabled**; you've run the `status_code <> 200` and p95
  latency queries at least once and know where to look when it misbehaves.
- **Cost (7.5).** Model is **right-sized** (Haiku unless eval says otherwise); **scale-to-zero**
  decided deliberately; idle/abandoned endpoints deleted.
- **Quality gate (Module 4).** Evaluated against a real dataset; you know its failure modes before
  users find them.

**Challenge.** Run the full checklist against one real deployed agent and mark each line ✅ / ❌ /
⚠️. **STOP** and give me the line that's most *not* ready — that's your next task before shipping.

**Check.** (1) Why deploy under a service principal instead of your personal token? (2) Which single
checklist item, if skipped, most often causes the silent "works for me, broken for them" failure —
and why?

**Source.** [Deploy an agent](https://docs.databricks.com/aws/en/generative-ai/agent-framework/deploy-agent) · [Supervisor Agent permissions](https://docs.databricks.com/aws/en/generative-ai/agent-bricks/multi-agent-supervisor)

---

## Module 7 wrap
By here the learner can govern a deployed agent like a production system: set **Can Query / Can
Manage** on the endpoint, line up the **UC grants underneath** so it works for *other* identities,
keep **tokens scoped/rotated/out of git**, **monitor** via inference tables, **control cost** with
model choice + scale-to-zero, and clear a **go-to-prod checklist**. Next: **Module 8 — Capstone**,
where you design, build, evaluate, deploy, and hand off one real agent end to end — applying every
governance lever from this module to something you actually ship (and record your own Module 7 gap
clip while you're at it).
