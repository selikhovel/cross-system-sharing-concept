# Acme — External Service Integration

Design documentation for exchanging domain data (`Foo`, `Bar`, `Baz`) with
other internal company services over HTTP, without a message broker and without modifying the
domain model.

## This design is domain-agnostic

**Nothing here depends on what the data means.** The pattern — transactional outbox, deferred
materialisation, an anti-corruption layer, a cursor feed, an inbox — is driven by the *shape* of the
problem, not by the subject matter:

- aggregates change inside database transactions,
- someone outside the service needs to observe those changes,
- the wire format is owned by someone else,
- and no broker is available.

Any domain that fits those four sentences fits this design unchanged. Orders and shipments,
patients and encounters, accounts and ledgers, devices and telemetry, tickets and assignments — the
aggregate names change and nothing else does.

**Every name in these documents is a placeholder.** `Acme` is the service; `Foo` is the aggregate
that gets published; `Bar` and `Baz` are aggregates it references, present so that the fan-out and
flattening cases have somewhere to live; `peer-a`/`peer-b`/`peer-c` are external services.
Substitute your own and read on. The full table is in [CLAUDE.md](CLAUDE.md#naming--all-names-are-placeholders).

Value-level concepts — money, quantities with units, date-only versus timestamp, coordinates,
contact details — are kept concrete deliberately. They are cross-domain, and they carry the failure
modes this design exists to eliminate: precision loss, unit ambiguity as a silent 10× error, a
timezone shifting a date across a day boundary, truncation as a data-quality incident, personal data
leaving without a legal basis.

## Documents

| Document | What it answers |
|---|---|
| [TECHNICAL_PROPOSAL.md](TECHNICAL_PROPOSAL.md) | The full design: architecture, DDL, pipelines, security, operations, testing, capacity, rollout, risks |
| [MAPPING_MATRIX.md](MAPPING_MATRIX.md) | Field-by-field mapping specification and the losslessness checklist. **Fill in from the peer's schema before writing mappers** |
| [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) | Task detail per stage, with stage 1 broken down by day |
| [STAGE_1_READ_API.md](STAGE_1_READ_API.md) | **Start here to write code.** Stage 1 in full: what to check in an existing Clean Architecture / CQRS / EF Core service, where each file goes, the code, the tests, and the EF-specific traps |
| [BACKLOG.md](BACKLOG.md) | **Start here to plan.** The whole integration as epics → features → stories → tasks, with sizing, traceability to requirements, and which open questions block which item |
| [STAGES_DECK.md](STAGES_DECK.md) | **Start here to brief someone.** All seven stages with Mermaid architecture diagrams: goal, what a consumer can newly do, what the ordering costs, and when each guarantee arrives. Derived from ADR-0007, which stays authoritative |
| [ROADMAP.md](ROADMAP.md) | **Start here to schedule.** The same seven stages as a timeline, a gate graph and a waiting-on-whom map, plus stage 1 down to the commit. Effort and calendar reported separately |
| [openspec/](openspec/project.md) | The design restated as verifiable requirements, plus a [critical review](openspec/changes/add-cross-system-integration-layer/design.md) listing 14 defects found in the documents above |
| [ADR-0001](adr/ADR-0001-integration-style-and-transport.md) | Why outbox + hybrid push/pull HTTP instead of direct calls, CDC, or waiting for Kafka |
| [ADR-0002](adr/ADR-0002-change-capture-without-touching-the-domain.md) | How changes are captured without adding anything to the domain model |
| [ADR-0003](adr/ADR-0003-outbox-inbox-storage-and-delivery-semantics.md) | Storage, `SKIP LOCKED` claiming, at-least-once, compaction, the `xid8` feed watermark, retry/DLQ, retention |
| [ADR-0004](adr/ADR-0004-contract-mapping-payload-and-versioning.md) | Anti-Corruption Layer, snapshot vs delta payloads, versioning, and how losslessness is enforced by the build |
| [ADR-0005](adr/ADR-0005-authentication-and-transport-security.md) | Service-to-service auth (recommendation + fallback ladder), PII, SSRF, replay |
| [ADR-0006](adr/ADR-0006-initial-backfill-and-reconciliation.md) | Go-live backfill and ongoing divergence detection |
| [ADR-0007](adr/ADR-0007-staged-delivery-and-the-first-increment.md) | Seven delivery stages, why stage 1 is a read-only API, and what that ordering costs |

## Where to start

Read [ADR-0007](adr/ADR-0007-staged-delivery-and-the-first-increment.md) first. It says what the
first increment is — an on-demand, contract-shaped `GET` over `Foo`, with **no new tables and
no change to the write path** — and what building it first costs. Then
[STAGE_1_READ_API.md](STAGE_1_READ_API.md) is the implementation guide for that increment, starting
with what to verify in the existing service before writing a line. Everything below describes the
target the seven stages converge on.

## The design in six sentences

Domain changes are captured by an EF Core `SaveChangesInterceptor` into a small
`outbox_change_log` row **inside the same transaction** as the business write, so a change can
never be committed without its integration record. A background materializer later loads the
aggregate, maps it through an Anti-Corruption Layer into the **peer's** contract format, and
writes a finished payload to `outbox_message`. Delivery happens two ways over the same log: a
dispatcher **pushes** to each subscriber with retry, circuit breaking and dead-lettering, and
consumers can **pull** a cursor-paginated change feed for recovery and reconciliation. Inbound
messages land in an `inbox_message` table for deduplication and are applied asynchronously
through existing application command handlers, so external data passes every domain invariant.
The domain model is never modified, and "every field is mapped" is enforced by compiler
diagnostics plus a reflection test that names any field you forget. When Kafka eventually
arrives, only the dispatcher's transport changes.

## Status

**Proposed.** Twelve open questions are listed in TECHNICAL_PROPOSAL.md §2.3, with five more answered
and folded into the ADRs. **Three touch stage 1:** which revision of the peer's schema and when it
freezes (Q3), the auth mechanism (Q2), and the legal basis for the personal data the peer is entitled
to receive (Q6).

The exchange is **one-to-one** — exactly one consumer (Q17) — which defers three of the fourteen
defects, and production runs **PostgreSQL 17** (Q5), which closes risk R4.

Fourteen defects found while deriving the OpenSpec requirements are open in
[design.md](openspec/changes/add-cross-system-integration-layer/design.md); ADR-0007 records which
of them gate which stage. **None of them gate stage 1** — it is scoped to avoid all of them.
