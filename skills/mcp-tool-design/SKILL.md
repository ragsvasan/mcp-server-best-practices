---
name: mcp-tool-design
description: >
  Production MCP tool design best practices — use this skill whenever the user is building,
  reviewing, debugging, or designing an MCP server or individual MCP tools. Trigger on:
  "build an MCP server", "add a tool to my MCP", "review my MCP tools", "MCP tool latency",
  "my MCP tool is slow", "how many MCP tools", "MCP DB hits", "MCP connection pool",
  "MCP tool description", "MCP write tool", "MCP read tool", "tool annotations", "MCP error
  handling". Also trigger when reviewing any server.py, tools/*.py, or mcp_http.py that
  contains MCP tool definitions or dispatchers.
---

# MCP Tool Design — Production Best Practices

You are advising on production MCP tool design. Apply these rules to any new tool, review,
or architecture question. Always distinguish **standard backend problems** (the solution is
known, the stakes are higher in MCP) from **MCP-native problems** (no prior art, requires
new patterns).

---

## 1. State the Traffic Model First — Always

Before any capacity estimate, DB pool size, or concurrency recommendation, state this:

> MCP servers are sequential per session. One active user = ~1 in-flight request at any moment.
> 100 users ≈ 1–2 concurrent in-flight requests. Sessions burst for ~30 seconds (8–10 calls),
> then go quiet for hours.

Any recommendation that assumes web-app concurrency (N users = N concurrent requests) will
be wrong by 10–100×. This affects pool sizing, rate limits, Cloud Run concurrency, and
background task budgets.

---

## 2. DB Hit Discipline

### Standard problem, higher stakes in MCP

N+1 queries and multiple roundtrips are textbook web engineering. Every ORM covers this.
The MCP difference: a 5-roundtrip write that costs 200ms in a web app costs 800ms in MCP
because the LLM waits synchronously with no UI to mask the latency. There is no skeleton
screen, no optimistic update, no spinner with progress.

**Rules:**

- Every write tool completes in **one transaction**. No SELECT-then-INSERT.
- Collapse with `INSERT ... ON CONFLICT DO UPDATE RETURNING *`
- Auth token verification must hit an **in-memory cache (60s TTL)** — not the DB on every
  call. A cache miss that fails under pool pressure takes down every tool call simultaneously.
- When a DB hit is unavoidable on the write path, it must be `async`. Never block the event
  loop on a DB call.

**Flag this pattern immediately:**
```python
# BAD — 3 roundtrips
row = await conn.fetchrow("SELECT id FROM x WHERE key=$1", key)
if not row:
    await conn.execute("INSERT INTO x ...")
await conn.execute("UPDATE counters SET n=n+1 ...")

# GOOD — 1 roundtrip
await conn.execute("""
    WITH ins AS (
        INSERT INTO x (...) VALUES (...)
        ON CONFLICT (key) DO UPDATE SET touched_at=now()
        RETURNING (xmax=0) AS inserted
    )
    UPDATE counters SET n = n + (SELECT inserted::int FROM ins) WHERE ...
""")
```

---

## 3. Tool Design Rules — MCP-Native

These have no direct analogue in REST API or UI design.

### Tool count ≤ 25

Past ~25 tools, LLM selection accuracy degrades — especially when descriptions overlap.
This is a cognitive load problem on the model, not a documentation problem.

**Fix:** Action dispatch — one tool with an `action` enum replaces four per-operation tools.
```python
# BAD: 4 tools
close_thread(id), reopen_thread(id), escalate_thread(id), snooze_thread(id, until)

# GOOD: 1 tool
thread_action(id, action: "close"|"reopen"|"escalate"|"snooze", until?)
```

### Descriptions are routing contracts, not documentation

The LLM reads descriptions to decide which tool to call. A description that is technically
accurate but ambiguous relative to a neighboring tool causes **silent misdirection** — the
model routes to the wrong tool, gets a plausible-looking result, and does not retry.

For every pair of tools with overlapping domains, require an adversarial routing test:
```python
assert route_tool("what did we decide about the database?") == "search_memory"
# NOT search_decisions — different scope, same domain
```

Write the description last. Ask: "In two sentences, when should the model call this tool,
and when should it call the *other* one instead?"

### Idempotency keys on every write tool

HTTP clients retry because they got a network error. LLMs retry because the call timed out
and they are unsure whether the write landed — then call again next turn with a paraphrase
of the original arguments. Without idempotency keys you get silent duplicate records.

```python
# Every write tool schema must include:
"idempotency_key": {
    "type": "string",
    "description": "Optional. Omit to auto-generate from input hash. Prevents duplicate writes on LLM retry."
}
# Handler:
key = args.get("idempotency_key") or _stable_hash(args)
# INSERT ... ON CONFLICT (idempotency_key) DO UPDATE SET touched_at=now() RETURNING *
```

### Two-stage commit for destructive tools

There is no confirmation dialog in an LLM tool call. The model calls `delete_project` the
moment it decides to, with no preview opportunity for the user.

Every destructive tool must implement:
- Call **without** `confirmed=true` → returns plain-language preview (what will be deleted,
  count, IDs, consequences). No action taken.
- Call **with** `confirmed=true` → executes.

```python
"confirmed": {
    "type": "boolean",
    "description": "Set true ONLY after showing the user the preview and receiving explicit confirmation."
}
```

---

## 4. Tool Annotations — Required on Every Tool

```python
# Apply to every tool in your registry:
TOOL_HINTS = {
    "search_memory":   {"readOnlyHint": True,  "idempotentHint": True,  "destructiveHint": False},
    "log_decision":    {"readOnlyHint": False, "idempotentHint": False, "destructiveHint": False},
    "purge_records":   {"readOnlyHint": False, "idempotentHint": False, "destructiveHint": True},
}

# Apply additionalProperties:false post-hoc to all inputSchemas — never inline
for tool in TOOLS:
    schema = tool.get("inputSchema", {})
    if schema.get("type") == "object":
        schema["additionalProperties"] = False
```

- `readOnlyHint: true` — all read/list/get/search tools
- `idempotentHint: true` — all read tools (same args = same result)
- `destructiveHint: true` — delete, purge, deprovision, revoke, suspend, leave

---

## 5. Error Class Discipline — MCP-Native

Two error channels in the spec — they are **not interchangeable**:

| Channel | When to use | Model behavior |
|---|---|---|
| JSON-RPC `-32601` at transport | Unknown tool name | Model stops, reports to user |
| `isError: true` in result | Valid call, operation failed | Model may retry with adjusted args |

**Unknown tool names must return `-32601` at the transport layer.** Returning `isError: true`
in result content for an unknown tool causes the model to retry the unknown name in a loop,
burning context window.

```python
# In your dispatcher / middleware — BEFORE handler lookup:
if name not in PUBLIC_TOOL_NAMES:
    return JSONRPCError(code=-32601, message=f"Method not found: {name}")
```

---

## 6. Connection Pool Sizing

**The idle floor formula** (standard problem, MCP-specific cause):
```
idle connections = live instances × min_size   ← driven by instance count, not user count
```

Rules:
- `min_size=1` always on small Cloud SQL tiers (db-f1-micro: 25 conn limit)
- `max_size × max-instances ≤ Cloud SQL max_connections − 5`
- `--concurrency` on Cloud Run should match `max_size` — otherwise Cloud Run never scales
  out on concurrency, only on CPU

| Cloud SQL tier | Max conn | Safe max-instances | Safe max_size |
|---|---|---|---|
| db-f1-micro | 25 | 4 | 5 |
| db-g1-small | 100 | 10 | 8 |
| db-n1-std-1 | 400 | 40 | 8 |

Rolling deploy doubles the idle floor briefly: `2 × max-instances × min_size` connections
during the transition. `min_size=1` keeps this safe on any tier.

---

## 7. Performance

### Separate executor for slow tools
A shared `ThreadPoolExecutor` means slow tools (embedding, LLM sub-calls, 2–5s) block
fast tools (`list_threads`, `get_decisions`, <50ms). Separate them:

```python
_SLOW_EXECUTOR = ThreadPoolExecutor(max_workers=4, thread_name_prefix="mcp-slow")
SLOW_TOOLS = {"index_session_fragment", "wrap_session", "search_memory"}

async def _dispatch(name, args):
    handler = DISPATCH[name]
    if name in SLOW_TOOLS:
        return await loop.run_in_executor(_SLOW_EXECUTOR, handler, args)
    return await asyncio.to_thread(handler, args)
```

### Latency gate — automate or it drifts
Run before every commit touching auth or DB path:
- Call all public tools with valid auth
- Assert avg < 500ms, p95 < 1000ms
- Exit 1 on failure, blocking deploy

### In-process embedding models — do not use
Local models (SentenceTransformer) are CPU-bound. Cold start: 2–12s. Blocks event loop.
Use a managed API (Vertex AI `text-embedding-004`: ~5–15ms, same VPC).
Cache embeddings in DB by content hash. Embed at write time as a background task, not
on the read path.

---

## 8. Security Wiring Gaps

### RLS GUC reset — `setup=` not `init=`
```python
async def _reset_rls(conn):
    await conn.execute("SELECT set_config('app.current_profile', '', false)")

pool = await asyncpg.create_pool(
    setup=_reset_rls,   # fires on EVERY pool checkout — load-bearing
    init=_reset_rls,    # fires on new physical connection only
)
```
`setup=` is the load-bearing callback. Without it, a stale GUC from a prior request
leaks to the next borrower. Unit tests never catch this — they mock the DB.

### PII scrubber — middleware, not per-tool
Wire scrubbing once at the dispatch layer for all free-text args. Per-tool wiring creates
gaps as new tools are added:

```python
_FREE_TEXT_ARGS = {"body", "content", "summary", "notes", "description", "rationale"}

def _scrub_args(args: dict) -> dict:
    return {k: (_scrub(v) if k in _FREE_TEXT_ARGS and isinstance(v, str) else v)
            for k, v in args.items()}

# In dispatch, before any handler:
args = _scrub_args(args)
```

---

## Quick Checklist for New Tools

```
[ ] Single transaction write path (no SELECT-then-INSERT)
[ ] idempotency_key parameter on all write tools
[ ] confirmed=true gate on all destructive tools
[ ] readOnlyHint / destructiveHint / idempotentHint set
[ ] additionalProperties:false on inputSchema (post-hoc loop, not inline)
[ ] Unknown tool name → -32601 at transport layer, not isError:true in result
[ ] Description tested against neighboring tools for routing ambiguity
[ ] Tool count ≤ 25 after adding (consolidate if over)
[ ] Auth token verification hits in-memory cache, not DB
[ ] Latency gate passes (avg <500ms, p95 <1000ms)
```
