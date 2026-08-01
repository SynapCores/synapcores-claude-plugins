---
name: build-with-synapcores
description: >-
  Use when building application code that integrates with SynapCores AIDB
  (Node.js / Python / REST / MySQL wire / MCP). Covers what the SDKs
  actually expose — plain SQL, graph (Cypher), NL→SQL, transactions,
  recipes, schema introspection, filesystem RAG, vector search, agent
  memory (MEMORY_*), the agentic SQL function AGENT_RUN, durable agents
  (CREATE AGENT), and streaming chat. Shows production-ready patterns:
  auth, error handling, long-running queries, response-envelope
  unwrapping, and the Docker shape that operators actually run. Invoke
  whenever a user wants to "build on SynapCores", "use SynapCores from
  Node/Python", "write an app that calls AGENT_RUN", "query the graph
  from Python", "connect a MySQL driver to SynapCores", etc.
---

# Build Production Apps on SynapCores AIDB

> **Verified against:** engine **v1.14.0-ce** (current `:latest`), Python SDK
> **0.5.0** (PyPI), Node SDK **0.6.1** (npm) — checked 2026-08-01 against a
> **running v1.14.0-ce gateway** (route table + live MCP `tools/list`) and both
> installed SDK packages. When a claim below is engine-version-gated, the version
> is stated inline. (The two SDKs still trail the engine at 0.5.0 / 0.6.1 — new
> v1.14 surface like native vision is REST-only until the SDKs catch up.)

## The mental model in one paragraph

SynapCores is a **single-binary database** that speaks SQL but also gives you a
graph engine (Cypher), NL→SQL translation, server-side transactions with
savepoints, file/filesystem-backed RAG collections, recipe templates,
multimedia (audio/video/image/PDF) ingest, **native in-process image
description** (`POST /v1/multimodal/describe` — LLaVA vision, no cloud or
sidecar daemon, v1.14.0+), vector search (`EMBED`,
`COSINE_SIMILARITY`, `VECTOR(N)`), agent memory (`MEMORY_STORE` /
`MEMORY_RECALL`), LLM completion (`GENERATE`), a first-class agentic loop
callable as a SQL function (`AGENT_RUN(persona, task)`), and **durable agents
as database objects** (`CREATE AGENT`, v1.9.0+) that fire on a cron schedule or
on committed DML — **all on the same connection, against the same data, with
one auth token.** There is no separate vector DB, no Cypher service, no
orchestration layer, no Pinecone+LangChain+Neo4j+Redis stack. Your application
talks to one HTTP gateway (REST), optionally a MySQL wire port (v1.11.0+), and
one MCP endpoint (LLM-driven tool calls). Most code lives in `client.sql(...)`
— vector search and the AI surfaces are there when you need them, but they're
**not the front door.**

## What's actually in the SDKs

The two SDKs are **not** at parity. Python 0.5.0 covers far more of the
gateway; Node 0.6.1 is SQL + collections + a handful of sub-clients. Anything
the Node SDK lacks is still reachable over plain REST (see the appendix).

| Capability | Python `synapcores` 0.5.0 | Node `@synapcores/sdk` 0.6.1 |
|---|---|---|
| Plain SQL | `client.sql(sql, params)` → **DataFrame by default** | `client.sql(q, params)` / `client.executeQuery({sql, parameters})` |
| Collections (documents) | `client.create_collection` / `get_collection` → `.insert/.search/.vector_search/.query` | `createCollection` / `getCollection` → same shape |
| Embeddings | `client.embed(text)` | `client.embed(text)` |
| Agent memory | `client.memory.{store,recall,forget}` | `client.memory.{store,recall,forget}` |
| Schema | `client.schema.{list_databases,list_tables,get_columns,preview_table}` | `client.schema.*` |
| Recipes | `client.recipes.{list,list_categories,list_templates,execute_template,execute,...}` | `client.recipes.*` |
| AutoML | `client.automl.{train,get_model,list_models,...}` | `client.automl.*` |
| NLP | `client.nlp.*` | `client.nlp.*` |
| Graph (Cypher) | `client.graph.cypher(q)` + `.nodes/.edges/.indexes/.algorithms/.graphs` | ❌ — use REST `POST /v1/graph/match` |
| NL → SQL | `client.nl2sql.ask(q, execute=True)` | ❌ — use REST `POST /v1/nl2sql/query` |
| Transactions | `client.transactions.begin(...)` (context manager) | ❌ — use REST `/v1/transactions/*` |
| Filesystem RAG | `client.filesystem.collections.*` | ❌ — use REST `/v1/filesystem-collections/*` |
| Chat sessions + streaming | `client.chat.sessions.*`, `client.chat.stream(...)` | ❌ — use REST `POST /v1/ai/chat/stream` (SSE) |
| Multimodal (similarity/search/join/embed) | `client.multimodal.{search,join,embed,similarity}` | ❌ — REST `/v1/multimodal/*` |
| **Native vision — describe image** (v1.14.0+) | ❌ (not in 0.5.0) — REST `POST /v1/multimodal/describe` | ❌ — REST `POST /v1/multimodal/describe` |
| MCP | `client.mcp.{invoke,batch,info}` | ❌ — REST `/v1/mcp` or the WebSocket at `/mcp` |
| Import/export, integrations, backup | ❌ | `client.import`, `client.integrations`, `client.backup` |
| `EMBED` / `COSINE_SIMILARITY` / `GENERATE` / `AGENT_RUN` / `MEMORY_*` | inline SQL via `client.sql(...)` | inline SQL via `client.sql(...)` |
| Durable agents (`CREATE AGENT`) | DDL via `client.sql(...)`; REST `/v1/agents/*` | same |

**Node SDK naming gotcha:** the exported class is **`SynapCores`**, not
`SynapCoresClient`, and it is configured with `{host, port, apiKey}` — there is
**no `baseUrl`** and **no `login()`**. Supply `apiKey` or `jwtToken` up front.

## API shape (read this once, internalize, never grep for it again)

- **Base URL:** `http://localhost:8080` local; whatever the operator deploys for cloud. Both SDKs append `/v1` themselves.
- **Auth:** two flavors, both first-class on every endpoint:
  - **JWT** — `POST /v1/auth/login` with `{username, password}` → `{access_token, ...}`. Send `Authorization: Bearer <token>`.
  - **API key** — created via `POST /v1/api-keys`; the engine generates keys prefixed **`aidb_`**. Send as `Authorization: Bearer aidb_...` (the Python SDK also accepts an `ak_`-prefixed key for forward compatibility). Recommended for server-to-server.
  - Creating a key takes a **`permission`** field — `"ReadOnly"` or `"FullAccess"` — **not** `scopes`. The published OpenAPI has said `scopes`; the handler wants `permission`. `FullAccess` is equivalent to a JWT session.
  - Default admin user is `admin`; the bootstrap password is printed in the container/server logs on first boot.
- **Response envelope:** the gateway wraps ALL success responses as `{data: ..., meta: {...}}`. Errors come as `{error: {code, message}, meta: {...}}`. **Always unwrap `body.data`** in raw HTTP clients. Both SDKs handle this automatically.
- **Endpoints you'll hit constantly** (verified against the v1.14.0-ce route table — several differ from what you'd guess):

  | Endpoint | Method | What it does |
  |---|---|---|
  | `/health`, `/version` | GET | Liveness + engine version, no auth |
  | `/v1/auth/login` | POST | Get a JWT (API keys need no exchange) |
  | `/v1/api-keys` | GET/POST | Provision keys (`permission: ReadOnly\|FullAccess`) |
  | `/v1/query/execute` | POST | Run any SQL (incl. `AGENT_RUN`, `EMBED`, `GENERATE`, `MEMORY_*`) |
  | **`/v1/graph/match`** | POST | Cypher — body is `{"sql": "<cypher>"}`. *Not* `/graph/cypher`. |
  | `/v1/graph/match/profile` | POST | Same, with a profile/plan |
  | `/v1/graph/{nodes,edges,graphs,indexes,algorithms}` | various | Direct graph CRUD |
  | **`/v1/nl2sql/query`** | POST | NL → SQL. *Not* `/nl2sql/ask` (that's the SDK method name). |
  | `/v1/transactions/*` | POST | Begin / commit / rollback / savepoint |
  | `/v1/recipes`, `/recipes/execute`, `/recipes/import`, `/recipes/registry`, `/recipes/templates/:id/execute` | various | Recipe management + import-by-link |
  | **`/v1/filesystem-collections/*`** | various | Filesystem-backed RAG. *Not* `/filesystem/collections`. |
  | `/v1/schema/databases`, `/v1/schema/tables/:t/{columns,indexes,data}` | GET | Introspection (`/data` is the row preview) |
  | `/v1/vectors/collections/:c/{vectors,search}` | POST | First-class vector collections |
  | `/v1/collections/:c/{documents,search}` | POST | Document collections |
  | `/v1/agents`, `/v1/agents/:name/{execute,enable,pause,runs}` | various | Durable agents (v1.9.0+) |
  | `/v1/ai/sessions`, `/v1/ai/chat`, **`/v1/ai/chat/stream`** | POST | Chat sessions + **SSE** streaming |
  | **`/v1/multimodal/describe`** | POST | **Native in-process image description** (LLaVA vision, v1.14.0+). Body `{"image": "<base64>", "prompt": "..."}` — both required. |
  | `/v1/multimodal/{similarity,search,join,embed}` | POST | Cross-modal similarity / retrieval / semantic join / embedding |
  | `/v1/system/vision` | GET/PUT/DELETE | Configure an *external* vision provider (optional; native describe needs no config) |
  | `/v1/multimedia/*` | various | Media ingest (audio/video/image/PDF) |
  | `/v1/mcp`, `/v1/mcp/batch`, `/v1/mcp/info` | POST/GET | MCP over HTTP |
  | `/mcp?token=<jwt>` | WebSocket | MCP JSON-RPC for LLM clients |
  | `/ws/filesystem-collections/:id/progress?token=<jwt>` | WebSocket | RAG ingestion progress |

  **There is no `/ws/ai-chat`.** Chat streaming is server-sent events over
  `POST /v1/ai/chat/stream`. The WebSocket surfaces are `/ws`, `/ws/stream/insert`,
  `/mcp`, and the filesystem-collections progress socket.

## Quickstart — Python

```bash
pip install synapcores      # 0.5.0
```

```python
from synapcores import SynapCores

client = SynapCores(host="localhost", port=8080, api_key="aidb_...")
# or: SynapCores(host="...", port=8080, username="admin", password="...")

client.sql("""
  CREATE TABLE IF NOT EXISTS customers (
    id INT PRIMARY KEY,
    name TEXT,
    arr DOUBLE,
    region TEXT
  )
""")

client.sql(
    "INSERT INTO customers (id, name, arr, region) VALUES ($1, $2, $3, $4)",
    [1, "Helios Logistics", 80000.0, "us-west"],
)

# NOTE: sql() returns a pandas DataFrame when there are rows (as_dataframe=True
# is the default). Pass as_dataframe=False for a QueryResult with .rows/.columns.
df = client.sql("SELECT name, arr FROM customers ORDER BY arr DESC LIMIT 10")
for row in df.itertuples():
    print(f"{row.name}: ${row.arr:,.0f}")

result = client.sql("SELECT name, arr FROM customers", as_dataframe=False)
print(result.columns, result.rows, result.took_ms)
```

The SDK handles auth headers, response unwrapping, parameter binding, and
connection reuse. Use it as a context manager (`with SynapCores(...) as client:`)
or call `client.close()`.

## Quickstart — Node.js

```bash
npm install @synapcores/sdk   # 0.6.1
```

```javascript
import { SynapCores } from '@synapcores/sdk';

// host/port — NOT baseUrl. No login(); pass apiKey or jwtToken.
const client = new SynapCores({
  host: 'localhost',
  port: 8080,
  apiKey: process.env.SYNAPCORES_KEY,   // aidb_...
  timeout: 120_000,                     // AGENT_RUN/GENERATE need headroom
});

await client.sql(`
  CREATE TABLE IF NOT EXISTS customers (
    id INT PRIMARY KEY, name TEXT, arr DOUBLE, region TEXT
  )
`);

// executeQuery takes an object; parameters is a positional array.
await client.executeQuery({
  sql: 'INSERT INTO customers (id, name, arr, region) VALUES ($1, $2, $3, $4)',
  parameters: [1, 'Helios Logistics', 80000, 'us-west'],
});

const res = await client.executeQuery({
  sql: 'SELECT name, arr FROM customers ORDER BY arr DESC LIMIT 10',
  max_rows: 5000,
});
// QueryResult: { columns: [{name, data_type, nullable}], rows: any[][], execution_time_ms }
for (const [name, arr] of res.rows) {
  console.log(`${name}: $${arr.toLocaleString()}`);
}
```

For anything the Node SDK doesn't wrap (graph, nl2sql, transactions,
filesystem RAG, chat streaming, MCP), call REST directly — see the appendix.

## Pattern 1 — Natural language to SQL

```python
# Python — SDK method is ask(), REST path is /v1/nl2sql/query
ans = client.nl2sql.ask("top 10 customers by ARR this quarter", execute=True)
print(ans["sql"])
print(ans.get("rows"))
```

```javascript
// Node — no SDK wrapper; call REST
const ans = await fetch('http://localhost:8080/v1/nl2sql/query', {
  method: 'POST',
  headers: { 'content-type': 'application/json', authorization: `Bearer ${KEY}` },
  body: JSON.stringify({ question: 'top 10 customers by ARR this quarter', execute: true }),
}).then(r => r.json()).then(b => b.data);
```

`ask()` also accepts `database`, `dialect`, `tables`, `context`, `session_id`.
The translator is grounded in your live schema. Two-step (`execute=False`, show
the SQL, confirm, then re-run with `execute=True`) is the safer pattern for
write surfaces.

## Pattern 2 — Graph (Cypher) on the same data

The graph engine is co-located with the row tables — nodes/edges live in the
same RocksDB, accessed through the same connection. No ETL, no sync job.

```python
# Python. Body sent to the gateway is {"sql": <cypher>} against /v1/graph/match.
result = client.graph.cypher("""
  MATCH (c:Customer {key: 'helios-logistics'})-[:HAS_CONTACT]->(p:Person)-[:AUTHORED]->(post:SocialPost)
  WHERE post.churn_intent_score > 0.7
  RETURN p.name, post.body, post.churn_intent_score
  ORDER BY post.churn_intent_score DESC
""")
for row in result["rows"]:
    print(row)
```

**Parameter binding is not wired through the Cypher path** — the SDK accepts a
`params` argument but does not forward it. Inline literals (escaping them
yourself) or filter in a follow-up SQL step. `client.graph.cypher(q, graph="name")`
selects a named graph; `cypher_profile(...)` returns the plan.

Node has no graph wrapper: `POST /v1/graph/match` with `{"sql": "<cypher>"}`.

Mix freely — a Cypher MATCH followed by a regular `client.sql(...)` against the
same customer table is one tenant scope, one connection.

## Pattern 3 — Transactions with savepoints

```python
# Python — Tx is a context manager: commit on clean exit, rollback on exception.
with client.transactions.begin(isolation_level="SERIALIZABLE") as tx:
    tx.execute("UPDATE accounts SET balance = balance - $1 WHERE id = $2", [100, "a"])
    tx.savepoint("after_debit")
    tx.execute("UPDATE accounts SET balance = balance + $1 WHERE id = $2", [100, "b"])
    # tx.rollback_to("after_debit") rolls back just the credit half.
```

Node has no transactions sub-client in 0.6.1 — drive `/v1/transactions/*` over
REST, or use `client.executeBatchQueries({queries, transactional: true})` for
the common "several statements, all-or-nothing" case.

Read-only workloads can stay outside a transaction — the engine auto-commits.

## Pattern 4 — Recipes (pre-built workflows)

Recipes are versioned markdown packages bundling schema + seed + workflow SQL
into one importable unit. Use them when a use case has been done before (RAG
over docs, return-fraud triage, churn scoring, agentic triage):

```python
categories = client.recipes.list_categories()
templates  = client.recipes.list_templates()          # no category filter arg
result     = client.recipes.execute_template(
    "agentic-customer-churn-triage",
    parameters={"customer_id": 42},
)
```

Recipes are **decoupled from the binary** since v1.6.6.7 — `POST /v1/recipes/import`
imports a third-party recipe from a URL, and `GET /v1/recipes/registry` lists
the published registry. The catalog lives at https://synapcores.com/recipes/.

## Pattern 5 — Schema introspection (for tooling + UIs)

```python
dbs     = client.schema.list_databases()
tables  = client.schema.list_tables()
cols    = client.schema.get_columns("customers")   # [{name, data_type, nullable}, ...]
preview = client.schema.preview_table("customers", limit=5)
```

Skip raw `DESCRIBE` / `INFORMATION_SCHEMA` — the SDK normalises metadata across
the storage engines (row / columnar / immutable). Under the hood the preview is
`GET /v1/schema/tables/:table/data`.

## Pattern 6 — Filesystem-backed RAG with progress events

Point at a directory; get a watched, chunked, embedded collection:

```python
fs = client.filesystem.collections.create(
    name="docs",
    path="/data/docs",
    watch=True,        # re-ingest on file change
)
for evt in client.filesystem.collections.subscribe_progress(fs["id"]):
    print(evt.get("status"), evt.get("progress"), evt.get("filename"))

docs = client.filesystem.collections.documents(fs["id"])
```

**How retrieval actually works:** chunks land in a per-collection *vector
space*, not a SQL table. There is **no `rag_search()` SQL function** — `rag_search`
is a tool available to the in-database agent. So you retrieve one of three ways:

1. **Let an agent do it** — `SELECT AGENT_RUN('aidb-assistant', 'Use rag_search on docs to answer: ...')`, or the chat surface, which calls `rag_search` for you.
2. **Own the embedding table** — for full SQL control over retrieval, ingest into your own table with `EMBED` and query with `COSINE_SIMILARITY` (Pattern 7a). This is the most predictable option and what most production apps end up doing.
3. **CSV files** in a watched directory are additionally materialised as SQL tables named `{collection}_{filename_stem}` — query those directly.

The watcher is server-side; your app doesn't need a long-running ingestion process.

## Pattern 7 — Vector search

Two shapes; pick the one that matches your data model.

**(a) Embedding column on a row table** — when the vector is one column among many:

```python
client.sql("""
  CREATE TABLE docs (
    id INT PRIMARY KEY, title TEXT, body TEXT,
    embedding VECTOR(384)
  )
""")
client.sql(
    "INSERT INTO docs (id, title, body, embedding) VALUES ($1, $2, $3, EMBED($3))",
    [1, "Return Policy", "30 days for unopened items..."],
)
hits = client.sql("""
  SELECT title, COSINE_SIMILARITY(embedding, EMBED($1)) AS sim
  FROM docs
  ORDER BY sim DESC LIMIT 5
""", ["how long for refunds?"])
```

For >10K rows, add an HNSW index: `CREATE INDEX docs_emb_hnsw ON docs(embedding) USING HNSW`.

**(b) Collections** — when the document/vector IS the data:

```python
col = client.create_collection("product-docs", schema={...})
col.insert([{"id": "p1", "text": "...", "metadata": {"category": "shoes"}}])
hits = col.search("waterproof running shoes", top_k=5, filter={"category": "shoes"})
vec_hits = col.vector_search(vector=[...], top_k=5)
```

There is **no `create_vector_collection()` / `vector_collection()`** on the SDK
client. The raw vector subsystem (explicit dimensions + distance metric) is REST
only: `POST /v1/vectors/collections`, then `/v1/vectors/collections/:c/vectors`
and `/v1/vectors/collections/:c/search`.

## Pattern 8 — Agent memory (`MEMORY_*`, v1.8.5+)

Purpose-built for chatbots and agents that must remember across sessions. The
engine auto-creates `_memory_<namespace>` on first write (384-dim embedding by
default), so there's no schema step:

```python
mem_id = client.memory.store("conv_42", "User prefers email over phone",
                             metadata={"importance": 0.9})
for rec in client.memory.recall("conv_42", "how should I contact them?", top_k=3):
    print(rec.content, rec.similarity)
client.memory.forget("conv_42", mem_id)
```

```javascript
const id = await client.memory.store('conv_42', 'User prefers email over phone');
const hits = await client.memory.recall('conv_42', 'contact preference', { topK: 3 });
```

In SQL, `MEMORY_RECALL` is **table-valued** — it goes in `FROM`:

```sql
SELECT id, content, similarity
FROM MEMORY_RECALL('conv_42', 'what did they say about pricing', 5);
```

`MEMORY_UPSERT` (v1.8.9+) is the idempotent write: it matches by natural key or
by semantic similarity (default 0.95 cosine) and applies a policy —
`replace`, `replace_higher_confidence`, `merge_max_confidence`, `append_history`,
`noop_if_equal` — returning `'ADD' | 'UPDATE' | 'DELETE' | 'NOOP'`. Every
non-NOOP call is appended to `_system_agent_memory_audit`.

```sql
SELECT MEMORY_UPSERT('default', 'User is pescatarian',
  json_object('key','dietary','policy','replace_higher_confidence','confidence',0.95)) AS action;
```

Namespaces must match `^[A-Za-z_][A-Za-z0-9_]*$`. `MEMORY_STORE` is an
unconditional insert unless you pass `json_object('dedup', true)`.

## Pattern 9 — RAG chatbot: recall + generate in one statement

```sql
CREATE TABLE chat_memory (
  memory_id   INTEGER PRIMARY KEY,
  agent_id    TEXT,
  content     TEXT,
  importance  DOUBLE,
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  embedding   VECTOR(384)
);
```

```python
client.sql("""
  INSERT INTO chat_memory (memory_id, agent_id, content, importance, embedding)
  VALUES ($1, $2, $3, $4, EMBED($3))
""", [int(time.time()*1000), user_id, user_message, 0.7])

reply = client.sql("""
  WITH recalled AS (
    SELECT content, COSINE_SIMILARITY(embedding, EMBED($1)) * 0.7 + importance * 0.3 AS score
    FROM chat_memory WHERE agent_id = $2
    ORDER BY score DESC LIMIT 3
  )
  SELECT GENERATE(
    'You are a helpful assistant. Relevant memories: ' ||
    GROUP_CONCAT(content, ' || ') ||
    '. Now answer: ' || $1
  ) AS answer FROM recalled
""", [user_message, user_id], as_dataframe=False)
print(reply.rows[0][0])
```

No Pinecone, no LangChain, no Redis for memory state. (The `MEMORY_*` family in
Pattern 8 does the same thing with no schema of your own — use this hand-rolled
shape when you need custom scoring.)

## Pattern 10 — Agentic SQL with `AGENT_RUN` (v1.6.6.9+)

When the answer requires *reasoning over data* — not just retrieval — wrap the
work in `AGENT_RUN` and the database runs the ReAct loop server-side:

```python
recommendation = client.sql("""
  SELECT AGENT_RUN(
    'aidb-assistant',
    'Look at claim ' || $1 || '. Use execute_query to check that customer''s ' ||
    'prior claims for fraud patterns. Use rag_search on policy_docs to find ' ||
    'the applicable clauses. Output: recommended action with citations.'
  ) AS rec
""", [claim_id], as_dataframe=False)
```

The agent calls `execute_query`, `list_tables`, `describe_table`, and
`rag_search`, reasons, and returns a synthesised answer.

**Third argument — options (JSON object via `json_object(...)`):**

```sql
SELECT AGENT_RUN('aidb-assistant', 'Summarize open incidents',
  json_object('model','qwen2.5-coder:7b','max_iterations',7,'timeout_ms',180000)) AS out;
```

- `model` TEXT — outranks the persona and config default (v1.8.10+); an uninstalled name errors.
- `max_iterations` INT — clamped 1..10, default 5.
- `timeout_ms` INT — clamped 1000..600000.
- Out-of-range values are clamped; **unknown keys error**, so typos surface.
- `allow_writes` is **rejected here** — `AGENT_RUN`'s DB tools are always read-only. Write capability is declared once by an operator on a durable agent (below).

**Requirements:** v1.6.6.9+ engine, a tool-capable LLM in `[query.ai_service]`,
and a client timeout of 120s+ (calls take 5–60s). Returns NULL if no AI service
is wired.

**Composing with regular SQL** — one agent call per row:

```sql
WITH triaged AS (
  SELECT order_id,
         AGENT_RUN('returns-triage', 'Process return for order ' || CAST(order_id AS TEXT)) AS rec
  FROM pending_returns WHERE status = 'pending'
)
SELECT * FROM triaged;
```

## Pattern 11 — Durable agents (`CREATE AGENT`, v1.9.0+)

`AGENT_RUN` is one synchronous call. A **durable agent** is a database object:
it survives restarts, fires on a cron schedule and/or on committed DML, and
records every run in a queryable audit table.

```sql
-- Scheduled
CREATE AGENT daily_churn
  PERSONA 'retention-analyst'
  TASK 'Summarize accounts at churn risk from the last day.'
  ON SCHEDULE '0 8 * * *'
  WITH (max_iterations = 5);

-- Event-driven, allowed to write back
CREATE AGENT incident_triage
  PERSONA 'sre-triage'
  TASK 'Triage the alert in the activation row: classify severity, check related
        open incidents with execute_query, summarize into triage_notes.'
  ON INSERT INTO alerts WHERE severity IN ('high','critical')
  WITH (max_iterations = 5, allow_writes = TRUE, budget_tokens_per_day = 200000);
```

Event bindings are **AFTER semantics** — the agent observes committed rows and
enqueues asynchronously (microseconds on the write path; the LLM loop never runs
inside your INSERT) and cannot abort the DML. Agent-initiated writes never
re-fire event bindings, so no infinite loops.

`WITH` keys: `max_iterations` (1..10, default 3), `allow_writes` (default FALSE
— the only way an agent's tools may mutate), `timeout_seconds` (default 120),
`budget_tokens_per_day` (0 = unmetered), `on_budget_exhausted` (`'pause'`|`'fail'`),
`memory` (`'persistent'`|`'stateless'`), `max_retries` (default 1), `enabled`.

Manage them with `ALTER AGENT ... {ENABLE|DISABLE|SET (...)}`, `DROP AGENT`,
`SHOW AGENTS`, `DESCRIBE AGENT <name>`, `EXECUTE AGENT <name>`, or over REST at
`/v1/agents/:name/{execute,enable,pause,runs}`.

**Watch it act** — poll the audit table after an activating INSERT:

```sql
SELECT agent_name, activation, started_at, status, output, tokens_in, tokens_out, verified
FROM _system_agent_runs
ORDER BY started_at DESC;
```

`prev_hash` + `entry_hash` form a per-tenant sha256 hash-chain and `verified`
recomputes it on read (v1.9.1+), so tampering with a run in storage flips
`verified` to false — the audit trail is tamper-evident. `verified` is NULL for
pre-v1.9.1 runs.

**Community Edition caps enabled agents at 10** per install (disabled agents are
unlimited).

## Pattern 12 — Streaming chat (SSE)

For interactive UIs, `/v1/query/execute` blocks until the whole response is
ready. Use the chat subsystem for token-by-token streaming with persistent
server-side memory:

```python
session = client.chat.sessions.create(model="gpt-4o")
for chunk in client.chat.stream(session["id"], "Summarize today's alerts"):
    if chunk.get("delta"):
        print(chunk["delta"], end="", flush=True)
    elif chunk.get("done"):
        break
```

This is **server-sent events over `POST /v1/ai/chat/stream`**, not a WebSocket.
Node has no wrapper — read the SSE stream yourself from that endpoint.

`client.chat.tools.list()` / `.execute(name, args)` / `.sql(sql, params)` expose
the chat tool surface directly; `client.chat.cache.{stats,clear}` manages the
semantic cache.

**Agentic chat is ON by default since v1.12.1**: the chat runs a ReAct loop with
tools and can scaffold a first schema (non-destructive statements only — the
onboarding flow). Disable with `AIDB_CHAT_AGENTIC_MODE=false`.

## Pattern 13 — MCP integration (Claude Code, Cursor, custom agent loops)

If you're building an LLM agent yourself, connect it to SynapCores via MCP and
it gets the database tools as native function calls:

```jsonc
{
  "servers": {
    "synapcores": { "url": "ws://your-host:8080/mcp?token=<jwt>" }
  }
}
```

**Fourteen tools are registered** (v1.14.0-ce, confirmed from a live
`tools/list`): `query`, `execute`, `validate_query`, `list_tables`,
`describe_table`, `sql_manual`, `graph_query`, `semantic_search`, `embed_text`,
`generate_text`, `list_models`, `describe_model`, `train_model`, `predict`.

Changes from the v1.13 surface (if you cached the old "eight tools" list, update it):
- **Cypher now has its own tool — `graph_query`** (no more routing graph through SQL).
- **Vector + ML are exposed directly** — `semantic_search`, `embed_text`, `generate_text`, `train_model`, `predict`, `list_models`, `describe_model`.
- **`list_agents` / `execute_agent` were removed** — reach durable agents through `execute` (`CREATE AGENT` / `EXECUTE AGENT`) or the REST `/v1/agents/*` routes.
- There is still **no `rag_search` MCP tool** — retrieve via `semantic_search`, or `query`/`execute` (SQL, incl. `AGENT_RUN`).

`sql_manual` is the discovery surface: it documents the whole SynapCores SQL
dialect, including `AGENT_RUN`, `MEMORY_*`, and the `CREATE AGENT` family.

**Transport note (why a "bridge" sometimes appears):** the gateway speaks MCP
natively over **HTTP POST `/v1/mcp`** and **WebSocket `/mcp`** — nothing to
install server-side. But desktop clients that only speak MCP over **stdio**
(Claude Desktop, Cursor, etc.) need a tiny local stdio↔HTTP shim; the gateway
serves one on demand at **`GET /v1/mcp/bridge`** (`curl` it to `~/.synapcores/`
and point the client's `command` at it). Clients that can do WebSocket/remote MCP
connect straight to the URL below — no bridge.

From app code (not via an LLM), the Python SDK takes a **request object**:

```python
result = client.mcp.invoke({"tool": "query", "arguments": {"sql": "SELECT COUNT(*) FROM orders"}})
batch  = client.mcp.batch([
    {"tool": "list_tables", "arguments": {}},
    {"tool": "describe_table", "arguments": {"name": "orders"}},
])
info   = client.mcp.info()
```

## Pattern 14 — AutoML (train + predict against your tables)

```python
job = client.automl.train(
    dataset_table="customers",
    target_column="churned",
    task_type="binary_classification",
)
model = client.automl.get_model(job["model_id"])
preds = model.predict(rows=[{"arr": 80000, "tickets": 5, "nps": 4}])
```

Or inline as SQL:
`SELECT AUTOML_PREDICT('churn-model-v3', json_object('arr', arr, 'tickets', tickets)) FROM customers`.

## Pattern 15 — MySQL wire protocol (v1.11.0+)

Existing MySQL drivers and tools can talk to SynapCores directly — useful for
BI tools, migration utilities, and ORMs you can't change.

```toml
# gateway.toml — OFF by default
[mysql_wire]
enabled = true
bind = "127.0.0.1"     # default
port = 3307            # default: 3307, NOT 3306, so it coexists with a real MySQL
require_tls = false    # true rejects any client that won't negotiate SSL
```

```bash
mysql -h 127.0.0.1 -P 3307 -u admin -p    # password = an API key (aidb_...)
```

Auth is `mysql_native_password` against your **API key as the password** (the
engine stores `SHA1(SHA1(key))` at key-creation time). TLS uses the gateway's
`[server] tls_cert_path`/`tls_key_path`; with `require_tls = true` and no
certificate configured the listener refuses to start rather than accept
plaintext. The wire layer is a front-end onto the same execution path as REST —
same data, same tenant scoping.

MySQL-compatibility surface worth knowing: `INSERT ... ON DUPLICATE KEY UPDATE`,
PRIMARY KEY/UNIQUE/CHECK enforcement, `JSON_EXTRACT` + `->`/`->>`,
`JSON_SET/INSERT/REPLACE/REMOVE`, `JSON_CONTAINS/KEYS/LENGTH/VALID/UNQUOTE/TYPE/ARRAY`,
`ENUM`, MySQL type aliases + `ENGINE`/`DEFAULT CHARSET` options,
`ON UPDATE CURRENT_TIMESTAMP`, and `FULLTEXT` + `MATCH(...) AGAINST(...)`.
`synapcores import --from-mysqldump` migrates a dump.

## Pattern 16 — Immutable + encrypted tables (v1.12.0+)

```sql
CREATE IMMUTABLE TABLE audit_log (
  id INTEGER PRIMARY KEY, actor TEXT, action TEXT, ts TIMESTAMP
) WITH (ENCRYPTION='AES256GCM');
```

Rows can be inserted but never updated or deleted; `VERIFY TABLE audit_log` and
`VERIFY RECORD` check the chain. Encryption requires the
`AIDB_IMMUTABLE_MASTER_KEY` environment variable (hex-encoded); the DDL is
rejected fail-closed if it's missing or invalid. `'AES256GCM'` is the only
supported algorithm, `ENCRYPTION_KEY='name'` is not supported, and encrypted
immutable tables work only in the default database. Must be created empty with
an explicit column list — `CREATE IMMUTABLE TABLE ... AS SELECT` is unsupported.

## Pattern 17 — Native vision: describe an image (v1.14.0+)

New in v1.14 and the flagship feature of the release: the engine describes
images **in-process** — a bundled LLaVA model via libmtmd, **no cloud API and
no sidecar daemon**. One HTTP call; `image` (base64 bytes) and `prompt` are both
required:

```bash
curl -s -X POST http://localhost:8080/v1/multimodal/describe \
  -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d "{\"image\": \"$(base64 -w0 receipt.png)\", \"prompt\": \"Extract the merchant, date, and total.\"}"
```

```python
import base64, requests
img = base64.b64encode(open("receipt.png", "rb").read()).decode()
r = requests.post(f"{BASE}/v1/multimodal/describe", headers=H,
                  json={"image": img, "prompt": "Extract the merchant, date, and total."},
                  timeout=180).json()
print(r["data"])                      # remember: unwrap .data
```

- **REST-only** — Python SDK 0.5.0 / Node 0.6.1 predate it; call the endpoint directly.
- Runs on **CPU** (no GPU needed). Like `GENERATE`, the **first call is slow** (model load, tens of seconds); warm calls are faster — use a 180s+ client timeout.
- **Fully local:** the image bytes never leave the box — ideal for regulated / air-gapped describe-and-extract (receipts, IDs, screenshots, chart reading).
- Prefer an *external* vision model (e.g. GPT-4o vision)? Configure a provider at `PUT /v1/system/vision`; native describe needs no config.

## Production checklist

### Authentication
- [ ] API keys (`aidb_...`) for server-to-server; JWT for browser sessions.
- [ ] Create keys with `permission: "ReadOnly"` unless the caller genuinely writes.
- [ ] On 401, re-auth and retry once. Don't infinite-loop. Never log tokens.

### Connection pooling
- [ ] Both SDKs keep one persistent HTTP client per instance. Reuse one instance app-wide.
- [ ] High-RPS: 20–50 concurrent connections (Node `maxSockets`; Python `httpx.Limits(max_connections=50)`).

### Long-running queries
- [ ] **`AGENT_RUN` takes 5–60s.** Set a 120s HTTP timeout. Durable-agent runs default to a 120s server-side budget.
- [ ] **First cold `GENERATE` is ~29s** on bundled native models (warm ~3–5s). Warm-on-boot lands this under most defaults since v1.6.6.8, but don't ship a 30s client timeout.
- [ ] Filesystem RAG ingestion is async — subscribe to progress events, don't poll.
- [ ] Show a loading indicator; provide a cancel button.

### Error handling
- [ ] SDKs raise typed exceptions (`synapcores.exceptions.*`; `SynapCoresError` subclasses in Node).
- [ ] Raw HTTP: check `body.error` before `body.data`.
- [ ] Common codes: `query_error`, `query_too_long` (SQL >100 KB), `unauthorized`, `tenant_not_found`.
- [ ] **SQL length cap: 100 KB.** Batch inserts in chunks of ≤1000 rows.

### Row caps
- [ ] HTTP responses cap at `SQL_MAX_ROW_COUNT` (default 1000). Set the env var or pass `max_rows`.
- [ ] Check the `truncated` flag before treating a result as complete.

### Vector workloads
- [ ] For >10K-row tables, create an HNSW index on the embedding column — otherwise `COSINE_SIMILARITY` is a full scan.
- [ ] Embedding dimension MUST match the configured model (384 for the default MiniLM, 768 for some Ollama embedders, 1536 for OpenAI `text-embedding-3-small`).
- [ ] A no-LLM/low-footprint deployment still needs an **embedding** model configured — vectors and `MEMORY_*` depend on it.

### Multi-tenancy
- [ ] Enterprise scopes every request to the JWT/API-key's `tenant_id`. **CE is single-tenant.**
- [ ] Don't query across tenants from app code; use the per-tenant container.

### Docker shape for production
- [ ] `synapcores/community:v1.14.0-ce` — **pin the version**, don't use `:latest`.
- [ ] Mount `gateway.toml` read-only at `/etc/synapcores/gateway.toml`.
- [ ] Mount the data volume at `/var/lib/synapcores` — persistent.
- [ ] `AIDB_ACCEPT_LICENSE=1`, `AIDB_JWT_SECRET=<32-byte-secret>`, `RUST_LOG=info`.
- [ ] `AIDB_IMMUTABLE_MASTER_KEY=<hex>` if you use encrypted immutable tables.
- [ ] `-e SQL_MAX_ROW_COUNT=100000` for large result sets.
- [ ] Reverse-proxy via nginx/Caddy for TLS termination + rate limiting.
- [ ] **Telemetry is anonymous and opt-out, on by default** (install/heartbeat/version only, no customer data). Turn it off with `[telemetry] enabled = false`, `DO_NOT_TRACK=1`, or `SYNAPCORES_TELEMETRY=off`.

## Common gotchas

**Fixed since the v1.13.0-ce era** — older copies of this skill warned about these; they're **resolved on v1.14.0-ce** (verified live), so you can stop working around them:

- `SELECT DISTINCT` and `COUNT(DISTINCT ...)` now dedupe correctly (fixed v1.13.0.1).
- The blanket "named (non-default) database drops `WHERE` matches" no-op is fixed (v1.13.0.1) — filtered `SELECT` / `DELETE` / `UPDATE` work on named databases.
- The FFmpeg mismatch on Debian 12 / Ubuntu 24.04 is handled: release tarballs are now **distro-specific** (the `…-ubuntu24` build links FFmpeg 6). Use the installer or the Docker image and you won't hit it — only a *mismatched bare tarball* on the wrong distro fails.

**Still open against v1.14.0-ce (2026-08-01)** — re-check on your build:

1. **Cross-database index isolation edge.** If you run **multiple named databases** with **same-named tables whose primary keys overlap**, a PK-index lookup can occasionally resolve against the wrong database (load-dependent; full-table scans are unaffected). **The safe shapes never hit this:** one database per deployment (the default), or — if you must run several named DBs — give the tables distinct names or non-overlapping PK ranges.

**Design constraints that are not bugs:**

2. **`rag_search` is not a SQL function** — it's an agent/chat tool. `SELECT ... WHERE rag_search(...)` is not valid SQL. See Pattern 6.
3. **`EMBED` dimension must match `VECTOR(N)`.** Declaring `VECTOR(768)` with a 384-dim model configured produces rows that never match your queries.
4. **`AGENT_RUN`'s LLM must support tool calls.** `llama-3.2-1b` is too small. Use `qwen2.5-coder:7b` (native `local` provider or Ollama), or OpenAI/Anthropic/Gemini.
5. **One model serves both embed and complete** unless you set `embedding_model` explicitly — a cloud completion model like `gpt-4o` cannot embed. Set both.
6. **`WHERE` on `_system_agent_runs` has been unreliable** — filter in the client, or aggregate with `COUNT`.
7. **CE allows 10 enabled agents.** `CREATE AGENT` #11 fails until you disable one.
8. **Python `client.sql()` returns a DataFrame** when rows come back. Pass `as_dataframe=False` for `QueryResult`.
9. **Node SDK ≠ Python SDK.** Check the parity table before assuming a method exists.

## What to build first — three demo ideas in increasing depth

### 30-minute demo: NL→SQL over a CSV
- Ingest a CSV into one table.
- Expose `/ask?q=...` calling `client.nl2sql.ask(q, execute=True)`.
- Frontend: bare HTML + fetch + a textarea.
- Shows: zero SQL knowledge required to query your own data.

### 2-hour demo: RAG chatbot for your own docs
- Ingest docs into a table with `EMBED`, retrieve with `COSINE_SIMILARITY` (or a filesystem collection with `watch=True` + `AGENT_RUN`/`rag_search`).
- Store turns with `client.memory.store` and recall with `MEMORY_RECALL`.
- Frontend: minimal chat UI over `POST /v1/ai/chat/stream`.
- Shows: end-to-end RAG with persistent memory, no LangChain.

### 1-day demo: Autonomous triage
- `CREATE AGENT` bound to `ON INSERT INTO alerts WHERE severity IN ('high','critical')` with `allow_writes = TRUE`.
- Backend inserts alerts; the agent triages asynchronously.
- Frontend polls `_system_agent_runs` and renders the tamper-evident trail (`verified`).
- Shows: the database acting on its own data, with a cryptographic audit — the "one DB, no glue" story.

## Where to read more

- **SQL reference:** https://synapcores.com/posts/AI-Native-Database-SQL-reference
- **Recipes:** https://synapcores.com/recipes/
- **Engine source:** https://github.com/mataluis2k/aidb
- **Release binaries + Docker:** https://github.com/SynapCores/synapcores-releases
- **`synapcores` (PyPI 0.5.0):** https://pypi.org/project/synapcores/
- **`@synapcores/sdk` (npm 0.6.1):** https://www.npmjs.com/package/@synapcores/sdk
- **Python SDK source:** https://github.com/SynapCores/python-sdk
- **OpenClaw memory plugin:** https://www.npmjs.com/package/@synapcores/openclaw-memory
- **In-engine discovery:** call the MCP `sql_manual` tool — it is the authoritative, version-matched dialect reference.

---

## Appendix — Raw HTTP (when the SDK doesn't cover it)

The Node SDK needs this for graph, nl2sql, transactions, filesystem RAG, chat
streaming, and MCP. It's also the right shape for shell scripts and one-shot CLIs.

```python
import os, sys, requests
BASE = 'http://localhost:8080'
token = requests.post(f'{BASE}/v1/auth/login', json={
    'username': 'admin', 'password': os.environ['ADMIN_PASS'],
}, timeout=10).json()['data']['access_token']   # remember: unwrap .data

H = {'Authorization': f'Bearer {token}'}

def sql(stmt, params=None, max_rows=None, timeout=300):
    body = {'sql': stmt}
    if params is not None: body['parameters'] = params
    if max_rows is not None: body['max_rows'] = max_rows
    r = requests.post(f'{BASE}/v1/query/execute', headers=H, json=body, timeout=timeout).json()
    if r.get('error'): raise RuntimeError(r['error'])
    data = r['data']
    if data.get('truncated'):
        print(f"WARN: truncated at {data.get('max_rows', 1000)} rows", file=sys.stderr)
    return data

def cypher(q):
    r = requests.post(f'{BASE}/v1/graph/match', headers=H, json={'sql': q}, timeout=60).json()
    if r.get('error'): raise RuntimeError(r['error'])
    return r['data']

def nl2sql(question, execute=True):
    r = requests.post(f'{BASE}/v1/nl2sql/query', headers=H,
                      json={'question': question, 'execute': execute}, timeout=120).json()
    if r.get('error'): raise RuntimeError(r['error'])
    return r['data']
```

```javascript
const BASE = 'http://localhost:8080';
const H = { 'content-type': 'application/json', authorization: `Bearer ${process.env.SYNAPCORES_KEY}` };

const unwrap = async (res) => {
  const b = await res.json();
  if (b.error) throw new Error(b.error.message);
  return b.data;                                  // gateway wraps everything
};

const sql = (q, parameters, max_rows) =>
  fetch(`${BASE}/v1/query/execute`, {
    method: 'POST', headers: H,
    body: JSON.stringify({ sql: q, ...(parameters && { parameters }), ...(max_rows && { max_rows }) }),
  }).then(unwrap);

const cypher = (q) =>
  fetch(`${BASE}/v1/graph/match`, { method: 'POST', headers: H, body: JSON.stringify({ sql: q }) })
    .then(unwrap);

// SSE chat streaming — no WebSocket involved
async function* chatStream(sessionId, content) {
  const res = await fetch(`${BASE}/v1/ai/chat/stream`, {
    method: 'POST', headers: H, body: JSON.stringify({ session_id: sessionId, content }),
  });
  const reader = res.body.getReader();
  const dec = new TextDecoder();
  let buf = '';
  for (;;) {
    const { value, done } = await reader.read();
    if (done) return;
    buf += dec.decode(value, { stream: true });
    let i;
    while ((i = buf.indexOf('\n\n')) !== -1) {
      const frame = buf.slice(0, i).trim(); buf = buf.slice(i + 2);
      if (!frame) continue;
      const line = frame.startsWith('data:') ? frame.slice(5).trim() : frame;
      if (line === '[DONE]') return;
      try { yield JSON.parse(line); } catch { yield { delta: line }; }
    }
  }
}
```

Use the SDK whenever it covers the surface. The raw shape is documented here for
the cases where it doesn't.
