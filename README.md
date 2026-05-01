# MCP Server Infrastructure Best Practices

**Applies to:** Any MCP server backed by Cloud Run + PostgreSQL
**Derived from:** Production incidents, architecture reviews, and operational experience running MCP servers at scale

---

## 1. Traffic Model — Read This First

MCP servers are not web apps. Every capacity estimate that assumes web-app concurrency will be wrong by 10–100×.

**The actual model:**
- Each AI assistant session makes **one tool call at a time** — Claude calls a tool, waits for the result, decides, calls the next
- One active user = ~1 in-flight request at any moment
- Sessions are **burst-then-idle**: 8–10 requests over ~30 seconds, then quiet for hours
- Peak concurrent in-flight requests at N users is roughly N × 0.15–0.25 (not N)

**Planning numbers (3× stress multiplier applied):**

| Registered users | Peak RPS | Peak concurrent in-flight |
|-----------------|----------|--------------------------|
| 100             | ~0.05    | 1–2                      |
| 1,000           | ~1       | 3–5                      |
| 10,000          | ~32      | 10–20                    |
| 100,000         | ~319     | 50–100                   |
| 1,000,000       | ~3,200   | 400–600                  |

These numbers assume: 25% monthly active users, 1.2 sessions/day/MAU, 8.5 requests/session.

**Implication:** A web-app architect who sees "100 users" will assume 100 concurrent DB connections. The real number is 1–2. Every recommendation that follows flows from this.

---

## 2. Connection Pool Sizing (asyncpg + Cloud Run)

### The idle floor problem

Each live Cloud Run instance opens `min_size` database connections **on startup, regardless of traffic**. This is not request-driven. It is permanent overhead.

```
idle connections = live instances × min_size
```

At peak load, a second pool is also consumed:

```
total connections = live instances × max_size  (ceiling)
```

**The incident (2026-04-30):** One registered user. Four Cloud Run instances live (from a prior scaling event). 4 × min_size=2 = 8 idle connections. Background telemetry burst. db-f1-micro ceiling (25) hit. Auth token refresh failed → entire read side appeared broken. Root cause: instance count × min_size, not user load.

### Correct pool config for Cloud Run + PostgreSQL

```python
# backend/db.py
pool = await asyncpg.create_pool(
    dsn=DATABASE_URL,
    min_size=1,      # NOT 2 — idle floor = instances × 1, not instances × 2
    max_size=5,      # safe for db-f1-micro at max-instances=4 (4×5=20, 5 spare)
    command_timeout=25,
    max_inactive_connection_lifetime=60,
)
```

**Rules:**
- `min_size=1` always on small Cloud SQL tiers (< 100 connections). The cold-acquire penalty (~150ms) is acceptable.
- `max_size` × `max-instances` must leave at least 5 connections spare for admin, auth refresh, and background tasks.
- When you upgrade Cloud SQL tier, you can raise both — see the tier table below.

### Tier table

| Cloud SQL tier | Max connections | Safe `max-instances` | Safe `max_size` | Cost/mo |
|---------------|----------------|---------------------|----------------|---------|
| db-f1-micro   | 25             | 4                   | 5              | $9      |
| db-g1-small   | 100            | 10                  | 8              | $27     |
| db-n1-std-1   | 400            | 40                  | 8              | $49     |
| db-n1-std-2   | 800            | 80                  | 8              | $98     |

**When to upgrade:**
- db-f1-micro → db-g1-small: when MAU > 250 sustained, or when you want max-instances > 4
- db-g1-small → db-n1-std-1: when MAU > 3,000 sustained
- Add HA (regional, not zonal): before your first paying customer

---

## 3. Cloud Run Configuration

### The deploy config drift hazard

If you have more than one deploy path (CI/CD + manual scripts), they **will** diverge. When they do, whichever ran last silently owns the production config. You won't know until an incident.

**Rule:** All Cloud Run params must live in exactly one canonical file. All other deploy paths must reference or copy from it.

**Params that must be consistent across all deploy files:**
- `--max-instances`
- `--min-instances`
- `--concurrency`
- `--memory`
- `--cpu`

### Correct concurrency setting

`--concurrency` tells Cloud Run how many simultaneous requests one instance can handle before it scales out. For MCP backends:

- **Do not use concurrency=1000** — this tells Cloud Run each instance can handle 1000 simultaneous requests, which means it will never scale out based on concurrency, only on CPU. Under CPU-bound tools (embeddings, LLM calls), this can leave one instance handling all traffic while other instances drain slowly.
- **Use concurrency=8** — matches the pool size. Cloud Run scales out when an instance has 8+ in-flight, which is also when the pool is near capacity.

```yaml
# cloudbuild.yaml / deploy script — keep in sync
--min-instances=1
--max-instances=4          # raise after Cloud SQL tier upgrade
--concurrency=8            # matches pool max_size
--memory=1Gi
--cpu=1
```

### Rolling deploy connection spike

During a rolling deploy, both old and new revisions are live simultaneously. Both have pools.

```
worst-case during deploy = 2 × max-instances × min_size
```

With `min_size=1, max-instances=4`: worst case = 8 idle connections during deploy. Safe on any tier.
With `min_size=2, max-instances=4`: worst case = 16 idle connections during deploy. Hits 64% of db-f1-micro budget before any request runs.

This is why `min_size=1` matters most during deploys, not steady-state.

---

## 4. Auth Path Isolation

Never put token refresh on the same DB connection pool as data queries.

**The failure mode:** Under pool saturation, auth token refresh fails. MCP clients get 401. They retry. Retries add load. The system cannot self-recover because recovery requires the DB that is saturated.

**Fixes in priority order:**

1. **In-memory token cache with TTL** — most token verifications should hit cache, not DB. 15–60s TTL is sufficient. Instance-local is fine for beta; add LISTEN/NOTIFY cross-instance invalidation before paid customers.

2. **Bounded background task concurrency** — telemetry buffers, session writers, and other fire-and-forget tasks must have a bounded connection budget. Use a semaphore or a separate small pool (size=2) for background work.

3. **`min_size=1`** — reduces idle floor so pool pressure from a burst doesn't cascade into auth.

---

## 5. RLS / Multi-Tenancy (PostgreSQL GUC pattern)

If you use `SET LOCAL app.current_user` or similar GUC-based tenant isolation:

**Do:**
```python
# Always use SET LOCAL (transaction-scoped), never session-scoped
await conn.execute(
    "SELECT set_config('app.current_profile', $1, true)",  # true = transaction-local
    profile_name,
)
```

**Always wire `setup=` on pool creation:**
```python
pool = await asyncpg.create_pool(
    ...
    setup=_reset_rls_gucs,   # fires on EVERY checkout, before caller body
    init=_reset_rls_gucs,    # fires on new physical connection
)
```

The `setup=` callback is the load-bearing gate. It fires synchronously inside `pool.acquire()` before the caller runs, so a stale GUC from a prior request is always reset before the next request sees the connection.

**Never** use bare `pool.acquire()` on a code path that processes tenant data. Use a `ScopedConnection` wrapper that sets the GUC and auto-resets on exit. Enforce this in CI with a lint script.

---

## 6. Embedding / Vector Search

### Use an API, not an in-process model

Local embedding models (SentenceTransformer, etc.) on Cloud Run are CPU-bound. A single 2.4s embedding call blocks the event loop thread and creates a thundering herd risk under any concurrency.

**Use Google Vertex AI `text-embedding-004`:**
- ~5–15ms latency (same VPC, no external egress)
- $0.000025/1K chars — negligible cost at any scale
- Uses existing GCP service account IAM — no new API keys
- Singleton client, no cold-start after first call per instance

### Cache embeddings in the DB

```python
async def embed_text_cached(text: str, pool) -> list[float]:
    text_hash = hashlib.sha256(text.encode()).hexdigest()
    # Check cache first
    row = await conn.fetchrow(
        "SELECT embedding FROM embedding_cache WHERE text_hash=$1 AND model=$2",
        text_hash, "text-embedding-004",
    )
    if row:
        return row["embedding"]
    # Cache miss — call Vertex
    vector = await call_vertex(text)
    await conn.execute(
        "INSERT INTO embedding_cache (text_hash, model, embedding) VALUES ($1,$2,$3) ON CONFLICT DO NOTHING",
        text_hash, "text-embedding-004", vector,
    )
    return vector
```

Cache hit rate for a search-heavy tool is high — users search the same topics repeatedly.

### Embed at write time, not read time

Run embedding as a FastAPI background task when documents are written. Read paths then do a pure pgvector ANN query with no Vertex call.

```python
# In your write route
background_tasks.add_task(_bg_embed_document, doc_id, content)
return response  # Return immediately, embed happens after
```

**Cost reality check:**
- 10K users, 5 searches/day, 20% cache miss: **<$1/mo**
- 100K users: ~$10/mo
- Any prior estimate over $10/mo for embeddings at <100K users is wrong

---

## 7. Cost Model

Cloud Run + Cloud SQL MCP backends scale sub-linearly. Cost does not spiral.

| Registered users | Cloud Run | Cloud SQL | Vertex AI | Total/mo |
|-----------------|-----------|-----------|-----------|---------|
| 100             | ~$0       | $9        | <$1       | **~$10** |
| 1,000           | ~$0       | $9        | <$1       | **~$10** |
| 10,000          | ~$5       | $27 (g1-small) | <$1  | **~$35** |
| 100,000         | ~$50      | $90       | ~$10      | **~$155** |
| 1,000,000       | ~$500     | $400      | ~$100     | **~$1,000** |

**Fixed floor:** ~$10/mo regardless of user count. This is the dominant cost until ~5,000 users.

**Upgrade triggers:**
- Cloud SQL g1-small: when MAU > 250 sustained
- Cloud SQL HA: before first paying customer
- Read replica: when MAU > 5,000

---

## 8. Background Tasks

Background tasks (telemetry, session writes, contradiction checks) compete for the same DB pool as request handlers. Under pool pressure, they are the tipping point.

**Rules:**
1. Background tasks must use bounded connection budgets — wrap with a semaphore or dedicated small pool
2. Retry loops must have exponential backoff with a cap — unbounded retries during saturation amplify the problem
3. Background tasks that are non-critical (telemetry, metrics) should fail silently, not retry aggressively
4. Background tasks that are critical (session writes) should have a queue with a size cap — drop old entries, never block the pool indefinitely

```python
# Bounded background pool — telemetry, metrics, non-critical writes
_BG_POOL_SEM = asyncio.Semaphore(2)  # max 2 concurrent background DB ops

async def _bg_write(data):
    async with _BG_POOL_SEM:
        async with pool.acquire(timeout=2.0) as conn:
            await conn.execute(...)
```

---

## 9. Health Probes

Cloud Run health probes must actually test DB connectivity. An in-memory `/healthz` that always returns 200 means zombie instances (pool failed to initialize) keep receiving traffic and returning 500s.

```python
@app.get("/healthz")
async def healthz():
    try:
        pool = await get_pool()
        async with pool.acquire(timeout=2.0) as conn:
            await conn.fetchval("SELECT 1")
        return {"status": "ok"}
    except Exception as exc:
        raise HTTPException(status_code=503, detail=str(exc))
```

This ensures Cloud Run kills and replaces instances where the pool failed to initialize, rather than routing traffic to them.

---

## 10. Checklist Before Launch

Copy this into your project's CLAUDE.md or pre-deploy checklist:

```
[ ] min_size=1 in asyncpg create_pool
[ ] max_size × max-instances ≤ Cloud SQL max_connections − 5
[ ] max-instances identical across all deploy files (cloudbuild.yaml, *.sh, etc.)
[ ] concurrency in Cloud Run = max_size (or close to it)
[ ] auth token verification has in-memory cache (not DB on every call)
[ ] background tasks have connection semaphore (max 2 concurrent)
[ ] health probe actually acquires from DB pool
[ ] setup= and init= on asyncpg pool reset tenant GUC on every checkout
[ ] no bare pool.acquire() calls on tenant-data code paths
[ ] Vertex AI embeddings (not in-process model)
[ ] embed_text_cached checks DB cache before calling Vertex
[ ] write-time embedding as background task (not blocking read path)
[ ] Cloud SQL HA enabled (before first paying customer)
[ ] PITR enabled and tested
```

---

## 11. Parallel Tool Call Bursts (Claude Fan-Out)

Claude can send N tool calls in a single response batch. 7 parallel `log_decision` calls were observed in production. Each acquires a DB connection. Without controls this exhausts the pool.

**Two required fixes — both needed, different layers:**

### Per-session admission control (dispatcher layer)

Add a session-keyed semaphore in `mcp_http.py` that limits concurrent in-flight calls per `client_id`. This works for **every tool**, not just writes.

```python
# mcp_http.py
_SESSION_SEM_LIMIT = int(os.environ.get("MCP_SESSION_CONCURRENCY", "3"))
_session_sems: dict[str, asyncio.Semaphore] = {}
_session_sems_lock = asyncio.Lock()

async def _get_session_sem(client_id: str) -> asyncio.Semaphore:
    async with _session_sems_lock:
        if client_id not in _session_sems:
            _session_sems[client_id] = asyncio.Semaphore(_SESSION_SEM_LIMIT)
        return _session_sems[client_id]

async def _session_concurrency_middleware(request, call_next):
    token = get_access_token()
    if token is None:
        return await call_next(request)
    sem = await _get_session_sem(str(token.client_id))
    async with sem:
        return await call_next(request)
```

Effect: Claude sends 7 parallel calls → 3 execute immediately, 4 queue. No pool exhaustion. Claude still gets all 7 results.

**Why not in-process semaphore for background tasks:** `asyncio.Semaphore` is per-process on Cloud Run. 4 instances × Semaphore(2) = 8 concurrent background ops cluster-wide, not 2. For background tasks this is acceptable at current scale (4 × 3 connections + 4 idle = 16 vs 25 limit). For foreground admission control, the session semaphore is also per-process — but since all calls within one Claude session hit the same instance via Cloud Run's session affinity, it is effectively per-session.

### Batch write tool (`log_decisions`)

Replace single-item write tools with array-accepting equivalents. Claude sends one call with an array instead of N parallel calls.

**Rules:**
- `maxItems=25` enforced in **three places**: JSON Schema (tool definition), Pydantic model, middleware body-size cap. All three, or it's bypassable.
- Use `executemany` / multi-value INSERT — one DB round-trip regardless of N.
- Use SAVEPOINTs for per-item error handling — one duplicate item must not roll back the whole batch.
- Return 207 Multi-Status with per-item results `[{index, status, id}]`. Never 409 for a partial dup.
- Keep the old single-item tool as a thin wrapper calling the array endpoint. Don't delete it — existing Claude configurations reference it.

```python
# Per-item SAVEPOINT pattern
for i, item in enumerate(decisions):
    try:
        async with conn.transaction():  # SAVEPOINT
            decision_id = await _insert_one_decision(conn, item)
            results.append({"index": i, "status": "ok", "id": decision_id})
    except UniqueViolationError as e:
        existing_id = _extract_existing_id(e)
        results.append({"index": i, "status": "duplicate", "existing_id": existing_id})
```

**Quota enforcement:** check upfront (fast-fail for bad UX) + re-validate with `SELECT FOR UPDATE` inside the transaction to prevent TOCTOU. If quota allows 7 and batch is 10, write 7 and return per-item `quota_exceeded` for the last 3.

---

## 12. Cloud Run Instance Count Controls DB Connection Ceiling

`max-instances` is the real global cap. In-process semaphores are process-local and provide no cluster-wide guarantee.

```
total connections (worst case) = max-instances × max_size + max-instances × min_size
                               = 4 × 5 + 4 × 1 = 24  (within 25 for f1-micro)

rolling deploy worst case      = 8 × 5 + 8 × 1 = 48  (exceeds f1-micro)
```

**Rule:** The rolling deploy case is the binding constraint. With `min_size=1` and `max_size=5`, rolling deploy uses up to 48 connections — breaching f1-micro's 25 limit. This is acceptable at beta because:
1. Rolling deploys happen at low traffic (off-hours)
2. Old instances drain fast once the new revision is healthy
3. The realistic peak during a deploy is 8 × (1 idle + 1 active foreground) = 16, not 48

Upgrade to g1-small (100 connections) when you raise `max-instances` beyond 4 or hit 250+ MAU.

**Cloud Run scaling is driven by `--concurrency`, not application-level semaphores.** Application semaphores queue requests inside a process — Cloud Run still counts those HTTP connections toward the instance's concurrency limit and may spawn new instances. The only way to control instance spawning is `--max-instances` and `--concurrency`.

---

## 13. Write Path Design

### Collapse roundtrips — the MCP latency multiplier

Multiple DB roundtrips on a write tool are a classic N+1 problem. Every backend engineer has solved this. What makes it acute in MCP is the synchronous wait: the LLM sits idle, the user watches a spinner, and there is no skeleton screen or optimistic UI to mask the latency. A 5-roundtrip write that costs 200ms in a web app costs 800ms in MCP — and feels 3× worse because progress is invisible.

**Rule:** Every write tool must complete in a single transaction. Read-then-write patterns (SELECT to check existence, then INSERT) must be collapsed into `INSERT ... ON CONFLICT` or a CTE.

```sql
-- Bad: 3 roundtrips (check exists, insert, update counter)
SELECT id FROM decisions WHERE idempotency_key = $1;
INSERT INTO decisions (...) VALUES (...);
UPDATE profiles SET decision_count = decision_count + 1 WHERE id = $2;

-- Good: 1 roundtrip
WITH ins AS (
    INSERT INTO decisions (...) VALUES (...)
    ON CONFLICT (idempotency_key) DO UPDATE SET updated_at = now()
    RETURNING id, (xmax = 0) AS inserted
)
UPDATE profiles SET decision_count = decision_count + (SELECT inserted::int FROM ins)
WHERE id = $1;
```

### Idempotency keys on every write tool

LLMs retry on timeout differently than HTTP clients. An HTTP client retries a failed POST because it got a network error. An LLM retries a write tool because it timed out and isn't sure whether the write landed — then calls it again in the next turn. Without idempotency keys, you get duplicate records silently.

**Rule:** Every write tool must accept an optional `idempotency_key` parameter. If omitted, generate one from a hash of the stable input fields. Store it in the DB with a unique constraint. Return the existing record on conflict.

```python
idempotency_key = args.get("idempotency_key") or _stable_hash(args)
# INSERT ... ON CONFLICT (idempotency_key) DO UPDATE SET touched_at=now() RETURNING *
```

### Deduplication is not the same as idempotency

Idempotency says "same call twice = same result." Deduplication says "two semantically identical calls from different invocations = one record." Both are needed. LLMs paraphrase — `"we will use Postgres"` and `"decision: use Postgres as the database"` are the same decision logged twice. Add semantic similarity check at write time (pgvector cosine distance < 0.08) and return the existing record with a `deduplicated: true` flag rather than creating a new one.

---

## 14. Tool Design and Description Engineering

> **The LLM is your UI.** Tool descriptions are not documentation — they are routing contracts. The model reads them to decide what to call, in what order, with what arguments. A poorly written description is a broken navigation menu.

### Tool count affects model accuracy

The model must read all tool descriptions to select the right one. As tool count grows past ~20, selection accuracy degrades — especially when multiple tools have overlapping domains. This is not a documentation problem; it is a cognitive load problem on the model.

**Rule:** Keep public tool count ≤ 25. Consolidate via action dispatch rather than adding per-operation tools.

```python
# Bad: 4 tools that do variations of the same thing
close_thread(id)
reopen_thread(id)
escalate_thread(id)
snooze_thread(id, until)

# Good: 1 tool with an action parameter
thread_action(id, action: "close"|"reopen"|"escalate"|"snooze", until?)
```

This is MCP-native. There is no equivalent discipline in REST API or UI design — a nav menu with 40 items is bad UX but it doesn't cause the router to pick the wrong page.

### Description routing failures are silent

When the model picks the wrong tool, it does not error — it calls the tool and gets a result that looks plausible but is wrong or empty. There is no 404. The model may try to reason about the empty result rather than retrying with the right tool.

**Test your descriptions adversarially.** For every pair of tools with overlapping domains, write a prompt that should route to tool A and verify it does not route to tool B. Add these as regression tests in your test suite.

```python
# Example: search_memory vs search_decisions have overlapping domains
# A prompt asking "what did we decide about the database?" should route to search_memory
# (covers decisions + threads + sessions) NOT search_decisions (filters decisions only)
assert route_tool("what did we decide about the database?") == "search_memory"
```

### Error class discipline: protocol errors vs tool errors

The spec defines two error channels:
- **JSON-RPC protocol error** (e.g. `-32601 Method Not Found`): returned at the transport layer; the model treats this as a hard stop and does not retry with the same tool name
- **Tool execution error** (`isError: true` in result content): the model treats this as a soft failure and may retry with adjusted arguments

Unknown tool names must return `-32601` at the transport layer, not `isError: true` in content. An `isError: true` response to an unknown tool name causes the model to retry the unknown tool repeatedly, burning turns and confusing the session.

This is MCP-native. HTTP APIs have a well-understood 404 contract; LLM tool dispatch does not, and getting it wrong has no visible symptom until you inspect tool call traces.

---

## 15. Performance Traps

### What is common-sense web engineering vs what is MCP-specific

Many performance problems in MCP backends look familiar. They are not new. But the symptoms and stakes differ enough that teams who have solved them before still get burned:

| Problem | Common in web? | MCP-specific twist |
|---|---|---|
| N+1 / multiple DB roundtrips | Yes — textbook | Multiplied by sync wait; no UI to mask latency |
| Auth token cache miss | Yes — standard | Pool saturation → auth fails → complete tool unavailability, no graceful degradation |
| In-process CPU-bound work | Yes — offload it | Blocks the event loop; fast tools starved behind slow ones in shared executor |
| Cold start latency | Yes — keep-alive | First tool call in a session is the only first impression; no page-level retry |
| Connection pool exhaustion | Yes — size the pool | Idle floor is driven by instance count × min_size, not user count (see §2) |

### ThreadPoolExecutor isolation for slow tools

A single `ThreadPoolExecutor` shared across all tools means a slow tool (embedding call, LLM sub-call) occupies a worker for 2–5 seconds. If `max_workers=4` and 4 slow tools fire simultaneously, every fast tool (`get_decisions`, `list_threads`) queues behind them.

**Rule:** Separate executor for slow tools. Fast tools use the default asyncio thread pool.

```python
_SLOW_TOOL_EXECUTOR = ThreadPoolExecutor(max_workers=4, thread_name_prefix="mcp-slow")

SLOW_TOOLS = {"index_session_fragment", "wrap_session", "search_memory"}

async def _run_dispatch(name, args):
    handler = _DISPATCH[name]
    if name in SLOW_TOOLS:
        return await asyncio.get_event_loop().run_in_executor(_SLOW_TOOL_EXECUTOR, handler, args)
    return await asyncio.to_thread(handler, args)
```

### In-memory token cache: the single highest-ROI fix

Verifying an auth token against the DB on every tool call is the most common MCP latency spike. At low traffic it looks fine (50ms). Under pool pressure it cascades: the token verify takes a connection, the data query waits, the tool call times out, the LLM retries.

A 60-second in-memory TTL cache eliminates ~98% of DB hits for token verification. Instance-local is sufficient for beta; add cross-instance invalidation (Postgres `LISTEN/NOTIFY`) before paid customers.

```python
_TOKEN_CACHE: dict[str, tuple[str, float]] = {}  # token_hash → (profile_id, expires_at)
_TOKEN_TTL = 60.0

def _verify_token_cached(raw_token: str) -> str | None:
    h = hashlib.sha256(raw_token.encode()).hexdigest()
    entry = _TOKEN_CACHE.get(h)
    if entry and entry[1] > time.monotonic():
        return entry[0]
    profile_id = _verify_token_db(raw_token)  # one DB call
    if profile_id:
        _TOKEN_CACHE[h] = (profile_id, time.monotonic() + _TOKEN_TTL)
    return profile_id
```

This cut avg tool call latency from 12,291ms to <200ms in production. The root cause was a 5-second timeout on the loopback HTTP call that verified the token — every tool call paid that cost.

### Latency gate: automate or it drifts

Latency regressions in MCP are invisible until a user complains. Add a latency gate to CI that:
- Calls all public tools with valid auth
- Asserts avg < 500ms, p95 < 1000ms
- Exits 1 on failure, blocking deploy

Run it before every commit touching the auth or DB path. The gate pays for itself the first time it catches a regression before it ships.

---

## 16. Security Wiring Gaps

These are not novel security concepts. Every item below has a well-known mitigation. What is MCP-specific is how easy they are to wire incorrectly and how the standard test suite does not catch the wiring gap.

### RLS GUC not reset on connection release

If you use PostgreSQL GUCs for tenant isolation (`SET LOCAL app.current_profile`), the GUC must be reset when the connection returns to the pool. A bare `pool.acquire()` with `SET LOCAL` in the transaction is not sufficient — if the transaction is committed without a RESET, the next borrower gets the previous tenant's GUC.

**The wiring gap:** unit tests mock the DB call. The GUC state is never set in tests, so the bug is invisible. Integration tests using the real pool catch it; unit tests never do.

```python
async def _reset_rls_gucs(conn):
    await conn.execute("SELECT set_config('app.current_profile', '', false)")

pool = await asyncpg.create_pool(
    ...,
    setup=_reset_rls_gucs,   # fires on every checkout — this is the load-bearing line
    init=_reset_rls_gucs,    # fires on new physical connection
)
```

`setup=` is not the same as `init=`. `init=` fires once when the physical connection is created. `setup=` fires on every logical checkout from the pool. Both are required.

### PII scrubber scope: wiring gap, not a scrubber gap

A PII scrubber that is wired to one tool but not others gives a false sense of compliance. The scrubber itself is correct. The gap is that write tools which store free-text (decision body, session notes, thread comments) each need to be explicitly wired — there is no framework-level guarantee.

**Rule:** Treat PII scrubbing as a middleware concern at the dispatch layer, not a per-tool responsibility. Wire it once in `_run_dispatch` before any handler is called, for all tools that accept free-text arguments.

```python
_FREE_TEXT_ARGS = {"body", "content", "summary", "notes", "description", "rationale"}

def _scrub_free_text(args: dict) -> dict:
    return {k: (_safe_scrub(v) if k in _FREE_TEXT_ARGS and isinstance(v, str) else v)
            for k, v in args.items()}

# In dispatch, before handler:
args = _scrub_free_text(args)
```

### Two-stage commit for destructive tools

There is no confirmation dialog in an LLM tool call. The model will call `purge_records` or `delete_project` the moment it decides to, with no opportunity for the user to see a preview and cancel.

The two-stage commit pattern replaces the dialog:
1. Call without `confirmed=true` → returns a human-readable preview of what will be deleted, a record count, and the IDs affected
2. User reads the preview in the conversation
3. User says "yes, do it" → model calls again with `confirmed=true`

This is the only pattern that provides a meaningful confirmation gate without requiring a UI. It is not in the spec. Build it into every destructive tool.

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| `min_size=2` on small Cloud SQL | 2× idle connection floor; deploy spikes hit ceiling | `min_size=1` |
| `concurrency=1000` on Cloud Run | Never scales out on concurrency; CPU spikes cause surprise scale-out | `concurrency=8` |
| Multiple deploy files with different Cloud Run params | Silent config drift; last deploy wins | Single canonical source |
| In-process embedding model (SentenceTransformer) | CPU-bound, 2–12s latency, thundering herd | Vertex AI API |
| Embedding on read path (not cached) | Vertex called on every search; latency + cost | embed_text_cached + write-time embed |
| Auth token refresh on shared data pool | Pool saturation → auth fails → 401 cascade | In-memory token cache |
| Bare `pool.acquire()` on tenant code paths | RLS GUC may not be reset; potential cross-tenant leak | ScopedConnection everywhere |
| PgBouncer for low-RPS MCP traffic | Adds operational complexity with no benefit | Only needed at ~30K MAU |
| Running panels with web-app concurrency assumption | 10–100× overestimate of connection pressure | Always state MCP traffic model first |
| In-process semaphore as cluster-wide rate limit | asyncio.Semaphore is per-process; 4 instances × Semaphore(2) = 8 concurrent, not 2 | Use per-session admission at dispatcher; use `max-instances` as the real ceiling |
| Single-item write tool called N times in parallel | N DB connections acquired simultaneously; 2 block on pool.acquire(15s) | Array-accepting tool + per-session admission control |
| Atomic transaction for independent batch items | One dup in 10-item batch → 409 → entire batch lost → retry amplifies the dup | SAVEPOINTs + 207 per-item result |
| No `maxItems` cap on array tools | Array of 500 → CPU spike on injection scan + 5s connection hold | `maxItems=25` in JSON schema + Pydantic + middleware body cap |
