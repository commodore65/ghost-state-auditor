# Ghost State Auditor

![ghost-state-auditor](assets/banner.png)


An AI coding agent skill that finds **ghost state** — data that exists in one layer (memory, DB, cache, API) but is missing, stale, or desynchronized in another.

These bugs pass all unit tests because each layer works correctly in isolation. The inconsistency only surfaces across boundaries: process restarts, daily resets, layer transitions, or multi-step workflows.

## The Problem

Unit tests verify that code is correct. But they can't catch **architectural gaps** where:

- A value is computed and used at runtime but **never written to the database** (Phantom State)
- Two storage layers should stay in sync but **one updates without the other** (Desynchronized State)
- An accumulator is periodically reset **without flushing to durable storage first** (Silent Reset)

These bugs are invisible during normal operation. They only break when the process restarts, a daily counter resets, or a user checks a lifetime metric that was actually reading from a session-scoped source.

## Install

```bash
npx skills add commodore65/ghost-state-auditor
```

Or manually copy into your project:

```bash
# Project-level (this repo only)
cp -r .claude/skills/ghost-state-auditor /path/to/project/.claude/skills/

# User-level (all your projects)
cp -r .claude/skills/ghost-state-auditor ~/.claude/skills/
```

## Usage

In Claude Code, use any of these triggers:

```
/ghost-audit
```

Or describe what you need:

```
"run a state audit on this project"
"check for persistence bugs"
"find ghost state in the codebase"
```

## What It Finds

### Class 1: Phantom State

Data computed at runtime but never persisted. Works perfectly until the process restarts.

```
Example: An order service calculates total revenue on each order completion,
stores it in a memory dict. The DB has an orders table with a revenue column
— but no INSERT is ever executed. Lifetime revenue resets to $0 on every
restart. All tests pass.
```

### Class 2: Desynchronized Coupled State

Two values that must maintain a relationship across storage layers, but one gets updated while the other doesn't.

```
Example: A daily summary function reads daily_counter and displays it as
"Lifetime Total" in the UI. The counter resets at midnight. Users see
"Total: $5.00" when the real lifetime total is $500.00.
```

### Class 3: Silent Reset

State that is periodically zeroed without first transferring the accumulated value to durable storage.

```
Example: An error counter increments throughout the day, resets at midnight
for the daily limit check. But no code persists the daily total before reset.
Historical error rates are lost.
```

## Audit Process

The skill runs through 8 systematic phases:

| Phase | What It Does |
|-------|-------------|
| 1. State Location Map | Maps every piece of state across memory, DB, file, cache, external API |
| 2. Persistence Gaps | Finds memory-only state that should be in the DB |
| 3. Coupled State Desync | Finds values in multiple locations that aren't synchronized |
| 4. Silent Resets | Finds accumulators zeroed without flushing |
| 5. Restart Simulation | Mentally walks through process restart scenarios |
| 6. Transaction Atomicity | Checks multi-write operations for partial failure handling |
| 7. Display Source Check | Verifies UI/notifications read from the authoritative source |
| 8. Verification Gate | Eliminates false positives with code trace + grep |

Built-in scoping rules prevent the audit from becoming too broad on large codebases — it prioritizes persistence layer, then stateful entrypoints, then high-risk flows.

Includes explicit exemptions for eventual consistency patterns (outbox, CDC, CQRS, saga, event-driven sync) to avoid false positives in distributed systems.

## Output

Findings are presented directly in the conversation. If you want a file, ask for it and it will be saved to `docs/ghost-state-audit.md`.

Each finding includes:
- Severity and ghost state class
- Exact file and line number
- What breaks and how to trigger it
- Minimal fix

## Severity Levels

| Severity | Criteria |
|----------|----------|
| CRITICAL | Business-critical data lost on restart. State desync causes wrong decisions or data corruption. |
| HIGH | Operational data lost. Display shows wrong values to user. |
| MEDIUM | Non-critical state lost. Requires manual recovery. |
| LOW | Cosmetic state lost. No operational impact. |

## Works With Any Stateful Application

This skill is **domain-agnostic**. It audits any application that maintains state across layers — not limited to any specific industry or tech stack.

| Application Type | Example Ghost State |
|-----------------|-------------------|
| SaaS / Web apps | User quota tracked in memory, billing reads from DB |
| Background workers | Job progress in memory, dashboard reads stale DB |
| E-commerce | Cart totals in session, never synced to persistent store |
| Messaging services | Unread count in cache, resets on deploy |
| IoT / Monitoring | Sensor aggregations in memory, lost on restart |
| CI/CD pipelines | Build stats ephemeral, no trend analysis possible |
| API services | Rate limit counter in memory, bypassed after restart |

**The only prerequisite is that the application has state.** Pure stateless functions, CLI tools, or static sites won't benefit from this audit.

## Compatibility

| Agent | Support |
|-------|---------|
| **Claude Code** | Native — invoke via `/ghost-audit` |
| **Cursor, Cline, Windsurf, Copilot** | Install via `npx skills add`, reads from `.claude/skills/` |
| **Any other LLM agent** | Copy the SKILL.md content into your system prompt — the methodology is plain markdown |

## License

MIT
