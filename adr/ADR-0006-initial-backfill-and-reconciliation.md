# ADR-0006 — Initial backfill and ongoing reconciliation

- **Status:** Proposed
- **Date:** 2026-07-25
- **Related:** ADR-0001, ADR-0003, ADR-0004

## Context

Ongoing traffic is moderate, but at go-live the peer has **nothing**. The entire existing
catalogue — every `Foo`, `Bar` and `Baz` — has to reach them before incremental
changes mean anything. A consumer that starts applying deltas to an empty store produces a
partial, silently wrong dataset.

The naive approach ("`UPDATE properties SET updated_at = now()` and let the outbox handle it")
fails in three specific ways: it writes the whole domain table (bloat, lock contention,
replication lag), it floods the delivery queue so that genuine live changes queue behind
hundreds of thousands of backfill messages, and — per ADR-0002 — bulk SQL bypasses the
`ChangeTracker` entirely, so it may not even produce outbox rows.

Backfill must therefore be a **first-class, resumable, throttled, separately-prioritised
mode**, not a side effect.

## Decision

### 1. Snapshot-then-stream handover

The standard, provably correct bootstrap sequence:

```
1. W  := current safe watermark      -- max(sequence) below the xid8 visibility watermark (ADR-0003 §5)
2. Stream the full catalogue as snapshot pages (keyset pagination)
3. Consumer applies snapshots
4. Consumer resumes the incremental feed from cursor W
```

Records changed *during* step 2 are covered twice — once by the snapshot page and once by an
incremental message after W. That is harmless: snapshot payloads plus the
`aggregateVersion` idempotency rule (ADR-0004 §3) make double application converge. **This is
the second time the state-transfer decision pays for itself**, and it is why we do not need a
freeze window, a maintenance mode, or a "stop writes during migration" conversation.

### 2. Backfill is served through the pull feed, not pushed

Backfill runs as a **paged pull** the consumer drives:

```
GET /integration/v1/foos/snapshot?cursor={opaque}&limit=500
Authorization: Bearer …            (scope: acme.backfill.read)

200 OK
{
  "watermark":   "W-1284412",       // returned on the FIRST page and pinned for the whole run
  "items":       [ /* 500 contract snapshots */ ],
  "nextCursor":  "eyJpZCI6IjljMWIu…",
  "hasMore":     true
}
```

- **Keyset pagination on `(created_at, id)`** — never `OFFSET`, which degrades quadratically
  and skips or repeats rows when data shifts under a long-running scan.
- The consumer controls the pace, so **backfill cannot outrun them** and cannot saturate our
  dispatcher.
- Resumable by definition: the cursor is the state, held by the consumer. A crash on page 1 400
  of 2 000 resumes at 1 400.
- Pages are materialised **on demand** from current domain state and are not written to the
  outbox at all — so backfill leaves no 30-day retention footprint and no queue pressure.

`watermark` is returned on the first page and echoed on every page so that a consumer that
loses it can recover; the consumer uses it as the incremental starting cursor once `hasMore`
is false.

### 3. Push-mode backfill exists, but is separately queued and throttled

Some peers will not implement a puller. For them, backfill can be pushed — with hard
separation from live traffic:

- Rows carry `origin = 'Backfill'` and `priority = 9` (live changes are `priority = 0`).
  **The dispatcher always drains priority 0 first**, so a two-day backfill never delays a
  live price change.
- A dedicated `backfill_run` table tracks `(id, aggregate_type, subscriber_id, cursor,
  total_estimated, completed_count, rate_limit_per_second, status, started_at, finished_at,
  last_error)` — pausable and resumable from the admin API.
- Throttled by a token bucket (default 50 msg/s, configurable per run) to protect both the
  peer and our database.
- Backfill messages are **never compacted against live messages** (a stale backfill snapshot
  must not supersede a fresh live update); the version check on the consumer side is the
  backstop, and the dispatcher additionally refuses to send a backfill message whose
  `aggregateVersion` is lower than an already-delivered live one.

### 4. Ongoing reconciliation

Backfill is not a one-time concern — divergence accumulates from dead letters, bugs, bulk SQL
bypassing capture (ADR-0002), and peer-side failures. A nightly job detects it cheaply:

```
GET /integration/v1/foos/checksums?bucket=modulo256

{ "generatedAt": "…", "buckets": [ { "bucket": 0, "count": 4 102, "hash": "9f2c…" }, … ] }
```

- Each aggregate contributes `sha256(canonical_contract_json)`; bucket hash is the XOR of member
  hashes (order-independent, cheap to maintain, no sort required).
- The consumer compares bucket hashes; only mismatching buckets are drilled into
  (`?bucket=17&detail=true` returns per-id hashes), then only genuinely divergent ids are
  re-fetched. 256 buckets over a 500 k catalogue means a full daily consistency check costs
  one small response instead of a full export.
- A mismatch raises an alert on our side too — it usually means messages were dead-lettered
  and nobody looked.

**Repair path:** a targeted re-emit endpoint (`POST /integration/v1/admin/reemit` with a list
of aggregate ids) re-materialises and re-queues those aggregates for a given subscriber.

### 5. Deletes during backfill

A snapshot feed cannot express "this id no longer exists". The consumer's rule, stated in the
contract: **after a full backfill run completes, any local record whose `aggregateId` was not
present in the run and whose `aggregateVersion` predates the run's watermark is stale** and
must be reconciled via the checksum endpoint (which will simply not list it). Hard deletes in
our system additionally emit a tombstone through the incremental feed
(`changeKind = "Deleted"`), retained for the full 30-day window.

## Alternatives considered

- **Touch every row to force outbox emission.** Rejected — bloat, lock contention, replication
  lag, queue flooding, and (per ADR-0002) bulk SQL may not even trigger capture. It also
  pollutes domain audit columns with a fake modification.
- **CSV/Parquet dump to shared storage.** Fast for the very first load and worth revisiting if
  the catalogue reaches millions of rows, but it needs a second format, a second mapping path,
  and a second security review (object-store credentials, ADR-0005). Not justified at this scale.
- **Freeze writes during migration.** Rejected: unnecessary given snapshot semantics, and a
  business non-starter.
- **Full daily re-export as the reconciliation mechanism.** Rejected: expensive, and it hides
  the fact that the incremental path is broken instead of surfacing it.
- **`OFFSET`-based paging.** Rejected: quadratic cost and non-repeatable reads under concurrent
  writes — it silently skips records, which is precisely the failure mode backfill exists to
  prevent.

## Consequences

### Positive

- Go-live needs no downtime, freeze window, or coordinated cutover.
- Backfill is resumable, throttled, and structurally incapable of starving live traffic.
- The same endpoint serves initial load, disaster recovery, and new-subscriber onboarding —
  one mechanism, three problems.
- Divergence becomes observable daily instead of being discovered by a business user.

### Negative / costs accepted

- **Snapshot pages are computed on demand**, so a large backfill puts sustained read load on the
  primary. Mitigation: serve backfill and checksum queries from a **read replica** (they need no
  write consistency, and the pinned watermark makes replica lag harmless).
- **Double delivery during the handover window** — accepted, harmless under version-based
  idempotency, but it must be stated in the consumer contract or someone will file it as a bug.
- **Checksum computation** requires a canonical JSON form (stable key order, normalised
  numbers/dates). That canonicaliser is shared with the contract serialiser and must be
  version-pinned: changing it changes every hash and triggers a false full-divergence alert.
  Treat a canonicaliser change as a contract change.
- **The consumer must implement the checksum comparison** for reconciliation to have value.
  If they will not, we still run it internally against our own last-delivered payload hashes —
  weaker, but it catches our own dead-letter gaps.
