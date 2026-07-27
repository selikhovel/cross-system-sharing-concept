# ADR-0003 — Outbox/Inbox storage, claiming and delivery semantics

- **Status:** Proposed
- **Date:** 2026-07-25
- **Related:** ADR-0001, ADR-0002, ADR-0004, ADR-0006

## Context

ADR-0001 chose an outbox/inbox architecture; ADR-0002 chose two-stage capture. This ADR
fixes the mechanics that determine whether it actually works in production:

- Where the tables live and how they relate to the business schema.
- How concurrent workers claim work without double-delivering or serialising.
- What delivery and ordering guarantees we offer.
- How the pull feed avoids the **lost-update-on-cursor** bug (the subtle one).
- Retry, dead-lettering, and retention.

## Decision

### 1. Storage: same database, dedicated `integration` schema

Outbox and inbox tables live in the **same PostgreSQL database** as the domain tables, in a
separate schema named `integration`, owned by a separate EF Core `DbContext`
(`IntegrationDbContext`) that shares the domain context's connection and transaction.

Same database is non-negotiable — atomicity with the business transaction is the entire
point of the pattern. A separate schema keeps the boundary visible, allows separate
migration history, distinct grants, and independent partition/retention policy.

```
acme (database)
├── public / core      ← domain tables, unchanged
└── integration         ← outbox_change_log, outbox_message, outbox_delivery,
                          inbox_message, dead_letter, subscriber, consumer_cursor,
                          backfill_run
```

Both contexts must enlist in one transaction. Because `AddDbContextPool` gives each context
its own connection by default, the unit of work explicitly shares one:

```csharp
await using var tx = await _domainDb.Database.BeginTransactionAsync(ct);
_integrationDb.Database.SetDbConnection(_domainDb.Database.GetDbConnection());
await _integrationDb.Database.UseTransactionAsync(tx.GetDbTransaction(), ct);
```

> Simpler alternative, recommended for phase 1: put the `integration` entities in the
> **existing** domain `DbContext` with `.ToTable("outbox_message", "integration")`. One context,
> one transaction, zero plumbing. Split into a second context only if the domain context's
> model becomes unwieldy. **We start with one context.**

#### In a modular monolith

There is no single `AppDbContext` to fall back on, so the sentence above needs a topology to attach
to. The decision is **one shared outbox for the whole monolith**, in the same `integration` schema,
written by the `DbContext` of whichever module produced the change.

The deciding argument is the consumer's cursor. The outbox's value is that it is *one* ordered log
with *one* cursor and *one* retention window. Splitting it per module pushes N cursors, N windows and
N catch-up procedures onto every consumer, and turns "resume after a day offline" from one operation
into N independent ones. Module autonomy is worth a lot; it is not worth that.

| Aspect | Consequence |
|---|---|
| Migration ownership | The `integration` schema is owned by a shared building block, not by a module. Give it its own migrations history table (`__EFMigrationsHistory` in the `integration` schema) so ownership can move without rewriting history |
| Module contexts | Every publishing module's context maps the integration entities. The write must land in the same transaction as the business row, which is the entire point |
| Cost accepted | Two or more contexts map tables they do not own. This is real coupling between modules, and it is the price of a single consumer-facing log |

**With one module, this costs nothing today** — the recommendation above applies verbatim, the
module's own context maps the tables, and the shared building block is extracted when a second module
actually needs it. Extracting early buys an abstraction with no second consumer.

### 2. Claiming: `FOR UPDATE … SKIP LOCKED`, no leader election

Workers claim work with the standard PostgreSQL queue idiom:

```sql
WITH claimed AS (
    SELECT id
    FROM integration.outbox_delivery
    WHERE subscriber_id   = @subscriberId
      AND status          = 'Pending'
      AND next_attempt_at <= now()
    ORDER BY next_attempt_at, id
    LIMIT @batchSize
    FOR UPDATE SKIP LOCKED
)
UPDATE integration.outbox_delivery d
   SET status        = 'InFlight',
       attempt       = d.attempt + 1,
       locked_by     = @workerId,
       locked_until  = now() + interval '2 minutes'
  FROM claimed c
 WHERE d.id = c.id
RETURNING d.*;
```

This gives horizontal scale-out with **no leader election, no distributed lock, no
Redis, no advisory-lock bookkeeping** — any number of pod replicas can run the dispatcher
concurrently and never claim the same row. `locked_until` is a crash-recovery reaper, not the
primary safety mechanism (the row lock is), which is why a worker dying mid-flight is safe:
the transaction aborts and the row returns to `Pending` immediately.

Backed by a partial index sized for exactly this query:

```sql
CREATE INDEX ix_outbox_delivery_claim
    ON integration.outbox_delivery (subscriber_id, next_attempt_at, id)
    WHERE status = 'Pending';
```

**Polling interval:** 1 s when the last batch was empty, immediately when it was full
(adaptive drain). `LISTEN`/`NOTIFY` was considered to cut idle latency; rejected for
phase 1 because notifications are lost if no session is listening, so polling is required
as the backstop anyway — adding NOTIFY buys a few hundred milliseconds for a second failure
mode to reason about. Revisit only if p95 latency requirements tighten below ~2 s.

### 3. Delivery guarantee: **at-least-once**, with version-based idempotency

We do not promise exactly-once — it is not achievable across an HTTP boundary we do not
control. We promise:

- **At-least-once delivery** of the latest state of every changed aggregate.
- A **stable `messageId`** (UUIDv7) per message, unchanged across retries, sent both in the
  envelope and as the `Idempotency-Key` HTTP header.
- A **monotonically increasing `aggregateVersion`** per aggregate.

Consumers are contractually required to do one of:
`if (incoming.aggregateVersion <= stored.aggregateVersion) ignore;`
or store seen `messageId`s in their own inbox. This requirement is stated in the
consumer-facing contract document, not left implicit.

### 4. Ordering: per-aggregate only, achieved by compaction rather than sequencing

Global ordering across aggregates is **not** offered — it has no business meaning and would
force single-threaded delivery.

Per-aggregate ordering is where naive outbox implementations break: with `SKIP LOCKED` and
N workers, two pending messages for the same `Foo` can be delivered out of order, and
a retry of message #1 can land *after* message #2. Rather than build stream locks, we exploit
the state-transfer decision from ADR-0002:

**Compaction (supersession).** When the materializer produces a new message for an aggregate,
it supersedes any still-`Pending` delivery for that aggregate and subscriber:

```sql
UPDATE integration.outbox_delivery
   SET status = 'Superseded', completed_at = now()
 WHERE subscriber_id = @subscriberId
   AND aggregate_key = @aggregateKey
   AND status        = 'Pending'
   AND sequence      < @newSequence;
```

Combined with the consumer-side `aggregateVersion` check, this makes out-of-order delivery
**harmless by construction**: the consumer converges on the highest version it has seen,
regardless of arrival order. It also collapses edit storms into a single HTTP call.

Compaction is **disabled** for `ChangeKind = Event` messages (ADR-0002 §3), which carry
transition semantics and must all be delivered. Those are marked `compactable = false` and,
for the aggregates that use them, the dispatcher claims per-aggregate with a stream lock —
scoped to the handful of event types that need it, not the whole system.

### 5. Pull feed correctness: `xid8` visibility watermark

The subtle bug in every cursor-paginated change feed: `sequence` is assigned by
`GENERATED ALWAYS AS IDENTITY` **at insert time**, but rows become visible **at commit
time**. Transaction A takes sequence 100 and commits slowly; transaction B takes 101 and
commits fast. A consumer reading now sees 101, advances its cursor to 101, and **never sees
100.** Silent, permanent data loss, invisible in testing and common under load.

Fix: every `outbox_message` row records the inserting transaction id, and the feed only
exposes rows whose transaction is guaranteed complete:

```sql
ALTER TABLE integration.outbox_message
    ADD COLUMN insert_xid xid8 NOT NULL DEFAULT pg_current_xact_id();

-- feed query: only rows below the oldest currently-running transaction
SELECT *
  FROM integration.outbox_message
 WHERE sequence    > @cursor
   AND insert_xid  < pg_snapshot_xmin(pg_current_snapshot())
 ORDER BY sequence
 LIMIT @limit;
```

`pg_snapshot_xmin(pg_current_snapshot())` is the oldest still-in-progress transaction id.
Anything below it has committed or aborted — so no row below the watermark can appear later.
This costs a small, bounded feed lag (the duration of the longest concurrent write
transaction) in exchange for a feed that cannot skip rows. Requires PostgreSQL 13+.

The same watermark is applied to the dispatcher's materialisation scan for the same reason.

### 6. Retry, backoff and dead-letter

| Attempt | Delay | Notes |
|---|---|---|
| 1 | immediate | |
| 2 | 5 s | |
| 3 | 30 s | |
| 4 | 2 min | |
| 5 | 10 min | |
| 6 | 1 h | |
| 7 | 6 h | |
| 8 | 24 h | last attempt |
| → | dead letter | alert fires |

All delays carry ±20 % jitter to avoid thundering herds after a peer recovers.

Two-tier resilience, deliberately separated:

- **In-process (Polly v8 via `Microsoft.Extensions.Http.Resilience`)** — handles blips:
  3 fast retries on timeout / 5xx / connection failure, per-attempt 10 s timeout, plus a
  **circuit breaker per subscriber** (opens at 50 % failures over 30 s, 30 s break). When the
  breaker is open the dispatcher skips that subscriber entirely instead of burning attempts.
- **Persistent (database)** — handles outages: the table-driven schedule above survives pod
  restarts and multi-hour peer downtime, which in-memory retries cannot.

**Error classification** decides which tier applies:

| Response | Classification | Action |
|---|---|---|
| 2xx | Success | mark `Delivered` |
| 409 / 200 with `alreadyProcessed` | Success (duplicate) | mark `Delivered` |
| 400, 422 (schema/validation) | **Permanent** | straight to dead letter, **no retry** — alert |
| 401, 403 | Permanent-ish | dead letter + credential alert (retrying makes it worse) |
| 404 (endpoint) | Permanent | dead letter + config alert |
| 408, 429, 5xx, timeout, DNS/TCP | Transient | backoff schedule; honour `Retry-After` on 429 |

Retrying a 422 forever is the most common outbox failure mode in the wild — a single
malformed record blocking a queue for days. Permanent failures are quarantined immediately.

Dead-lettered rows are **replayable**: an operator endpoint resets `status = 'Pending'`,
`attempt = 0` after the mapping or the peer is fixed (proposal §9.5).

### 7. Inbox: dedup on the peer's message id

```sql
CREATE UNIQUE INDEX ux_inbox_source_message
    ON integration.inbox_message (source_system, source_message_id);
```

The ingress endpoint does `INSERT … ON CONFLICT DO NOTHING` and returns **202 Accepted**
in both cases. A peer retrying a message it already sent gets a clean 202 and no duplicate
processing. Processing is a separate worker using the same `SKIP LOCKED` claim loop, with the
same backoff schedule and its own dead-letter path.

Inbound processing failures **must not** be reported to the peer as delivery failures —
we accepted the message; a mapping bug is our problem to fix and replay.

### 8. Retention and partitioning

| Table | Retention | Mechanism |
|---|---|---|
| `outbox_change_log` | 7 days after `Materialized` | daily `DELETE` batch |
| `outbox_message` | 30 days | monthly `RANGE` partition on `created_at`, `DETACH` + drop |
| `outbox_delivery` | 30 days after terminal state | follows its parent partition |
| `inbox_message` | 30 days after `Processed` | monthly partition |
| `dead_letter` | 180 days | manual review before purge |

30 days on `outbox_message` sets the **maximum consumer catch-up window**: a consumer offline
longer than that must re-bootstrap via backfill (ADR-0006). This is stated in the contract.
Partition drop is chosen over `DELETE` because deleting from a hot queue table produces bloat
and unpredictable autovacuum behaviour exactly when the queue is busiest.

## Alternatives considered

- **Advisory locks / single-instance dispatcher with leader election.** Rejected: `SKIP LOCKED`
  gives concurrency for free with less machinery and no split-brain risk.
- **Redis or an external queue as the delivery buffer.** Rejected: reintroduces a dual write
  (Postgres → Redis) — the exact problem the outbox solves — plus a new dependency.
- **Strict global ordering via a single-threaded dispatcher.** Rejected: caps throughput at
  one in-flight HTTP request, makes one slow subscriber block all others, and buys ordering
  nobody asked for.
- **`updated_at`-based feed cursor instead of `sequence` + `xid8`.** Rejected: clock skew and
  equal timestamps make it both lossy and non-resumable.
- **Separate database for integration tables.** Rejected: breaks atomicity, which is the point.

## Consequences

### Positive

- Horizontally scalable dispatch with no coordination service.
- The cursor feed is provably non-lossy (`xid8` watermark).
- Edit storms cost one HTTP request, not N.
- Permanent failures are isolated within one attempt instead of poisoning the queue.
- Queue tables stay small and predictable via partition drops.

### Negative / costs accepted

- **Requires PostgreSQL 13+** for `xid8` / `pg_snapshot_xmin`. (Verify the production version
  before implementation — this is a hard prerequisite.)
- **Feed lag equals the longest concurrent write transaction.** Long-running write
  transactions in the domain now have a visible integration cost; worth a `statement_timeout`
  review.
- **Compaction means skipped intermediate states** — the trade-off accepted in ADR-0002,
  restated here because this is where it becomes observable.
- **Extra write amplification** on every business transaction (one change-log insert per
  aggregate touched).
- **Partition management** is new operational work (`pg_partman` or a scheduled Quartz job).
