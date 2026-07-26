# Change: add-foo-read-api

Stage 1 of [ADR-0007](../../../adr/ADR-0007-staged-delivery-and-the-first-increment.md).
Implementation detail — discovery checklist, file layout, code, tests, EF traps — is
[STAGE_1_READ_API.md](../../../STAGE_1_READ_API.md).

## Why

Nothing in the integration design is consumable until a peer can look at a payload. The blocking
item is the peer's contract (Q3, risk R1) — the only part we cannot unblock ourselves — and both
prior orderings scheduled it behind five days of infrastructure that cannot validate a single field
name.

The target architecture already contains the answer: ADR-0006 §2's snapshot pages are
"materialised on demand from current domain state and are not written to the outbox at all". That
endpoint needs none of the outbox, none of the capture machinery and no new table. Built first, it
turns the eight unresolved gaps in `MAPPING_MATRIX.md` §7 into answers in days, and it is not
throwaway — stage 3 adds a pinned watermark and it becomes the snapshot endpoint verbatim.

## What Changes

- **Adds** one capability, `foo-read-api`: `GET /integration/v1/foos/{id}` and
  `GET /integration/v1/foos?cursor=&limit=`, serving the peer's contract shape built on
  demand from current domain state, keyset-paged, authenticated and rate-limited.
- **Adds** the anti-corruption layer's permanent parts at their cheapest moment: the flat snapshot
  read model, the compile-time mapper with unmapped members as build errors, the reflection
  field-coverage test, the round-trip property tests and the architecture tests.
- **Declares replace-all semantics.** Stage 1 emits no `aggregateVersion`, because a version needs a
  write-path hook and stage 1 has no incremental consumer to need one. Stage 2 adds the field
  additively (ADR-0004 §4). A version improvised from a timestamp is rejected — ADR-0007
  alternative (E).
- **Redacts personal data unconditionally**, per the §2.3 Q6 default. There is no subscriber table
  in stage 1, so there is nothing to grant against — which also means defect **D7** cannot bind
  here.
- **Adds** `CONSUMER_CONTRACT.md`, resolving defect **D13** at the first moment we make a promise to
  a consumer.
- **Excludes**, deliberately: change capture, the outbox, any worker, push delivery, change
  notification, deletions, inbound, the subscriber table, retention and partitioning. Each belongs
  to a later stage, and stage 1 is scoped so that none of the fourteen defects in
  `add-cross-system-integration-layer/design.md` gate it.

## Impact

| Area | Effect |
|---|---|
| `openspec/specs/` | One new capability, `foo-read-api`, on archive |
| `add-cross-system-integration-layer` | Now covers stages 2–7; its `contract-mapping` requirements are partially satisfied here and its `change-feed` capability supersedes this one's paging in stage 3 |
| `adr/ADR-0007` | This change is its stage 1 |
| `IMPLEMENTATION_PLAN.md` | Stage 1, days 1–5 |
| New document | `CONSUMER_CONTRACT.md` (**D13**) |
| Code (other repository) | New: `Acme.Integration`, `Acme.Integration.Contracts`, `Acme.Integration.Tests`. Modified: `Acme.Api` route registration only |
| Database | **None.** No migration, no new table, no column, no interceptor |
