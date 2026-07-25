# ProReel Estate — External Service Integration

Design documentation for exchanging domain data (`Property`, `Building`, `Location`) with
other internal company services over HTTP, without a message broker and without modifying the
domain model.

## Documents

| Document | What it answers |
|---|---|
| [TECHNICAL_PROPOSAL.md](TECHNICAL_PROPOSAL.md) | The full design: architecture, DDL, pipelines, security, operations, testing, capacity, rollout, risks |
| [MAPPING_MATRIX.md](MAPPING_MATRIX.md) | Field-by-field mapping specification and the losslessness checklist. **Fill in from the peer's schema before writing mappers** |
| [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) | Day-by-day build order, starting with what to do today |
| [ADR-0001](adr/ADR-0001-integration-style-and-transport.md) | Why outbox + hybrid push/pull HTTP instead of direct calls, CDC, or waiting for Kafka |
| [ADR-0002](adr/ADR-0002-change-capture-without-touching-the-domain.md) | How changes are captured without adding anything to the domain model |
| [ADR-0003](adr/ADR-0003-outbox-inbox-storage-and-delivery-semantics.md) | Storage, `SKIP LOCKED` claiming, at-least-once, compaction, the `xid8` feed watermark, retry/DLQ, retention |
| [ADR-0004](adr/ADR-0004-contract-mapping-payload-and-versioning.md) | Anti-Corruption Layer, snapshot vs delta payloads, versioning, and how losslessness is enforced by the build |
| [ADR-0005](adr/ADR-0005-authentication-and-transport-security.md) | Service-to-service auth (recommendation + fallback ladder), PII, SSRF, replay |
| [ADR-0006](adr/ADR-0006-initial-backfill-and-reconciliation.md) | Go-live backfill and ongoing divergence detection |

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

**Proposed.** Eight open questions are listed in TECHNICAL_PROPOSAL.md §2.3; two of them —
the peer's OpenAPI schema and the production PostgreSQL version — should be resolved before
implementation starts. The rest of the plan is not blocked on them.
