---
name: ghost-state-auditor
description: "Finds ghost state bugs: data computed but never persisted, state coupled across layers (memory/DB/API) but not synchronized, and values that silently reset or drift on process restart. Triggers on /ghost-audit or 'state audit' or 'persistence audit'."
---

# Ghost State Auditor

Finds **ghost state** — data that exists in one layer (memory, DB, cache, API response) but is missing, stale, or desynchronized in another. These bugs pass all unit tests because each layer works correctly in isolation. The inconsistency only surfaces across boundaries: process restarts, daily resets, layer transitions, or multi-step workflows.

This is the bug class that testing alone cannot catch: the code is correct, but the **architecture** has a gap.

## When to Activate

- User says `/ghost-audit` or `state audit` or `persistence audit`
- After a bug where "data was lost" or "values keep resetting"
- When reviewing a long-running daemon, bot, or service
- Before deploying a new feature that adds state tracking
- Periodic health check on any stateful system

## When NOT to Use

- Pure stateless APIs or CLI tools
- Simple CRUD apps where ORM handles all persistence
- Performance-only audits (use a profiler instead)

---

## The Three Ghost State Classes

### Class 1: Phantom State (Computed, Never Persisted)

Data is calculated and used at runtime but never written to durable storage. It works perfectly... until the process restarts.

```
PATTERN: value = compute(inputs)   # exists in memory
         # ... used throughout runtime ...
         # NEVER written to DB/file/cache
         # Process restarts -> value = 0 (ghost)
```

**Example: Order processing service**
- `total_revenue` calculated on each order completion, stored in a memory dict
- `get_lifetime_revenue()` sums from that dict — works perfectly during runtime
- DB has an `orders` table with a `revenue` column (schema exists)
- But no INSERT is ever executed — lifetime revenue resets to $0 on every restart
- All unit tests pass because they run within a single process lifetime

### Class 2: Desynchronized Coupled State

Two or more values must maintain a relationship (invariant) across different storage layers or code paths. One gets updated, the other doesn't.

```
PATTERN: layer_A.value is updated by operation X
         layer_B.value SHOULD also be updated
         But layer_B.value is stale/missing/wrong
```

**Examples:**
- Memory counter vs DB counter (diverge after partial failures)
- Cache vs source-of-truth (cache never invalidated)
- Daily summary reads `daily_counter` but labels it "Lifetime Total" in the UI
- Config in memory vs config on disk (hot-reload race)
- User's subscription quota tracked in memory but billing reads from DB

### Class 3: Silent Reset State

State that is periodically zeroed, overwritten, or replaced — erasing accumulated history without transferring it to a durable aggregate.

```
PATTERN: accumulator += new_value    # grows over time
         # ... at some trigger (midnight, restart, overflow) ...
         accumulator = 0              # reset without persisting
```

**Examples:**
- Daily revenue counter resets at midnight but lifetime total was derived from it
- Error counter resets on restart, no historical error rate available
- Session-scoped metrics never flushed to persistent storage
- Notification badge count cleared on deploy, unread items still exist in DB

---

## Core Rules

```
RULE 0: MAP LAYERS BEFORE YOU HUNT
Never start checking code until you have a complete map of where state
lives: memory (dataclass, dict, variable), database (table, column),
file (config, log), cache (Redis, in-process), API response.

RULE 1: EVERY WRITE NEEDS A PERSISTENCE COUNTERPART
If state is written to memory AND a DB table exists for it, verify
there is actual INSERT/UPDATE code connecting them. Schema alone is not
persistence — executing the query is.

RULE 2: EVERY RESET NEEDS A FLUSH
If any accumulator or counter is periodically reset (daily, on restart,
on overflow), verify the accumulated value is persisted BEFORE the reset.

RULE 3: EVERY DISPLAY NEEDS A TRUE SOURCE
If a value is shown to the user (UI, dashboard, API, notification, log), trace
it back to its source. Is the source the right one? Is it the lifetime
value or just the current session/day?

RULE 4: RESTART IS THE ULTIMATE TEST
Mentally restart the process at any point. What state survives? What
doesn't? Everything that doesn't survive must either be: (a) intentionally
ephemeral, or (b) a ghost state bug.

RULE 5: EVIDENCE-BASED FINDINGS ONLY
Every finding must show: the specific state, where it's written, where
it's NOT written/persisted, and what breaks as a result.
```

---

## Scoping Rules

For large codebases, audit in priority order — do NOT attempt to map the entire codebase at once:

```
PRIORITY 1: Persistence layer
  - Database schemas, ORM models, migration files
  - All INSERT/UPDATE/DELETE statements
  - Which tables are written to vs. schema-only

PRIORITY 2: Stateful entrypoints
  - Long-running processes (daemons, workers, consumers)
  - Classes with __init__ that initialize counters/accumulators
  - Singletons and module-level state

PRIORITY 3: High-risk flows
  - Operations involving money, quotas, or user-facing metrics
  - Multi-step operations (two or more writes in sequence)
  - Scheduled tasks that reset or aggregate state

PRIORITY 4: Supporting state (only if time permits)
  - Caches, session stores, temp files
  - Config hot-reload mechanisms
  - Background job state
```

If the codebase is large (>50 files with state), explicitly state which scope you are auditing and note what was excluded.

---

## Eventual Consistency Exemptions

Some architectures intentionally desynchronize state across layers. Do NOT flag these as bugs:

```
EXEMPT PATTERNS:
- Outbox pattern: DB write + outbox row, async worker publishes events later
- Event-driven sync: State changes propagated via message queue (Kafka, RabbitMQ, SQS)
- Materialized views: DB view refreshed on schedule, intentionally stale
- Cache-aside: Cache miss triggers DB read, TTL-based staleness is by design
- CDC (Change Data Capture): Debezium/similar streams DB changes to consumers
- CQRS: Write model and read model intentionally diverge, synced eventually
- Saga pattern: Multi-service transactions with compensating actions

WHEN YOU SEE THESE:
1. Verify the async sync mechanism actually exists and is wired up
2. Check that failure/retry handling exists for the async path
3. If sync mechanism is missing or broken -> FINDING (not the pattern itself,
   but the incomplete implementation)
4. Document as "eventual consistency by design" in the State Location Map
```

---

## Audit Process

### Phase 1: Map All State Locations

Following the scoping rules above, for each piece of state in scope, record WHERE it lives:

**1a. Memory State** — Find all:
- Class/instance attributes (`self.x = ...`, dataclass fields)
- Module-level variables and singletons
- Dict/list accumulators (`self.positions: dict = {}`)
- Counters (`self.request_count`, `self.processed_count`)

**1b. Database State** — Find all:
- Table schemas (CREATE TABLE, ORM models)
- INSERT/UPDATE/UPSERT statements
- Which tables are actually written to vs. only schema-defined

**1c. File State** — Find all:
- Config files read at startup (YAML, JSON, .env)
- Log files, CSV exports, state files
- Candidate/temp files written during operation

**1d. Cache State** — Find all:
- Redis/Memcached keys
- In-process caches (`@lru_cache`, dict caches)
- Memoized computations

**1e. External API State** — Find all:
- Remote service state (payment provider balances, inventory systems, third-party quotas)
- Third-party service state that mirrors local state

**Output:** A State Location Map.

```
+------------------+------------------+------------------+------------------+
| State            | Memory           | Database         | External         |
+------------------+------------------+------------------+------------------+
| orders           | self.orders{}    | orders table     | payment API      |
| lifetime_revenue | get_total()      | SUM(orders.amt)  | -                |
| daily_revenue    | counter.daily    | daily_stats tbl  | -                |
| request_count    | self.req_count   | COUNT(requests)  | -                |
| user_quotas      | quota_cache{}    | quotas table     | billing API      |
| config           | self.config      | -                | config/*.yaml    |
+------------------+------------------+------------------+------------------+
```

---

### Phase 2: Find Persistence Gaps (Class 1 — Phantom State)

For EACH state in the Memory column from Phase 1:

```
CHECK 1: Does a corresponding DB table/column exist?
  YES -> CHECK 2
  NO  -> Is the state intentionally ephemeral?
         YES -> OK (document as "ephemeral by design")
         NO  -> FINDING: Phantom State (no persistence layer)

CHECK 2: Is there actual INSERT/UPDATE code connecting memory to DB?
  YES -> CHECK 3
  NO  -> FINDING: Phantom State (schema exists, no write code)
         This is the classic "schema without writes" pattern.

CHECK 3: Is the write code reachable in all relevant code paths?
  YES -> OK
  NO  -> FINDING: Partial persistence (some paths persist, others don't)
```

**Red flags to grep for:**
- DB table defined but no INSERT for that table name
- `self.state.x += value` without a nearby `conn.execute("UPDATE/INSERT...")`
- Dataclass with `total_amount: float = 0.0` but no serialization to DB
- `dict[str, X]` used as primary state store (dicts don't survive restart)

---

### Phase 3: Find Coupled State Desync (Class 2)

For EACH state that exists in multiple locations (multiple columns in Phase 1 map):

```
CHECK 1: When memory state changes, is DB state updated in the same operation?
  NO -> FINDING: Write-through gap

CHECK 2: When DB state changes (migration, manual edit, external tool),
         is memory state refreshed?
  NO -> FINDING: Read-through gap (may be acceptable for startup-only reads)

CHECK 3: When the display/report reads state, does it read from the
         authoritative source?
  NO -> FINDING: Wrong source (e.g., reading daily_count but displaying as "Lifetime Total")

CHECK 4: Are there two different code paths that should maintain
         the same coupled state but handle it differently?
  YES -> FINDING: Parallel path desync
```

**Parallel path analysis** (adapted from state-inconsistency-auditor):

For operations that achieve similar outcomes through different paths:
- `complete_order()` vs `cancel_order()` — both finalize an order
- Normal flow vs admin override — same state changes needed?
- User-initiated vs automated/cron — same persistence path?
- Single operation vs batch — all state updated per item?

Build a comparison table:

```
+------------------+-----------------+------------------+------------------+
| Coupled State    | complete_order  | cancel_order     | admin_override   |
+------------------+-----------------+------------------+------------------+
| memory updated   | YES             | YES              | YES              |
| DB record saved  | YES             | ???              | ???              |
| metrics updated  | YES             | YES              | ???              |
| notification     | YES             | YES              | ???              |
+------------------+-----------------+------------------+------------------+
```

`???` entries are your primary audit targets.

---

### Phase 4: Find Silent Resets (Class 3)

For EVERY reset, clear, or overwrite operation:

```
SCAN FOR:
- `= 0` or `= 0.0` on any counter/accumulator
- `= {}` or `.clear()` on any dict/list
- `reset_daily()`, `reset()`, `__init__()` re-initialization
- Cron/scheduled tasks that zero out values
- Process startup that reinitializes state from defaults

FOR EACH RESET FOUND:
  CHECK: Is the accumulated value persisted BEFORE the reset?
    YES -> OK
    NO  -> FINDING: Silent reset erases unpersisted state

  CHECK: Does any downstream consumer depend on the lifetime
         (not just current-period) value?
    YES -> Is it reading from the persisted source or the resettable one?
           Resettable -> FINDING: Consumer reads ephemeral source for lifetime data
```

---

### Phase 5: Restart Simulation

Mentally walk through a process restart at these critical moments:

1. **Mid-operation restart:** What happens if the process dies between step 1 and step 2 of a multi-step operation? Between DB write and notification send?

2. **Clean restart:** What state is lost? What state is correctly reloaded from DB? What state starts from defaults and shouldn't?

3. **Post-daily-reset restart:** If the process restarts right after midnight reset, is any data from yesterday lost that should have been summarized?

For each scenario, trace:
```
State X:
  Before restart: value = [what]
  After restart:  value = [what]
  Expected:       value = [what]
  Match? YES/NO -> if NO, FINDING
```

---

### Phase 6: Transaction & Atomicity Check

For EVERY operation that modifies multiple pieces of state:

```
CHECK 1: Are all related writes in the same transaction/atomic block?
  NO -> What happens if the process fails between write 1 and write 2?

CHECK 2: If the operation involves both memory and DB writes,
         what is the order?
  Memory first, then DB -> On DB failure, memory is ahead of DB
  DB first, then memory -> On memory update failure, DB is ahead
  -> Is there rollback/recovery for partial completion?

CHECK 3: For async operations with gather/concurrent writes,
         what happens if one succeeds and the other fails?
  -> Is there compensating logic?
```

---

### Phase 7: Display Source Verification

For EVERY user-facing display (notification, dashboard, API endpoint, log line, email report):

```
TRACE the value back to its source:
1. What variable is displayed?
2. Where does that variable come from?
3. Is that the authoritative source or a derived/cached/ephemeral copy?
4. Does the label match what the value actually represents?
   (e.g., "Lifetime Total" label but actually showing today's daily_count value)
```

---

### Phase 8: Verification Gate (MANDATORY)

Every CRITICAL, HIGH, and MEDIUM finding MUST be verified:

**Method A: Code Trace**
1. Read the exact code that should persist/sync the state
2. Trace all call chains — confirm no hidden persistence exists
3. Check decorators, hooks, event handlers for implicit saves
4. Verdict: TRUE POSITIVE / FALSE POSITIVE

**Method B: Grep Verification**
1. Grep for INSERT/UPDATE with the table name
2. Grep for the state variable name near any DB write
3. If zero results -> confirmed no persistence
4. If results found -> read the code to verify it's actually executed

**Common False Positive Patterns:**
- **Lazy persistence:** State IS saved, but in a periodic flush job (check cron/scheduled tasks)
- **ORM auto-commit:** Some ORMs auto-persist on session flush — check session config
- **Event-driven save:** A listener/hook saves state on change events
- **Eventual consistency:** Outbox, CDC, CQRS, or saga patterns intentionally delay sync (see Exemptions section)
- **Intentional ephemeral:** Some state (like request count) genuinely doesn't need persistence

---

## Severity Classification

| Severity | Criteria |
|----------|----------|
| **CRITICAL** | Business-critical data lost on restart (revenue, balances, user records). State desync causes wrong decisions or data corruption. |
| **HIGH** | Operational data lost (metrics, counters) that affects monitoring/alerting. Display shows wrong values to user. |
| **MEDIUM** | Non-critical state lost (history, statistics). Requires manual recovery after restart. |
| **LOW** | Cosmetic state lost (uptime counter, session stats). No operational impact. |

---

## Output Format

Present findings directly in the conversation. If the user requests a file, save to `docs/ghost-state-audit.md` (create the directory if needed).

```markdown
# Ghost State Audit Report - [DATE]

## State Location Map
[Table from Phase 1]

## Persistence Gap Matrix
[Which memory states have DB persistence and which don't]

## Parallel Path Comparison
[Table from Phase 3 if applicable]

## Restart Impact Analysis
[Table from Phase 5]

## Findings

### GS-001: [Title]
**Severity:** CRITICAL | HIGH | MEDIUM | LOW
**Class:** Phantom State | Desynchronized State | Silent Reset
**Verification:** Code trace / Grep

**State:** [what state is affected]
**Memory location:** `file.py:line` — `self.variable`
**Expected persistence:** `table.column` (or "none exists")
**Actual persistence:** [none / partial / wrong source]

**What breaks:**
- [Concrete consequence: "Lifetime revenue resets to $0.00 on service restart"]

**Trigger:**
1. [Step-by-step sequence to expose the ghost state]

**Fix:**
```[language]
[Minimal code fix]
```

---

## Summary
- State locations mapped: [N]
- Persistence gaps found: [N]
- Coupled state desyncs found: [N]
- Silent resets found: [N]
- After verification: [N] TRUE POSITIVE | [N] FALSE POSITIVE
- Final: [N] CRITICAL | [N] HIGH | [N] MEDIUM | [N] LOW
```

---

## Anti-Hallucination Protocol

```
NEVER:
- Assume a DB table is written to without finding the actual INSERT/UPDATE code
- Claim state is "lost on restart" without verifying it's not loaded from DB at startup
- Report a finding without showing the exact code location
- Assume a reset is a bug without checking if the value is persisted first
- Ignore ORM auto-flush, event handlers, or periodic save jobs

ALWAYS:
- Grep for actual SQL/ORM write operations, not just schema definitions
- Trace the full lifecycle: create -> update -> persist -> restore -> display
- Verify by checking both the write path AND the read path
- Show exact file paths and line numbers
- Distinguish between "no persistence layer exists" vs "persistence exists but is broken"
```

---

## Quick-Start Checklist

- [ ] Phase 1: Map all state across memory, DB, file, cache, external
- [ ] Phase 2: Find phantom state (memory-only, no persistence code)
- [ ] Phase 3: Find coupled state desync (multiple locations, not synchronized)
- [ ] Phase 4: Find silent resets (accumulators zeroed without flushing)
- [ ] Phase 5: Simulate restart at critical moments
- [ ] Phase 6: Check transaction atomicity for multi-write operations
- [ ] Phase 7: Verify display sources match authoritative data
- [ ] Phase 8: Verify all C/H/M findings with code trace + grep
- [ ] Present findings (save to `docs/ghost-state-audit.md` if requested)
