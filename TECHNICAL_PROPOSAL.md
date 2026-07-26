# Technical Proposal — External Service Integration for Acme

- **Version:** 1.0 (draft for review)
- **Date:** 2026-07-25
- **Author:** Backend / Integration
- **Status:** Proposed — pending answers to the open questions in §2.3
- **ADRs:** [0001](adr/ADR-0001-integration-style-and-transport.md) ·
  [0002](adr/ADR-0002-change-capture-without-touching-the-domain.md) ·
  [0003](adr/ADR-0003-outbox-inbox-storage-and-delivery-semantics.md) ·
  [0004](adr/ADR-0004-contract-mapping-payload-and-versioning.md) ·
  [0005](adr/ADR-0005-authentication-and-transport-security.md) ·
  [0006](adr/ADR-0006-initial-backfill-and-reconciliation.md) ·
  [0007](adr/ADR-0007-staged-delivery-and-the-first-increment.md)
- **Companion documents:** [MAPPING_MATRIX.md](MAPPING_MATRIX.md) ·
  [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md)

---

## 1. Summary

Acme must exchange `Foo`, `Bar` and `Baz` data bidirectionally with
other internal company services over HTTP. Kafka is unavailable, the peer owns the wire format,
and the domain model must not be modified.

**Proposal:** a transactional **Outbox/Inbox** integration layer living entirely in the
infrastructure ring, with a **two-stage outbox** (cheap transactional change capture →
deferred contract materialisation), **hybrid push + pull** delivery, an **Anti-Corruption
Layer** whose completeness is enforced by the compiler and the test suite, and a **resumable
backfill** for go-live.

Four properties this buys, in priority order:

1. **No lost changes** — the message becomes durable in the same transaction as the business
   state. No dual write anywhere in the design.
2. **No domain contamination** — verified by architecture tests; the domain assemblies do not
   compile against anything in `Integration`.
3. **Provable losslessness** — every domain field is mapped or explicitly excluded with a
   reason, enforced at build time (Mapperly diagnostics-as-errors) and by a reflection-based
   coverage test that names any field you forget.
4. **Broker-ready** — when Kafka arrives, only the dispatcher's transport changes. The outbox,
   contracts, mapping and inbox all survive.

Estimated effort: **~4 weeks of engineering for one engineer to production-ready** — calendar time
is longer, because `MAPPING_MATRIX.md` §7 carries eight gaps that each need a round trip with
another team. Delivery is staged (ADR-0007): a peer can `GET` our catalogue in their contract shape
at the **end of stage 1, day 2**, with no new tables and no change to the write path.

---

## 2. Scope, assumptions and open questions

### 2.0 Domain independence

**This design does not depend on what the aggregates mean.** It is selected by four structural facts,
none of which is about subject matter:

| Fact | What it forces |
|---|---|
| Aggregates change inside database transactions | Capture must be transactional, or changes are lost — hence the outbox (ADR-0001) |
| Someone outside the service must observe those changes | A durable log with two read paths, push and pull (ADR-0001) |
| The wire format is owned by someone else | An anti-corruption layer, and losslessness enforced by the build (ADR-0004) |
| No broker is available | We reproduce the slice of one we need, broker-ready (ADR-0001) |

Any domain matching those four sentences fits this design with **no change other than the aggregate
names**: orders and shipments, patients and encounters, accounts and ledger entries, devices and
telemetry, tickets and assignments.

`Foo`, `Bar`, `Baz`, `Acme` and `peer-a` are placeholders throughout — see
[CLAUDE.md](CLAUDE.md#naming--all-names-are-placeholders). `Bar` and `Baz` exist so the
**fan-out** case (a referenced aggregate changes, so every `Foo` referencing it must be re-emitted)
and the **flattening** case (a hierarchy the peer wants flat) have somewhere to live. Delete them
and those two problems become invisible until an implementer hits them.

Value-level concepts stay concrete — money, quantities with explicit units, date-only versus
timestamp, coordinates, contact details. They are cross-domain, and §7 plus `MAPPING_MATRIX.md` §6
exist because of them.

### 2.1 In scope

- Outbound change publication for `Foo`, `Bar`, `Baz`.
- Inbound ingestion from peer services into the domain via an ACL.
- Outbox/inbox schema, workers, delivery, retry, dead-lettering, replay.
- Cursor-based change feed and snapshot/backfill endpoints.
- Contract mapping with losslessness guarantees.
- Observability, alerting, runbook, reconciliation.

### 2.2 Out of scope

- Modifying the domain model (explicit constraint).
- Introducing any message broker.
- Peer-side implementation (their inbox, their idempotency).
- Real-time (sub-second) propagation guarantees.
- Replacing existing synchronous read APIs.

### 2.3 Open questions — **blocking, need answers before/during week 1**

| # | Question | Blocks | Default if unanswered |
|---|---|---|---|
| Q1 | Snapshot or delta payloads? (ADR-0004 §3) | Dispatcher ordering, feed design | **Build for snapshots** |
| Q2 | Which auth mechanism is available? (ADR-0005) | Ingress + outbound clients | **OAuth2 client credentials**, HMAC fallback |
| Q3 | Peer OpenAPI schema — where is it? | All mapping work | Blocks mapper implementation entirely |
| Q4 | Embed `Bar`/`Baz` in `Foo`, or references only? | Materializer fan-out | **References only** |
| Q5 | PostgreSQL major version in production? | `xid8` watermark (needs 13+) | Verify before day 1 |
| Q6 | Is the peer authorised to receive contact PII? | Mapping + redaction | **Redact until confirmed** |
| Q7 | Does the peer push to us, do we poll them, or both? | Inbound design | **Build ingress first**, poller second |
| Q8 | Catalogue size (row counts per aggregate)? | Backfill sizing | Assume ~500 k `Foo` records |
| Q17 | Is there a pilot consumer who can read a stage 1 read-only API? (ADR-0007) | The value of stage 1, and therefore the staging order | **Assume yes**; if none exists, re-read ADR-0007 alternative (A) |

Q3 is the true critical path: **nothing in the mapping layer can be finished without the
peer's schema.** Everything else (outbox, dispatcher, feed, inbox plumbing) can proceed in
parallel against a stub contract.

### 2.4 Assumptions

- .NET 10 / C# 14, EF Core 10, Npgsql, PostgreSQL 13+.
- The service runs multiple replicas in Kubernetes; workers run in-process alongside the API
  (a separate worker deployment is a configuration change, §11.3).
- Corporate network HTTP is available between the services; no internet egress needed.
- Moderate steady-state volume (thousands of changes/day) plus a one-off full backfill.

---

## 3. Architecture

### 3.1 Overall flow

```
        ┌──────────────────────── Acme ────────────────────────┐
        │                                                                 │
 HTTP   │  Api ──► Application ──► Domain ──► EF Core ──┐                 │
 write  │                          (UNCHANGED)          │ SaveChangesAsync│
 ───────┼──►                                            ▼                 │
        │                        ┌── SaveChangesInterceptor ──┐           │
        │                        │  change capture (ADR-0002) │           │
        │                        └──────────┬─────────────────┘           │
        │                     ONE TRANSACTION│                            │
        │            ┌───────────────────────▼──────────────────────┐     │
        │            │ core.*  +  integration.outbox_change_log    │     │
        │            └───────────────────────┬──────────────────────┘     │
        │                                    │ async                      │
        │            ┌───────────────────────▼──────────────────────┐     │
        │            │ MaterializerWorker                            │     │
        │            │  load aggregate → Snapshot → ACL → Contract   │     │
        │            │  validate vs peer JSON Schema                 │     │
        │            └───────────────────────┬──────────────────────┘     │
        │                                    ▼                            │
        │            outbox_message ──fan-out──► outbox_delivery          │
        │                    │                        │                   │
        │                    │                        ▼                   │
        │                    │              DispatcherWorker ────HTTP────────►  Peer
        │                    │              (Polly, breaker, DLQ)  POST     │   service
        │                    ▼                                             │
        │            GET /integration/v1/*/changes?cursor=… ◄──────HTTP─────────  Peer
        │            GET /integration/v1/*/snapshot?cursor=…               │   (pull)
        │            GET /integration/v1/*/checksums                       │
        │                                                                  │
        │  ── inbound ───────────────────────────────────────────────────  │
        │                                                                  │
        │   POST /integration/v1/inbound/{type} ◄────HTTP──────────────────────  Peer
        │            │  authn + schema validation only                     │   (push)
        │            ▼                                                     │
        │      integration.inbox_message ──► InboxWorker ──► Translator    │
        │                                          │          (ACL)        │
        │                                          ▼                       │
        │                                   Application command ──► Domain │
        │                                                                  │
        │   InboundPollerWorker ────────HTTP GET────────────────────────────────►  Peer
        └──────────────────────────────────────────────────────────────────┘
```

### 3.2 Project layout

Existing projects untouched; three new ones:

```
src/
├── Acme.Domain/                    ← UNCHANGED
├── Acme.Application/               ← + IIntegrationEventEnqueuer (interface only)
├── Acme.Infrastructure/            ← + interceptor registration, EF configs
├── Acme.Api/                       ← + endpoint mapping only
│
├── Acme.Integration/               ← NEW: outbox, inbox, workers, mappers, ACL
│   ├── ChangeCapture/                       (interceptor, aggregate root resolver)
│   ├── Outbox/                              (entities, materializer, dispatcher, feed)
│   ├── Inbox/                               (ingress handler, inbox worker, translators)
│   ├── Snapshots/                           (internal flat read models)
│   ├── Mapping/                             (Mapperly mappers = the ACL)
│   ├── Delivery/                            (typed HttpClients, resilience, auth handlers)
│   ├── Backfill/                            (snapshot pager, checksums, runs)
│   └── Persistence/                         (EF configs, migrations for `integration` schema)
│
├── Acme.Integration.Contracts/     ← NEW: generated from the peer's OpenAPI
└── tests/Acme.Integration.Tests/   ← NEW: unit + Testcontainers + WireMock
```

**Dependency rule (architecture-tested):**
`Integration → Application → Domain`. Never the reverse. `Domain` and `Application` must not
reference `Integration` or `Integration.Contracts`.

### 3.3 Library choices and licensing

| Purpose | Library | License | Note |
|---|---|---|---|
| PostgreSQL / EF Core | `Npgsql`, `Npgsql.EntityFrameworkCore.PostgreSQL` | PostgreSQL (MIT-like) ✅ | already in use |
| Object mapping (ACL) | **`Mapperly`** | Apache-2.0 ✅ | compile-time; diagnostics-as-errors are load-bearing (ADR-0004 §5a) |
| HTTP resilience | `Microsoft.Extensions.Http.Resilience` (Polly v8) | MIT ✅ | retry, timeout, circuit breaker |
| Validation | `FluentValidation` | Apache-2.0 ✅ | inbound DTO validation |
| JSON Schema validation | `JsonSchema.Net` | MIT ✅ | payload vs vendored peer schema |
| Contract codegen | `Kiota` or `NSwag` | MIT ✅ | from peer OpenAPI |
| Scheduling | `Quartz.NET` (only if cron-style jobs needed) | Apache-2.0 ✅ | plain `BackgroundService` preferred |
| Logging | `Serilog` | Apache-2.0 ✅ | + redaction policy (ADR-0005 §3) |
| Telemetry | `OpenTelemetry.*` | Apache-2.0 ✅ | traces + metrics |
| Tests | `xUnit`, `NSubstitute`, `Testcontainers.PostgreSql`, `WireMock.Net` | Apache-2.0 / MIT ✅ | |
| Assertions | `FluentAssertions` **v7** | MIT ✅ | ⚠️ **v8+ is commercial — pin to v7** (or use `Shouldly`, MIT) |

**Deliberately avoided for licensing:** `MediatR` v13+ (RPL-1.5 commercial),
`AutoMapper` v14+ (commercial), `MassTransit` v9+ (commercial), `Hangfire.Pro` (paid).
None are needed: handlers use the existing application pattern, mapping uses Mapperly, and
`BackgroundService` + `SKIP LOCKED` replaces a job framework entirely.

---

## 4. Data model

Full DDL for the `integration` schema. Managed by EF Core migrations in
`Acme.Integration/Persistence/Migrations`, applied by a dedicated migration job
(not on app startup).

```sql
CREATE SCHEMA IF NOT EXISTS integration;
```

### 4.1 `outbox_change_log` — stage 1, transactional capture

```sql
CREATE TABLE integration.outbox_change_log (
    id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aggregate_type text        NOT NULL,
    aggregate_id   uuid        NOT NULL,
    change_kind    text        NOT NULL
                   CHECK (change_kind IN ('Created','Updated','Deleted','Event')),
    occurred_at    timestamptz NOT NULL,
    correlation_id uuid,
    causation_id   uuid,
    trace_parent   text,
    actor_id       text,
    -- original values for deletes; pre-built payload for explicit integration events
    snapshot_hint  jsonb,
    status         text        NOT NULL DEFAULT 'Pending'
                   CHECK (status IN ('Pending','Materialized','Failed','Skipped')),
    attempt        int         NOT NULL DEFAULT 0,
    last_error     text,
    insert_xid     xid8        NOT NULL DEFAULT pg_current_xact_id(),
    created_at     timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX ix_change_log_claim
    ON integration.outbox_change_log (id)
    WHERE status = 'Pending';

CREATE INDEX ix_change_log_aggregate
    ON integration.outbox_change_log (aggregate_type, aggregate_id, id DESC);
```

Rows are intentionally tiny — this table is written inside every business transaction, so its
insert cost is on the user's critical path and nothing else here is.

### 4.2 `outbox_message` — stage 2, the contract payload (partitioned)

```sql
CREATE TABLE integration.outbox_message (
    sequence          bigint GENERATED ALWAYS AS IDENTITY,
    id                uuid        NOT NULL,          -- messageId, UUIDv7, stable across retries
    aggregate_type    text        NOT NULL,
    aggregate_id      uuid        NOT NULL,
    aggregate_version bigint      NOT NULL,          -- monotonic; consumer idempotency key
    message_type      text        NOT NULL,          -- acme.foo.changed
    contract_version  text        NOT NULL,          -- '1'
    change_kind       text        NOT NULL,
    payload           jsonb       NOT NULL,          -- full envelope incl. data
    payload_hash      bytea       NOT NULL,          -- sha256(canonical json) — reconciliation
    compactable       boolean     NOT NULL DEFAULT true,
    origin            text        NOT NULL DEFAULT 'Live'    -- Live | Backfill | Replay
                      CHECK (origin IN ('Live','Backfill','Replay')),
    occurred_at       timestamptz NOT NULL,
    produced_at       timestamptz NOT NULL DEFAULT now(),
    correlation_id    uuid,
    trace_parent      text,
    insert_xid        xid8        NOT NULL DEFAULT pg_current_xact_id(),
    created_at        timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (sequence, created_at)               -- partition key must be in the PK
) PARTITION BY RANGE (created_at);

CREATE UNIQUE INDEX ux_outbox_message_id
    ON integration.outbox_message (id, created_at);

CREATE INDEX ix_outbox_message_feed
    ON integration.outbox_message (sequence)
    INCLUDE (insert_xid);

CREATE INDEX ix_outbox_message_aggregate
    ON integration.outbox_message (aggregate_type, aggregate_id, sequence DESC);

-- monthly partitions, created ahead of time by a scheduled job or pg_partman
CREATE TABLE integration.outbox_message_2026_08
    PARTITION OF integration.outbox_message
    FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

> **Note:** partitioning forces `created_at` into the primary key and every unique index.
> If that complicates EF Core mapping more than it is worth at current volume, ship
> unpartitioned in phase 1 with a scheduled `DELETE … WHERE created_at < now() - 30 days`
> in batches, and partition later. **Recommendation: ship unpartitioned, partition when the
> table exceeds ~10 M rows.** The DDL above is the target state.

### 4.3 `outbox_delivery` — per-subscriber fan-out

```sql
CREATE TABLE integration.outbox_delivery (
    id               bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    message_id       uuid        NOT NULL,
    message_sequence bigint      NOT NULL,
    subscriber_id    text        NOT NULL REFERENCES integration.subscriber(id),
    aggregate_key    text        NOT NULL,            -- 'Foo:9c1b…' — compaction key
    aggregate_version bigint     NOT NULL,
    status           text        NOT NULL DEFAULT 'Pending'
                     CHECK (status IN ('Pending','InFlight','Delivered','Superseded','DeadLettered')),
    priority         smallint    NOT NULL DEFAULT 0,  -- 0 = live, 9 = backfill
    attempt          int         NOT NULL DEFAULT 0,
    next_attempt_at  timestamptz NOT NULL DEFAULT now(),
    locked_by        text,
    locked_until     timestamptz,
    last_status_code int,
    last_error       text,
    completed_at     timestamptz,
    created_at       timestamptz NOT NULL DEFAULT now()
);

-- the claim query's index; column order matches ORDER BY exactly
CREATE INDEX ix_delivery_claim
    ON integration.outbox_delivery (subscriber_id, priority, next_attempt_at, id)
    WHERE status = 'Pending';

-- compaction lookup
CREATE INDEX ix_delivery_compaction
    ON integration.outbox_delivery (subscriber_id, aggregate_key, message_sequence)
    WHERE status = 'Pending';

-- stuck-lock reaper
CREATE INDEX ix_delivery_stuck
    ON integration.outbox_delivery (locked_until)
    WHERE status = 'InFlight';
```

### 4.4 `subscriber` — who receives what

```sql
CREATE TABLE integration.subscriber (
    id                text        PRIMARY KEY,        -- 'peer-a'
    display_name      text        NOT NULL,
    callback_url      text,                            -- NULL = pull-only subscriber
    contract_version  text        NOT NULL DEFAULT '1',
    auth_mode         text        NOT NULL,            -- OAuth2 | Mtls | Hmac
    auth_config_ref   text        NOT NULL,            -- secret manager key, NEVER the secret
    aggregate_types   text[]      NOT NULL,            -- {'Foo','Bar'}
    filter_expression jsonb,                           -- subscriber-scoped filters, MAPPING_MATRIX §5
    pii_granted       boolean     NOT NULL DEFAULT false,
    rate_limit_rps    int         NOT NULL DEFAULT 20,
    enabled           boolean     NOT NULL DEFAULT true,
    created_at        timestamptz NOT NULL DEFAULT now(),
    updated_at        timestamptz NOT NULL DEFAULT now()
);
```

Subscribers are **data, not code**: onboarding a new consumer is a row plus an IdP client, with
no redeploy. `callback_url` is validated against an internal-host allowlist at write time
(ADR-0005 §3, SSRF).

### 4.5 `inbox_message` — inbound deduplication

```sql
CREATE TABLE integration.inbox_message (
    id                bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    source_system     text        NOT NULL,
    source_message_id text        NOT NULL,
    message_type      text        NOT NULL,
    contract_version  text,
    payload           jsonb       NOT NULL,
    received_at       timestamptz NOT NULL DEFAULT now(),
    status            text        NOT NULL DEFAULT 'Pending'
                      CHECK (status IN ('Pending','InFlight','Processed','DeadLettered','Ignored')),
    attempt           int         NOT NULL DEFAULT 0,
    next_attempt_at   timestamptz NOT NULL DEFAULT now(),
    locked_by         text,
    locked_until      timestamptz,
    processed_at      timestamptz,
    last_error        text,
    trace_parent      text
);

CREATE UNIQUE INDEX ux_inbox_dedup
    ON integration.inbox_message (source_system, source_message_id);

CREATE INDEX ix_inbox_claim
    ON integration.inbox_message (next_attempt_at, id)
    WHERE status = 'Pending';
```

### 4.6 Supporting tables

```sql
CREATE TABLE integration.dead_letter (
    id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    direction     text        NOT NULL CHECK (direction IN ('Outbound','Inbound')),
    subscriber_id text,
    source_system text,
    message_id    uuid,
    aggregate_type text,
    aggregate_id  uuid,
    payload       jsonb,
    failure_kind  text        NOT NULL,   -- Validation | Mapping | Permanent4xx | Exhausted
    error         text        NOT NULL,
    attempts      int         NOT NULL,
    first_failed_at timestamptz NOT NULL,
    dead_lettered_at timestamptz NOT NULL DEFAULT now(),
    replayed_at   timestamptz,
    replayed_by   text
);

-- our cursor when WE poll a peer
CREATE TABLE integration.consumer_cursor (
    source_system text        PRIMARY KEY,
    cursor        text        NOT NULL,
    updated_at    timestamptz NOT NULL DEFAULT now()
);

-- id correlation: ours <-> theirs, so we never overwrite our own key
CREATE TABLE integration.external_reference (
    aggregate_type text NOT NULL,
    aggregate_id   uuid NOT NULL,
    system         text NOT NULL,
    external_id    text NOT NULL,
    created_at     timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, system)
);
CREATE UNIQUE INDEX ux_external_reference_reverse
    ON integration.external_reference (system, external_id);

CREATE TABLE integration.backfill_run (
    id             uuid PRIMARY KEY,
    aggregate_type text NOT NULL,
    subscriber_id  text,                      -- NULL = pull-mode run
    cursor         text,
    total_estimated bigint,
    completed_count bigint NOT NULL DEFAULT 0,
    rate_limit_rps int    NOT NULL DEFAULT 50,
    watermark      bigint,                    -- pinned outbox sequence for the handover
    status         text   NOT NULL DEFAULT 'Running'
                   CHECK (status IN ('Running','Paused','Completed','Failed')),
    started_at     timestamptz NOT NULL DEFAULT now(),
    finished_at    timestamptz,
    last_error     text
);
```

---

## 5. Outbound pipeline

### 5.1 Stage 1 — change capture

Per ADR-0002. Key implementation points:

- `SaveChangesInterceptor.SavingChangesAsync` adds change-log entities to the same `DbContext`.
- `IAggregateRootResolver` maps a tracked entity to `(rootType, rootId)` via an explicit,
  startup-validated registration map.
- Deleted aggregates capture `OriginalValues` into `snapshot_hint` — the row is gone by
  materialisation time.
- One row per aggregate per transaction (`Distinct()`), not one per changed table row.
- **Guard:** a banned-symbols/analyzer rule flags `ExecuteUpdateAsync`/`ExecuteDeleteAsync` on
  integrated entity types, which bypass the `ChangeTracker` entirely.

### 5.2 Stage 2 — materialisation

`MaterializerWorker` (a `BackgroundService`), loop:

1. Claim a batch of `Pending` change-log rows below the `xid8` watermark, `FOR UPDATE SKIP LOCKED`.
2. **Collapse** by `(aggregate_type, aggregate_id)`, keeping the newest — an aggregate edited
   five times in the batch is materialised once.
3. Load aggregates in bulk: `WHERE id = ANY(@ids)`, `AsNoTracking()`, projected straight into
   the flat `Snapshot` read model (no aggregate rehydration, no lazy loading).
4. Apply filter rules (MAPPING_MATRIX §5) → publish, tombstone, or skip.
5. Map `Snapshot → Contract` through Mapperly (the ACL).
6. Compute `aggregateVersion` (see §5.3), build the envelope, compute `payload_hash`.
7. **Validate against the peer's vendored JSON Schema.** Failure ⇒ dead letter now, with the
   offending field named — never a doomed retry loop against the peer.
8. Insert `outbox_message`; fan out `outbox_delivery` rows for every enabled subscriber whose
   `aggregate_types` and `filter_expression` match, at each subscriber's `contract_version`.
9. **Compact** superseded pending deliveries (ADR-0003 §4).
10. Mark change-log rows `Materialized`.

Steps 7–10 run in one transaction, so a crash re-materialises rather than half-emitting.

### 5.3 `aggregateVersion`

Must be monotonic per aggregate and comparable by the consumer. Options, in order of preference:

1. **`xmin` as a concurrency token** — EF Core supports `builder.UseXminAsConcurrencyToken()`,
   and PostgreSQL bumps it on every row update for free. Caveat: `xid` wraps around, so it is
   comparable only over a bounded window. **Not safe as a durable version.**
2. **A dedicated monotonic counter per aggregate** — a `version bigint` column bumped on write.
   Adding it to a domain table is a schema change but *not* a domain-model change (it is an EF
   shadow property: `builder.Property<long>("IntegrationVersion")`), so it respects the
   constraint. **Recommended.**
3. **The outbox `sequence`** — globally monotonic, trivially available, and monotonic per
   aggregate as a consequence. Simplest, and sufficient for last-writer-wins. **Recommended
   default for phase 1**; switch to (2) if a consumer needs a version tied to our state rather
   than to our emission order.

Ship with (3), design the envelope so switching to (2) is a materializer change only.

### 5.4 Dispatch (push)

`DispatcherWorker`, one logical loop per subscriber (bounded parallelism across subscribers):

1. Skip the subscriber entirely if its circuit breaker is open.
2. Claim a batch (ADR-0003 §2 SQL), priority 0 before priority 9.
3. POST the envelope with headers:
   `Idempotency-Key: {messageId}`, `traceparent`, `Content-Type: application/vnd.acme.foo.v1+json`,
   plus auth per ADR-0005.
4. Classify the response (ADR-0003 §6 table) → `Delivered`, backoff, or dead letter.
5. `AllowAutoRedirect = false`; per-request timeout 10 s; per-attempt Polly retry for blips.

Batching: if the peer exposes a bulk endpoint, send up to N envelopes per request and process
a per-item result array — otherwise one request per message with pipelined concurrency
(default 4 in flight per subscriber, bounded by `rate_limit_rps`).

### 5.5 Change feed (pull)

```
GET /integration/v1/foos/changes?cursor={seq}&limit=500
Authorization: Bearer …                       scope: acme.feed.read

200 OK
{
  "items":      [ { …envelope… } ],
  "nextCursor": "1284412",
  "hasMore":    true
}
```

- Cursor is the opaque `sequence`. Consumer owns it — we store nothing per consumer.
- **`insert_xid < pg_snapshot_xmin(pg_current_snapshot())` is mandatory** on this query.
  Without it the feed silently skips rows (ADR-0003 §5). This is the single most important
  line of SQL in the proposal.
- `limit` capped at 1 000; `AsNoTracking()`; served from a read replica where available.
- Deleted aggregates appear as tombstone envelopes, not as absences.
- Retention (30 days) sets the maximum catch-up window; a cursor older than the oldest retained
  row returns **410 Gone** with `{"error":"cursor_expired","action":"backfill"}` — an explicit,
  actionable failure instead of a silently incomplete response.

### 5.6 Backfill and reconciliation endpoints

Per ADR-0006: `GET …/snapshot?cursor=&limit=` (keyset paged, pinned watermark) and
`GET …/checksums?bucket=` (256-bucket XOR hashes), plus admin `POST …/admin/reemit`.

---

## 6. Inbound pipeline

### 6.1 Ingress endpoint

```csharp
group.MapPost("/inbound/{messageType}", async (
    string messageType,
    [FromBody] JsonElement body,
    HttpContext ctx,
    IInboxWriter inbox,
    IContractSchemaValidator validator,
    CancellationToken ct) =>
{
    // 1. authN/authZ handled by the auth middleware + scope policy (ADR-0005)
    // 2. structural validation only — no domain access
    var validation = validator.Validate(messageType, body);
    if (!validation.IsValid)
        return TypedResults.ValidationProblem(validation.Errors);   // 400: peer bug, tell them

    // 3. durable, deduplicated insert
    var outcome = await inbox.AcceptAsync(new InboundMessage(
        SourceSystem:    ctx.User.FindFirst("client_id")!.Value,
        SourceMessageId: ctx.Request.Headers["Idempotency-Key"].ToString(),
        MessageType:     messageType,
        Payload:         body,
        TraceParent:     Activity.Current?.Id), ct);

    // 202 for both new and duplicate — a retrying peer must not get an error
    return TypedResults.Accepted($"/integration/v1/inbound/{outcome.InboxId}");
})
.RequireAuthorization("inbound.write")
.WithName("AcceptInboundMessage");
```

Deliberate properties: constant-time work, no domain access, no transaction beyond one insert,
idempotent by `ON CONFLICT DO NOTHING`, and a validation failure that is unambiguously
attributed to the peer (400) rather than to us (500).

Missing `Idempotency-Key`: reject with 400 rather than minting one — a peer that cannot
identify its own message cannot be deduplicated, and silently accepting guarantees duplicates.

### 6.2 Inbox worker and translation

Claim → translate (peer contract → our command, via the inbound ACL) → execute the existing
application command handler → mark `Processed`. Same backoff and dead-letter machinery as
outbound.

Rules:

- **Translation errors are permanent** (dead letter immediately, alert): a payload that does not
  map will not map on retry.
- **Concurrency conflicts are transient** (retry with backoff): the aggregate was busy.
- **Domain rule rejections are terminal but expected**: dead letter with `failure_kind =
  'DomainRejected'`, and — if the peer's contract supports it — a negative acknowledgement
  callback. Do not retry; the domain said no.
- Inbound writes go through the **existing application command handlers**, never through direct
  `DbContext` mutation. All domain invariants apply to external data exactly as to user input.
  This is the whole point of the ACL.
- The inbound path is itself change-captured, so absorbing a peer update naturally re-emits it
  outbound. **Echo suppression is required**: messages produced from an inbound message of
  source S carry `causationSource = S` and are not fanned out back to S.

### 6.3 Outbound polling (if we pull from a peer)

`InboundPollerWorker`: read `consumer_cursor`, `GET {peer}/changes?since=…`, insert each item
into `inbox_message`, advance the cursor **only after the batch is durably inserted**. Same
resilience policies. Cursor advancement after insertion (not after processing) is correct
because the inbox is itself durable and retried.

---

## 7. Contract and mapping

See ADR-0004 and MAPPING_MATRIX.md. Enforcement summary:

| Guarantee | Mechanism | Fails at |
|---|---|---|
| Unmapped member | Mapperly `RMG012`/`RMG020` as errors | compile |
| Domain field not accounted for | reflection coverage test | test |
| Value corruption (rounding, TZ, truncation) | round-trip property tests | test |
| Enum drift | exhaustive `switch`, no silent default | compile / first message |
| Peer schema drift | vendored schema + daily diff job | CI |
| Invalid payload | JSON Schema validation at materialisation | before delivery |

---

## 8. Security

See ADR-0005. Implementation checklist:

- [ ] OAuth2 client-credentials `DelegatingHandler` with token cache (refresh at 80 % TTL)
- [ ] JWT bearer validation with JWKS caching; issuer/audience/exp/nbf; per-endpoint `client_id` allowlist
- [ ] Scope-based authorization policies (`feed.read`, `inbound.write`, `backfill.read`, `admin`)
- [ ] Secrets from the secret manager only; `auth_config_ref` in the DB, never the secret
- [ ] Serilog redaction policy for `Authorization`, `X-Signature`, `X-Api-Key`, `*token*`, `*secret*`
- [ ] Rate limiting per `client_id`; `MaxRequestBodySize` 1 MB
- [ ] `AllowAutoRedirect = false`; callback URL allowlist validated at registration and dispatch
- [ ] PII gating per subscriber (`pii_granted`) with a test asserting redaction
- [ ] `integration.purge_subject(subjectId)` for erasure requests

---

## 9. Operations

### 9.1 Metrics (OpenTelemetry)

| Metric | Type | Why it matters |
|---|---|---|
| `integration_outbox_pending_age_seconds` | gauge | **Primary SLI.** Age of the oldest pending row. One number that says "is the pipeline healthy". |
| `integration_outbox_pending_count` | gauge | queue depth by stage and subscriber |
| `integration_delivery_attempts_total` | counter | by subscriber, status class |
| `integration_delivery_duration_seconds` | histogram | peer latency, p50/p95/p99 |
| `integration_deadletter_total` | counter | by direction, failure kind |
| `integration_materialization_duration_seconds` | histogram | mapping cost |
| `integration_mapping_unmapped_total` | counter | by field — the silent-loss detector |
| `integration_feed_requests_total` | counter | by consumer, status |
| `integration_inbox_pending_age_seconds` | gauge | inbound equivalent of the primary SLI |
| `integration_reconciliation_divergent_total` | gauge | from the nightly checksum job |
| `integration_circuit_state` | gauge | breaker state per subscriber |

### 9.2 Alerts

| Alert | Condition | Severity |
|---|---|---|
| Outbox stalled | `pending_age > 300 s` for 5 min | **page** |
| Outbox growing | `pending_count` up 3 intervals with no deliveries | page |
| Dead letters | any new dead letter | ticket (page if > 10/h) |
| Subscriber down | breaker open > 15 min | ticket |
| Permanent 4xx spike | `> 5` 400/422 in 10 min | **page** — usually a contract break |
| Auth failures | any 401/403 outbound | page — credential expiry |
| Divergence | `reconciliation_divergent > 0` | ticket |
| Unmapped values | `mapping_unmapped_total > 100/day` | ticket |
| Schema drift | daily diff job fails | ticket |

### 9.3 Tracing

W3C trace context flows: HTTP request → command handler → change-log row (`trace_parent`) →
materializer (linked span) → dispatcher → peer's `traceparent` header. A single trace shows
the whole journey of a change, which is what makes "where did this update go?" a two-minute
question instead of a two-hour one.

### 9.4 Reconciliation

Nightly Quartz job: compute bucket checksums, compare against the peer's (or against our own
last-delivered `payload_hash` set if the peer has not implemented it), emit
`integration_reconciliation_divergent_total`, write divergent ids to a report table.

### 9.5 Runbook

| Symptom | First check | Action |
|---|---|---|
| `pending_age` climbing | Is the dispatcher running? Breaker state? | Check subscriber health, peer status page |
| All deliveries 401 | Token acquisition logs | Rotate/renew credentials; IdP reachable? |
| All deliveries 422 | Recent deploy? Vendored schema diff? | Compare payload vs schema, roll back mapper, dead-letter and replay |
| One aggregate always fails | Dead-letter row payload | Data-quality issue — fix source data, then replay |
| Consumer says "missing data" | Their cursor vs our retention | Under 30 days: replay from cursor. Over: backfill |
| Feed returns 410 | Cursor older than retention | Start a backfill run |
| Dead-letter queue growing | `failure_kind` breakdown | Validation ⇒ mapper bug; Permanent4xx ⇒ contract break |
| Worker holding stale locks | `locked_until < now()` on `InFlight` | Reaper job resets them; investigate the pod |

**Replay procedure:** `POST /integration/v1/admin/deadletters/replay` with ids or a filter →
resets to `Pending`, `attempt = 0`. Requires `admin` scope; every replay is audit-logged.

---

## 10. Testing

| Level | Tool | Coverage |
|---|---|---|
| Unit | xUnit + NSubstitute | mappers, enum tables, envelope building, error classification, backoff schedule |
| Foo-based | Bogus / AutoFixture | round-trip losslessness, boundary values, non-ASCII, DST edges |
| Coverage | reflection | every domain field mapped or excluded (ADR-0004 §5b) |
| Architecture | NetArchTest | Domain/Application do not reference Integration or `System.Net.Http` |
| Contract | JsonSchema.Net | generated payloads validate against the vendored peer schema |
| Integration | Testcontainers.PostgreSql | interceptor writes in the same transaction; `SKIP LOCKED` under concurrency; compaction; **`xid8` watermark under a deliberately slow concurrent transaction** |
| Integration | WireMock.Net | retry/backoff, breaker, 4xx vs 5xx classification, idempotency-key stability across retries |
| Failure injection | Testcontainers + kill | dispatcher killed mid-flight ⇒ redelivered, never lost; DB down ⇒ no partial state |
| End-to-end | full stack | domain write → peer receives correct contract; peer POST → domain updated |
| Load | NBomber / k6 | backfill at target rate; queue drain time; feed pagination over 1 M rows |

Two tests deserve to be written first because they encode the two central claims:

1. **The interceptor test** — assert that rolling back the business transaction leaves *no*
   change-log row, and committing leaves exactly one. This is "no dual write" as an executable
   statement.
2. **The watermark test** — open a slow transaction that inserts sequence N, insert and commit
   N+1 in another, assert the feed returns **neither** until the slow one commits, then both.
   This is the bug that is otherwise found in production six months later.

---

## 11. Performance and capacity

### 11.1 Expected load

At ~10 k changes/day (~0.12/s average, assume 20× peak = 2.4/s): entirely unremarkable for
PostgreSQL. The design's cost centres are the **backfill** (500 k aggregates) and the
**materialisation read**, not steady state.

### 11.2 Sizing

- `outbox_message` at ~4 KB/row × 10 k/day × 30 days ≈ **1.2 GB** retained. Comfortable.
- Backfill at 50 msg/s ⇒ 500 k aggregates ≈ **2.8 hours**. Raise the rate limit if the peer
  can take it; the throttle exists to protect them, not us.
- Dispatcher batch 100, 4 concurrent requests/subscriber ⇒ ~400 msg/s ceiling per subscriber,
  ~30× headroom over peak.

### 11.3 Scaling levers, in the order to pull them

1. Raise dispatcher batch size and per-subscriber concurrency (config).
2. Add API replicas — `SKIP LOCKED` scales workers linearly with no coordination.
3. Split workers into their own deployment (`WORKERS_ENABLED=true`, `API_ENABLED=false`) so
   dispatch load cannot affect API latency.
4. Move feed/backfill/checksum reads to a **read replica**.
5. Partition `outbox_message` monthly.
6. Only then: consider CDC or a broker.

### 11.4 Impact on the write path

The added cost per business transaction is **one small insert per changed aggregate**
(~200 bytes, no serialisation) — sub-millisecond, inside a transaction that is already open.
Materialisation, mapping and HTTP all happen off the request path. This is the central
performance argument for the two-stage design.

---

## 12. Rollout

The staging decision, its seven stages with exit criteria, the alternatives weighed, and the cost
accepted live in **[ADR-0007](adr/ADR-0007-staged-delivery-and-the-first-increment.md)**. Task
detail per stage is in [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md). Not restated here — two
copies of a rollout order drift within a sprint.

In summary: stage 1 is an on-demand, contract-shaped **read API** over the domain with no new
tables, so contact with the peer's contract happens first rather than behind five days of
infrastructure. Change capture is stage 2, the incremental feed stage 3, push delivery stage 4,
inbound stage 5, backfill runs and reconciliation stage 6, hardening stage 7.

What this section owns is the mechanism that makes any stage reversible.

Feature flags: `Integration:ReadApiEnabled`, `Integration:CaptureEnabled`,
`Integration:MaterializationEnabled`, `Integration:DispatchEnabled`, `Integration:FeedEnabled`,
`Integration:InboundEnabled`, plus per-subscriber `enabled`. **Every stage is independently
reversible without a deploy**, and stages 2 and 3 accumulate and validate real production payloads
before anything is delivered — which is how contract surprises get found on our terms rather than
during a joint go-live call.

---

## 13. Risks

| # | Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|---|
| R1 | Peer schema not available / changes late | Blocks all mapping work | **High** | Q3 escalated now; build everything else against a stub contract; vendor + daily diff job |
| R2 | Peer mandates delta payloads | +1 sprint, ordering machinery | Medium | ADR-0004 §3 specifies the delta variant; envelope already supports it |
| R3 | Bulk SQL bypasses change capture | Silent data gaps | Medium | Analyzer ban + explicit attribute + nightly reconciliation |
| R4 | PostgreSQL < 13 in production | `xid8` watermark unavailable | Low | **Verify in week 1**; fallback: lag the feed by a fixed safety window (weaker) |
| R5 | Feed cursor bug ships unnoticed | Silent consumer data loss | Medium | The watermark test in §10 is a release gate |
| R6 | Auth mechanism undecided at implementation time | Ingress cannot ship | Medium | ADR-0005 fallback ladder; build behind an `IIntegrationAuthenticator` seam |
| R7 | PII sent without legal basis | Compliance incident | Low | Redact by default; `pii_granted` opt-in; Q6 |
| R8 | Backfill saturates the peer | Peer outage caused by us | Medium | Consumer-driven pull; token-bucket throttle; priority separation |
| R9 | Dead letters accumulate unnoticed | Silent divergence | Medium | Alert on first dead letter; nightly reconciliation |
| R10 | Outbox table growth | Disk/performance | Low | Retention + partitioning; monitored |
| R11 | Domain refactor breaks mapping silently | Wrong data at peer | Medium | Coverage test names the field; round-trip tests |
| R12 | Kafka becomes available mid-project | Rework fear | Low | Only the dispatcher transport changes — argued in ADR-0001 |

---

## 14. Decisions requested

1. **Approve the architecture** (ADR-0001 … 0006).
2. **Approve the staging** (ADR-0007) — in particular that stage 1 is a read-only API and that
   the "no lost changes" guarantee therefore arrives at stage 3, not stage 2.
3. **Answer Q1–Q8 and Q17** (§2.3) — Q3 (peer schema), Q2 (auth) and Q17 (pilot consumer) are
   stage 1 prerequisites; Q5 (PostgreSQL version) is needed before stage 3.
4. **Nominate the peer-team counterpart** who owns the contract and can resolve the gaps in
   MAPPING_MATRIX.md §7. Stage 1 cannot exit without them.
