# Module 3 — Tools & Data (lesson-by-lesson, v0.1)

**Objective:** give your agent real power — let it query your *structured* tables (Genie),
*author your own* tools (UC SQL/Python functions), *retrieve* from unstructured docs (Vector
Search / RAG), and *plug in* whole tool catalogs through MCP. By the end you can point an agent
at any data in your workspace and at external systems, all governed by Unity Catalog.
**Prereqs:** Module 1 (tools as UC functions, `UCFunctionToolkit`, the agent loop, grants) and
Module 2 (authoring with `ResponsesAgent`). A running serverless **SQL warehouse**, a Unity
Catalog catalog + schema you can write to (`workspace.default` is fine), and the local 3.11
venv with `databricks-openai` + `databricks-mcp` (both installed). `databricks-vectorsearch`
is an **optional** add (`pip install databricks-vectorsearch`) — only needed to *build* an index in 3.5.
**Sandbox:** a Databricks notebook is easiest; SQL snippets run in a SQL cell or the SQL Editor.
**Default tutor model:** `claude-haiku-4-5`.

> Format per lesson: **Concept** (why) → **Watch** (optional) → **Walkthrough** (run it) →
> **Challenge** (do it, then STOP) → **Check** (1–2 questions) → **Source**.

> Note on the two Vector Search packages (verified 2026-06-17): the **retriever tool** an agent
> calls comes from `databricks_openai` (`from databricks_openai import VectorSearchRetrieverTool`),
> which is installed in this stack. **Creating/managing** an index uses a *separate, optional*
> package — `databricks-vectorsearch` (`from databricks.vector_search.client import
> VectorSearchClient`) — which is **not** installed by default here; `pip install
> databricks-vectorsearch` when you need to build an index. `databricks-langchain` is **not** the
> path for either. (Databricks is mid-rebrand of Vector Search → "AI Search," but these import names hold.)

---

### Lesson 3.1 — The four kinds of tool an agent can hold
**Concept.** Every useful agent tool falls into one of four buckets, and knowing which you need tells you *which Databricks feature to reach for*:
1. **Structured data** — query rows in governed tables → **Genie Spaces** (NL→SQL) or a UC SQL function.
2. **Unstructured data** — find passages in docs/PDFs/wikis → **Vector Search** retrieval (RAG).
3. **Run code / compute** — execute logic the model shouldn't guess → a **UC Python UDF** (like `python_exec` from Module 1).
4. **External API / system** — reach outside Databricks → a **UC connection / external MCP server**.
All four are exposed to the model the *same way* — as a callable tool with a name + JSON schema — so the agent loop from Module 1 never changes. What changes is the *kind of thing* behind the tool.

**Walkthrough.** Sort the tools you already have. In a notebook:
```python
# Built-ins and your own functions all live in Unity Catalog — list what's in your schema.
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
for f in w.functions.list(catalog_name="workspace", schema_name="default"):
    print(f.full_name, "→ returns", f.data_type)
```
`python_exec` (Module 1) is bucket 3. By the end of this module you'll have added bucket 1 (Genie), bucket 2 (Vector Search), and bucket 4 (MCP).

**Challenge.** Pick the agent you'd actually want to build (a support bot, a sales-data Q&A, etc.) and write down which of the four buckets each of its tools falls into. **STOP** and tell me your agent idea and its tool buckets.

**Check.** (1) Which bucket does "answer questions over our PDF handbook" fall into, and which Databricks feature serves it? (2) Why does the agent loop stay identical no matter which bucket a tool comes from?

**Source.** [AI agent tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/agent-tool)

---

### Lesson 3.2 — Author your own UC function tools (SQL + Python UDFs)
**Concept.** The most portable tool on Databricks is a **Unity Catalog function**: governed, versioned, callable by any agent, and described to the LLM automatically by `UCFunctionToolkit` (Module 1.3). A **SQL UDF** is perfect for a *parameterized query* over your tables (bucket 1 without the full Genie experience); a **Python UDF** is for *computation/logic* (bucket 3). The function's comment and parameter comments become the tool's description the model reads — so write them like you're talking to the model.

**Walkthrough.** A SQL function tool — "look up a customer's lifetime revenue":
```sql
CREATE OR REPLACE FUNCTION workspace.default.customer_ltv(customer_id STRING COMMENT 'the customer id to look up')
RETURNS TABLE(total_revenue DOUBLE, order_count BIGINT)
COMMENT 'Returns lifetime revenue and order count for a single customer id.'
RETURN
  SELECT sum(amount) AS total_revenue, count(*) AS order_count
  FROM workspace.default.orders
  WHERE orders.customer_id = customer_ltv.customer_id;
```
A Python function tool — "compute a date N business days from today":
```sql
CREATE OR REPLACE FUNCTION workspace.default.add_business_days(n INT COMMENT 'number of business days to add')
RETURNS STRING
LANGUAGE PYTHON
COMMENT 'Returns the ISO date that is n business days after today (skips weekends).'
AS $$
  import datetime
  d, added = datetime.date.today(), 0
  while added < n:
      d += datetime.timedelta(days=1)
      if d.weekday() < 5:
          added += 1
  return d.isoformat()
$$;
```
Then expose both to an agent exactly as in Module 1.3:
```python
from databricks_openai import UCFunctionToolkit, DatabricksFunctionClient
client = DatabricksFunctionClient()
tools = UCFunctionToolkit(
    function_names=["workspace.default.customer_ltv", "workspace.default.add_business_days"],
    client=client,
).tools
print([t["function"]["name"] for t in tools])  # dots → double underscores
```

**Challenge.** Create the `add_business_days` function (it needs no tables), test it with `SELECT workspace.default.add_business_days(5);`, then build a `UCFunctionToolkit` over it and print the generated tool name + description. **STOP** and paste the tool schema's `description` field — confirm it matches your SQL `COMMENT`.

**Check.** (1) Where does the *tool description* the model sees come from in a UC function? (2) When would you choose a SQL UDF over a Python UDF for a data-lookup tool — and vice versa?

**Source.** [Create UC function tools for agents](https://docs.databricks.com/aws/en/generative-ai/agent-framework/create-custom-tool) · [CREATE FUNCTION (SQL)](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-sql-function)

---

### Lesson 3.3 — Genie Spaces: natural language → SQL over your tables
**Concept.** A **Genie Space** is a domain-scoped, natural-language chat interface over a set of Unity Catalog tables: a business user asks "what were Q2 sales by region?" and Genie writes + runs the SQL and returns the table/chart. It's Databricks' answer to "the DatabaseQuerier" — instead of you hand-writing a SQL UDF per question (3.2), Genie *generates* the SQL on demand, steered by **instructions**, **example SQL queries**, and **trusted assets** you curate. The curation is the whole game: a Genie Space is only as good as the business context you give it.

**📺 Watch (optional, then DO):** [Creating an AI/BI Genie Space](https://www.databricks.com/resources/demos/videos/data-warehouse/creating-ai-bi-genie-space) (databricks)

**Walkthrough.** Build one in the UI (no code needed to *create* it):
1. Sidebar → **Genie** → **New**.
2. **Select data** — pick a SQL **warehouse** (serverless recommended) and up to ~**30 tables/views** from Unity Catalog. You need `SELECT` on the data and `CAN USE` on the warehouse.
3. Open the space and **ask a question** in plain English; Genie shows the SQL it generated + the result.
4. **Curate** via the space's config / knowledge:
   - **General instructions** — org terminology ("'revenue' means `net_amount`, exclude refunds").
   - **Example SQL queries** — paste a few correct queries; Genie learns the shapes.
   - **Trusted assets / SQL expressions** — reusable, verified expressions Genie can rely on.
5. Note the **Space ID** from the URL (`.../genie/rooms/<space_id>`) — you'll need it in 3.4.

**Challenge.** Create a Genie Space over one or two small tables you own, ask it one real question, then add **one general instruction** that corrects a term it got wrong and re-ask. **STOP** and tell me the question, the SQL Genie generated, and whether the instruction changed the result.

**Check.** (1) What three kinds of curation make a Genie Space more accurate? (2) Genie can only touch tables you gave it `SELECT` on and that are registered where — and why does that matter for governance?

**Source.** [AI/BI Genie overview](https://docs.databricks.com/aws/en/genie/) · [Set up a Genie Space](https://docs.databricks.com/aws/en/genie/set-up)

---

### Lesson 3.4 — Connect a Genie Space as an agent tool
**Concept.** A Genie Space is great on its own, but the real unlock is letting an *agent* call it as a tool — so a supervisor or assistant can decide "this is a structured-data question" and hand it to Genie, then weave the result into a larger answer. Databricks exposes Genie to agents as a first-class tool (and, in 3.8, as a managed MCP server). The agent passes the user's natural-language question straight through; Genie does the NL→SQL.

**📺 Watch (optional):** [Genie Code in action: end-to-end demo](https://www.databricks.com/resources/demos/videos/genie-code-in-action) (databricks)

**Walkthrough.** Wire the space (from 3.3) into agent code with the `GenieAgent` helper from `databricks-langchain`:
```python
from databricks_langchain.genie import GenieAgent

genie_tool = GenieAgent(
    genie_space_id="<your_space_id>",        # from the room URL in 3.3
    genie_agent_name="sales_genie",
    description="Answers questions about sales orders and revenue by querying our warehouse tables.",
)
# genie_tool is a runnable you can invoke directly...
result = genie_tool.invoke({"messages": [{"role": "user", "content": "Total revenue last month by region?"}]})
print(result)
```
> ⚠️ Verify: the exact helper name/import has churned (`GenieAgent` in `databricks_langchain` vs. wiring Genie purely through the managed MCP server in 3.8). Confirm `pip show databricks-langchain` and the current Genie-tool doc before teaching. The **managed MCP route in 3.8 is the most future-proof** way to expose Genie to an agent.

When you later deploy this agent (Module 5), you also declare Genie as a **resource** so auto-auth works — conceptually a `DatabricksGenieSpace(genie_space_id=...)` in the agent's resources list. `> ⚠️ verify: exact resource class name.`

**Challenge.** Take the space you built in 3.3, wrap it as a tool/runnable, and invoke it once from Python (not the UI) with a real question. **STOP** and tell me the answer it returned and whether it matched the UI.

**Check.** (1) When the agent calls the Genie tool, who writes the SQL — the agent's LLM or Genie? (2) Why does a *deployed* agent need Genie declared as a resource (hint: same reason a UC function needs `EXECUTE`, Module 1.8)?

**Source.** [AI/BI Genie overview](https://docs.databricks.com/aws/en/genie/) · [Managed MCP servers (Genie)](https://docs.databricks.com/aws/en/generative-ai/mcp/managed-mcp)

---

### Lesson 3.5 — Retrieval / RAG: Vector Search indexes
**Concept.** For *unstructured* data (docs, PDFs, support tickets, wiki pages) you can't write SQL — you need **semantic retrieval**: turn text into embedding vectors and find the chunks nearest the question. **Mosaic AI Vector Search** does this: a managed **endpoint** serves an **index** built from a source Delta table, using an **embedding model** (a Foundation Model endpoint like `databricks-gte-large-en`). The cleanest type is a **Delta Sync index** — you point it at a Delta table and Databricks keeps the embeddings in sync as rows change. This is the "R" in RAG; the agent (3.6) is the "AG."

**Walkthrough.** Source table needs **Change Data Feed** on. Then create endpoint + Delta Sync index:
```python
from databricks.vector_search.client import VectorSearchClient   # pip install databricks-vectorsearch (optional, index-management only)
vsc = VectorSearchClient()

vsc.create_endpoint(name="lakepilot_demo_endpoint", endpoint_type="STANDARD")

index = vsc.create_delta_sync_index(
    endpoint_name="lakepilot_demo_endpoint",
    source_table_name="workspace.default.docs",          # has columns: id, text
    index_name="workspace.default.docs_index",
    pipeline_type="TRIGGERED",                            # or "CONTINUOUS"
    primary_key="id",
    embedding_source_column="text",                      # Databricks computes embeddings
    embedding_model_endpoint_name="databricks-gte-large-en",
)
```
Query it directly to sanity-check retrieval *before* wiring an agent:
```python
idx = vsc.get_index("lakepilot_demo_endpoint", "workspace.default.docs_index")
print(idx.similarity_search(
    query_text="how do I reset my password?",
    columns=["id", "text"],
    num_results=3,
))
```

**Challenge.** With a tiny Delta table (even 5–10 rows of text + an `id`, CDF enabled), create an endpoint + Delta Sync index and run one `similarity_search`. (Index build takes a few minutes — `idx.describe()` shows status.) **STOP** and paste the top result your query returned.

**Check.** (1) Why can't you answer "find the passage about refunds in our handbook" with a SQL function? (2) In a Delta Sync index, what keeps the embeddings current when the source table changes?

**Source.** [Mosaic AI Vector Search](https://docs.databricks.com/aws/en/vector-search/vector-search) · [Create & query an index](https://docs.databricks.com/aws/en/generative-ai/create-query-vector-search)

---

### Lesson 3.6 — Build a retrieval (RAG) agent
**Concept.** Now turn that index into a **retriever tool** and give it to an agent — the model decides when a question needs the docs, calls the tool, and grounds its answer in the retrieved passages instead of hallucinating. `databricks-openai` ships `VectorSearchRetrieverTool` (`from databricks_openai import VectorSearchRetrieverTool` — confirmed present in this stack), which wraps your index into exactly the tool-schema shape the agent loop already speaks (Module 1). (`databricks-langchain` is not the path here.)

**📺 Watch (optional):** the RAG product tour in [Build, Deploy & Assess a RAG App](https://www.databricks.com/resources/demos/tours/data-science-and-ai/mosaic-ai-agent-framework-evaluation) (databricks) — also a Module 4 reference.

**Walkthrough.** Build the retriever tool over the 3.5 index and hand it to a served LLM:
```python
from databricks_openai import VectorSearchRetrieverTool, DatabricksOpenAI

vs_tool = VectorSearchRetrieverTool(
    index_name="workspace.default.docs_index",
    num_results=3,
    columns=["id", "text"],
    tool_name="search_docs",
    tool_description="Search the internal documentation for relevant passages.",
)

client = DatabricksOpenAI()
messages = [{"role": "user", "content": "How do I reset my password?"}]
resp = client.chat.completions.create(
    model="databricks-claude-3-7-sonnet",
    messages=messages,
    tools=[vs_tool.tool],          # note: .tool is the JSON schema
)
print(resp.choices[0].message.tool_calls)   # model's decision to retrieve
```
From here it's the **same agent loop** as Module 1.4–1.5: run the tool call, append the retrieved passages as a tool message, call `create()` again for the grounded final answer. (The packaged `ResponsesAgent` version from Module 2 is how you'd ship it.)

**Challenge.** Build the `VectorSearchRetrieverTool` over your 3.5 index, run one chat-completions call with `tools=[vs_tool.tool]`, and confirm the model *chose* to call `search_docs`. **STOP** and paste the `tool_calls` (the query it sent to retrieval).

**Check.** (1) What does adding retrieval prevent the model from doing on doc questions? (2) Two distinct things happen — the model's *retrieval call* and the *passages returned*. Which one grounds the final answer, and how does it get back to the model?

**Source.** [Unstructured retrieval agent tools](https://docs.databricks.com/aws/en/generative-ai/agent-framework/unstructured-retrieval-tools) · [Retrieval agent tutorial](https://docs.databricks.com/aws/en/generative-ai/tutorials/agent-framework-notebook)

---

### Lesson 3.7 — Model Context Protocol (MCP) in plain English
**Concept.** Without MCP, every agent needs *custom glue* for every tool source — one integration for Genie, another for your vector index, another for a third-party API. **MCP** is an open standard that flips this: a tool source runs an **MCP server** that *advertises its tools* in one consistent shape, and any agent (the **MCP client**) can discover and call them without bespoke code. Think USB-C for agent tools — one port, many devices. On Databricks this means you can plug whole *catalogs* of tools into an agent at once, with **Unity Catalog permissions always enforced**. Three flavors:
- **Managed MCP servers** — zero setup; Databricks hosts them for AI Search, Genie Spaces, UC functions, Databricks SQL (3.8).
- **External MCP servers** — call third-party servers through Databricks-managed proxies + UC connections (managed OAuth).
- **Custom MCP servers on Databricks Apps** — host your own proprietary toolset.

**Walkthrough (conceptual — no run yet).** Map each Module-3 thing you built to an MCP server URL pattern:
| Tool source (this module) | Managed MCP server URL pattern |
|---|---|
| Genie Space (3.3–3.4) | `https://<host>/api/2.0/mcp/genie/{genie_space_id}` |
| Vector Search index (3.5) | `https://<host>/api/2.0/mcp/ai-search/{catalog}/{schema}/{index}` |
| UC functions (3.2) | `https://<host>/api/2.0/mcp/functions/{catalog}/{schema}/{function}` |
| Databricks SQL | `https://<host>/api/2.0/mcp/sql` |

So everything you built in 3.2–3.6 is *also* reachable as an MCP tool — same governance, one interface.

**Challenge.** No code. In your own words, write the one-sentence reason MCP exists (the "custom glue" problem it removes) and name which of the three flavors you'd use to expose **your own** internal pricing API to an agent. **STOP** and give me both.

**Check.** (1) What problem does MCP remove for an agent author? (2) When an agent calls a Databricks managed MCP tool, what still gates which data it can actually reach?

**Source.** [Model Context Protocol (MCP) on Databricks](https://docs.databricks.com/aws/en/generative-ai/mcp/)

---

### Lesson 3.8 — Add MCP tools: managed and custom servers
**Concept.** Now make MCP real: connect to a **managed** MCP server, list the tools it advertises, and call one — proving the "plug in a catalog of tools" promise. The same client pattern works for any of the managed servers (and, with a different URL, for custom/external ones). This is the most future-proof way to give an agent your Genie Space, your vector index, and your UC functions all at once.

**📺 Watch (optional):** [Agent Bricks + MCP Integration on Databricks](https://www.databricks.com/resources/demos/videos/agent-bricks-mcp-integration-databricks) (databricks)

**Walkthrough.** Connect to the UC-functions managed server and list/call tools:
```python
from databricks_mcp import DatabricksMCPClient
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()
host = w.config.host  # https://<your-workspace-host>

# Managed server exposing the built-in system.ai functions as MCP tools:
mcp_url = f"{host}/api/2.0/mcp/functions/system/ai"
mcp = DatabricksMCPClient(server_url=mcp_url, workspace_client=w)

tools = mcp.list_tools()
print([t.name for t in tools])               # the advertised tools

# Call one (shape per the listed tool's input schema):
result = mcp.call_tool("system__ai__python_exec", {"code": "print(6*7)"})
print(result)                                 # -> 42
```
Point the URL at *your own* objects to expose what you built this module:
```python
# Your Genie Space (3.3) as MCP:
genie_mcp = DatabricksMCPClient(server_url=f"{host}/api/2.0/mcp/genie/<space_id>", workspace_client=w)
# Your Vector Search index (3.5) as MCP:
vs_mcp    = DatabricksMCPClient(server_url=f"{host}/api/2.0/mcp/ai-search/workspace/default/docs_index", workspace_client=w)
```
**Custom server (concept):** host an MCP server as a **Databricks App**, then point a `DatabricksMCPClient` at its app URL — same `list_tools` / `call_tool` interface, your proprietary tools inside.
> ⚠️ Verify: exact `databricks_mcp` method names (`list_tools` / `call_tool`) and the tool-call result shape against `pip show databricks-mcp` — the package is young and signatures may shift.

**Challenge.** Connect a `DatabricksMCPClient` to a managed server (`/api/2.0/mcp/functions/system/ai` is the easiest), `list_tools()`, and `call_tool` on one of them. **STOP** and paste the tool list and the result of your call.

**Check.** (1) What's the advantage of reaching Genie + your vector index + your UC functions through MCP instead of three bespoke tool integrations? (2) For a managed MCP server, what determines which tools actually show up in `list_tools()` for a given user (hint: governance)?

**Source.** [Use Databricks managed MCP servers](https://docs.databricks.com/aws/en/generative-ai/mcp/managed-mcp) · [MCP on Databricks](https://docs.databricks.com/aws/en/generative-ai/mcp/)

---

## Module 3 wrap
By here the learner can: classify any tool into the four buckets; **author their own** SQL and
Python UC-function tools with model-readable descriptions; build and curate a **Genie Space** for
NL→SQL over their tables and call it from an agent; stand up a **Vector Search** Delta Sync index
and turn it into a **retrieval (RAG)** agent tool; explain **MCP** in plain English; and connect a
real **managed (or custom) MCP server** to list and call a catalog of tools — all under Unity
Catalog governance. Next: **Module 4 — Evaluation & Quality** — now that the agent can reach real
data and tools, you'll learn whether it's actually *good*: MLflow Tracing deep-dive, LLM-judge
Agent Evaluation, building an eval dataset, and a deterministic quality-gate layer before results ship.
