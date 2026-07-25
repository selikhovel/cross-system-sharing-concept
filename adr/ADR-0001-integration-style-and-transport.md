# ADR-0001 — Integration style and transport for external service exchange

- **Status:** Proposed
- **Date:** 2026-07-25
- **Deciders:** ProReel Estate backend team, Integration architecture
- **Supersedes:** —
- **Related:** ADR-0002, ADR-0003, ADR-0004, ADR-0005, ADR-0006

## Context

ProReel Estate is a domain-driven ASP.NET Core service on PostgreSQL. Its aggregates
(`Property`, `Building`, `Location`, and related entities) must be exchanged **bidirectionally**
with other internal company services:

- **Outbound:** other services need to observe changes to our catalogue.
- **Inbound:** other services push or expose data we must absorb into our own model.

Constraints established with the team:

| Constraint | Value |
|---|---|
| Message broker | **Kafka is not available** and will not be installed in the near term |
| Available transport | HTTP inside the corporate network |
| Contract ownership | **The external system dictates the wire format** (ADR-0004) |
| Delivery model | **Hybrid: we push, and consumers may also pull** |
| Domain model | **Must not be modified** to serve integration concerns |
| Volume | Moderate ongoing traffic **plus a one-off full-catalogue backfill** (ADR-0006) |
| Consistency need | No data loss; every domain field must be accounted for |

Without a broker, the reliability guarantees Kafka would normally provide (durable log,
replay, consumer offsets, at-least-once delivery) have to be reproduced by us. The
temptation is to call the peer service's HTTP API directly from the command handler
after `SaveChangesAsync`. That is a **dual-write**: the database commit and the HTTP call
are not atomic, so a crash, a timeout, or a 503 between them silently loses the change
forever. This is the single failure mode that must be designed out.

## Decision

We adopt an **asynchronous, durable, HTTP-based integration built on the Transactional
Outbox and Inbox patterns**, with a hybrid push/pull delivery model.

### 1. Outbound: outbox + dispatcher + feed API

```
Domain change (business transaction)
        │  same DB transaction
        ▼
 integration.outbox_change_log        ← change capture (ADR-0002)
        │  async
        ▼
 Materializer  ──►  integration.outbox_message   (external contract JSON, ADR-0004)
        │
        ├──►  integration.outbox_delivery (per subscriber fan-out)
        │             │
        │             ▼
        │      Dispatcher worker ──HTTP POST──► subscriber endpoint   [PUSH]
        │
        └──►  GET /integration/v1/{resource}/changes?cursor=…         [PULL]
```

- **Push** is the primary path: a `BackgroundService` claims pending deliveries with
  `FOR UPDATE … SKIP LOCKED` and POSTs them to each subscriber, with retry/backoff and a
  dead-letter table.
- **Pull** is served from the *same* `outbox_message` table as a cursor-paginated change
  feed. It is not a second source of truth — it is a second read path over one log.

The pull feed exists for three reasons that push alone cannot cover:

1. **Recovery.** A consumer that was down for a day, or that lost its own database, can
   re-read from a cursor without us re-driving anything.
2. **Reconciliation.** Detecting silent divergence requires the consumer to be able to
   walk our log independently.
3. **Backfill.** The initial migration (ADR-0006) is naturally a paged pull, not a
   million POST requests.

### 2. Inbound: ingress + inbox + anti-corruption layer

```
 Peer service ──HTTP POST──►  /integration/v1/inbound/{type}     [PUSH to us]
 Peer service  ◄──HTTP GET──  Inbound poller worker              [PULL by us]
        │
        ▼
 integration.inbox_message  (raw payload, dedup key, status)
        │  async
        ▼
 Translator (ACL)  ──►  Application command  ──►  Domain aggregate
```

The ingress endpoint does **only** authentication, schema validation, and an insert into
`inbox_message`. It never touches the domain synchronously. This keeps the endpoint fast,
makes it trivially idempotent on the peer's `messageId`, and means a bug in our mapping
logic returns a 202 and a retryable inbox row instead of a 500 that makes the peer
believe delivery failed.

### 3. What we explicitly do not do

- **No dual writes.** No `HttpClient` call inside a command handler or a `DbContext`
  transaction, ever.
- **No distributed transactions / two-phase commit.**
- **No shared database** with peer services.
- **No synchronous coupling on the write path.** A peer being down must never fail a
  ProReel Estate business operation.
- **Synchronous query calls to peer APIs are allowed** for read-side enrichment
  (e.g. resolving a reference), but only behind a typed client with timeout, retry,
  circuit breaker, and a defined fallback — never inside a write transaction.

## Alternatives considered

### A. Direct synchronous HTTP calls from command handlers

*Rejected.* Dual-write problem: state committed but message lost, or message sent but
transaction rolled back. Couples our write availability to every peer's uptime. Turns a
1-peer outage into a ProReel Estate outage. It is also unfixable incrementally — retrofitting
reliability later means re-touching every handler.

### B. Debezium / PostgreSQL logical decoding (CDC)

*Rejected for now.* CDC would give a genuinely zero-touch capture from the WAL, which is
attractive. But: Debezium's standard deployment target is Kafka Connect (unavailable);
Debezium Server can sink to HTTP but is another runtime to operate, and the company has no
existing CDC platform. CDC also emits *table-row* changes, so we would still need the whole
materialisation and mapping layer described here — the saving is only in the capture step,
which ADR-0002 solves in ~120 lines of C#. Worth revisiting if the outbox tables ever
become a write-throughput bottleneck.

### C. Wait for Kafka / install RabbitMQ

*Rejected.* Kafka is blocked at the company level, not by us. Introducing RabbitMQ as a
shadow broker would require the same platform approval Kafka failed to get, and would
*still* need an outbox on our side to avoid the dual-write between Postgres and the broker.
The outbox is therefore not wasted work: **if a broker arrives later, only the dispatcher's
transport changes — the outbox, the contract mapping, and the inbox all stay**. This is
explicitly designed as a broker-ready architecture.

### D. Scheduled batch export (nightly file / full dump)

*Rejected as the primary mechanism.* Latency measured in hours, no per-change semantics,
and reconciling a full dump is more expensive than streaming deltas. Retained only as a
degenerate case of the backfill mode (ADR-0006).

### E. Push-only (no feed endpoint)

*Rejected.* Leaves no recovery path for a consumer that lost state, forces us to build
a bespoke "replay to subscriber X" admin tool, and makes reconciliation impossible for the
consumer to self-serve. The feed is a thin read over data we are already storing.

### F. Pull-only (no push)

*Rejected as the sole mechanism.* Acceptable latency only with aggressive polling by every
consumer, which multiplies load on our database by the number of consumers, and gives the
consumer no signal that something changed. Kept as the secondary path.

## Consequences

### Positive

- **No lost changes.** The message becomes durable in the same transaction as the business
  state; delivery is retried until it succeeds or is explicitly dead-lettered.
- **Peer outages are absorbed**, not propagated. Our write path never depends on a peer.
- **One log, two read paths.** Push and pull cannot diverge because they read the same rows.
- **Broker-ready.** Swapping HTTP push for Kafka later is a change to one worker class.
- **Auditable.** The outbox is a complete, queryable record of everything we ever emitted,
  which is the artefact you want the first time a peer says "we never got that update".
- **Testable end to end** with Testcontainers + WireMock, no broker infrastructure.

### Negative / costs accepted

- **At-least-once, not exactly-once.** Consumers must be idempotent. We make this cheap by
  emitting a stable `messageId` and an `aggregateVersion` (ADR-0004) — but it is a hard
  requirement we must document in the consumer-facing contract.
- **Eventual consistency.** Peers see our state after a delay (target: p95 < 10 s under
  normal load). Any peer requiring read-your-write consistency must call our synchronous
  query API instead.
- **Operational surface.** New tables, workers, metrics, alerts and a runbook to own
  (see the technical proposal §9).
- **Database load.** Outbox insert on every tracked write, plus dispatcher polling. Mitigated
  by narrow capture rows (ADR-0002), partial indexes, batch claiming, and monthly partition
  archival.
- **We are re-implementing a slice of a broker.** Accepted deliberately; the slice is small
  and well understood, and the alternative is an unavailable platform dependency.

## Compliance / verification

- Architecture test (`NetArchTest`): no type in `ProReelEstate.Domain` or
  `ProReelEstate.Application` may reference `System.Net.Http` or the integration assembly.
- Architecture test: no `IHttpClientFactory` consumer may be resolved inside a class
  implementing `ICommandHandler<,>`.
- Integration test: kill the dispatcher mid-flight, restart, assert the message is delivered
  exactly once *as observed by the consumer's idempotency key*, and never lost.
