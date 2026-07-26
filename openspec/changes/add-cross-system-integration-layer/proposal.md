# Change: add-cross-system-integration-layer

> **Scope note.** Stage 1 of [ADR-0007](../../../adr/ADR-0007-staged-delivery-and-the-first-increment.md)
> was split out into [`add-foo-read-api`](../add-foo-read-api/proposal.md) — a read-only
> API with no new tables, scoped so that none of the defects below gate it. This change now covers
> **stages 2–7**: change capture, materialisation, the incremental feed, push delivery, inbound,
> backfill and reconciliation, and hardening. Its `contract-mapping` requirements are partially
> satisfied by stage 1, and its `change-feed` capability supersedes stage 1's paging in stage 3.

## Why

Acme must exchange `Foo`, `Bar` and `Baz` data bidirectionally with
other internal services over HTTP, with no broker available, a wire format the peer owns, and a
domain model that must not be modified. The design for this exists as six ADRs plus a technical
proposal, but it exists only as prose: "the feed must not skip rows", "no field is silently
lost", "backfill cannot starve live traffic" are argued in paragraphs, not stated as verifiable
requirements an implementer can check off or a reviewer can test against.

This change converts the accepted design into requirements with scenarios, and — in the course
of doing so — records the twelve places where the documents contradict each other or specify
behaviour that cannot be implemented as written (see `design.md`).

## What Changes

- **Adds** nine capability specs derived from the existing documents:
  `change-capture`, `outbox-materialization`, `outbox-delivery`, `change-feed`,
  `inbound-ingestion`, `contract-mapping`, `integration-security`,
  `backfill-reconciliation`, `integration-operations`.
- **States as requirements** the four load-bearing guarantees the design claims: transactional
  capture with no dual write, a cursor feed that cannot skip rows (`xid8` watermark),
  build-enforced mapping losslessness, and backfill that cannot starve live delivery.
- **Pins the corrected behaviour** for twelve defects found while deriving the specs. Each is
  recorded in `design.md` with severity and a recommended resolution; each has a requirement
  that states the behaviour the implementation must exhibit. Four are blocking:
  - `aggregateVersion` as defined (the outbox `sequence`) is unavailable at envelope-build time
    and undefined for backfill snapshot pages — the idempotency contract has no value at
    go-live, which is the one moment it must hold (**D1**, **D2**).
  - The dispatcher claim SQL does not order by `priority`, so a 500 k-row backfill drains ahead
    of live changes — the starvation ADR-0006 §3 promises is impossible actually happens
    (**D3**).
  - Echo suppression depends on a `causationSource` that exists in no table and in no envelope
    field, and direct-source suppression cannot break a three-system loop (**D6**).
  - PII redaction and subscriber-scoped filters are per-subscriber, but `outbox_message` stores
    one payload and one hash per contract version — redacted subscribers diverge by
    construction and every checksum comparison for them is a false positive (**D7**, **D8**).
- **Does not modify** any ADR, `TECHNICAL_PROPOSAL.md`, `MAPPING_MATRIX.md` or
  `IMPLEMENTATION_PLAN.md`. ADRs are the decision record; the defects are raised here as open
  questions and resolved by amending or superseding the source documents, not by editing them
  from a spec.

## Impact

| Area | Effect |
|---|---|
| `openspec/specs/` | Nine new capabilities on archive; none exist today |
| `adr/ADR-0002` | D6 (capture provenance), D9 (stage-1 retry policy) need an amendment or a superseding ADR |
| `adr/ADR-0003` | D3 (priority in the claim SQL), D4 (compaction column name), D5 (materialiser watermark rationale), D10 (retention vs DDL) |
| `adr/ADR-0004` | D1/D2 (`aggregateVersion` source), D6 (envelope provenance field), D11 (tombstone schema) |
| `adr/ADR-0006` | D2 (snapshot page versions), D8 (subscriber-scoped checksums) |
| `TECHNICAL_PROPOSAL.md` | §2.3 gains open questions Q9–Q16; §4 DDL gains columns for D1, D6, D7, D9, D12 |
| `MAPPING_MATRIX.md` | §5 filter rules need durable per-subscriber scope state (D12); §7 gains the tombstone-schema gap (D11) |
| New document | A **consumer-facing contract** is referenced by ADR-0003 §3 and ADR-0006 §5 but does not exist (D13) |
| Code (other repository) | `Acme.Integration`, `Acme.Integration.Contracts`, `Acme.Integration.Tests`; interceptor registration and endpoint mapping in existing projects |
