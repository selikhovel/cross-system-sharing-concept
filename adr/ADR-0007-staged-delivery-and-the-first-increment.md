# ADR-0007 — Staged delivery, and what the first increment is

- **Status:** Proposed
- **Date:** 2026-07-26
- **Related:** ADR-0001 (integration style), ADR-0002 (capture), ADR-0003 (storage and delivery),
  ADR-0004 (contract and mapping), ADR-0006 (backfill) · revises the phasing in
  TECHNICAL_PROPOSAL §12
- **Open question:** Q17 — does a pilot consumer exist who can integrate against a read-only API
  in stage 1? If nobody consumes stage 1, its value is only the contract validation, and the
  order below is worth re-examining.

## Context

ADR-0001 … ADR-0006 describe the target architecture. They do not say what to build **first**, and
the two documents that try to answer that disagree in spirit:

- `TECHNICAL_PROPOSAL.md` §12 phases the rollout as *capture → materialise → feed → push →
  backfill → inbound*. Phases 0 and 1 are deliberate dark launches: two phases with no
  consumer-visible result.
- `IMPLEMENTATION_PLAN.md` orders the work day by day starting from the interceptor, on the
  argument that transactional capture is the load-bearing part and everything else is plumbing
  on top of it.

Both are defensible and both share one property worth challenging: **contact with the peer's
contract happens late.** The proposal's own risk register puts the peer's schema at
`Likelihood: High / Impact: blocks all mapping work` (R1) and §2.3 calls Q3 "the true critical
path". The plan agrees, then schedules the mapping work in week 2 — behind five days of
infrastructure that cannot validate a single field name.

Meanwhile the first consumer-visible artefact under §12 arrives in phase 2, after the outbox, the
materialiser, the change feed and the visibility watermark are all working. That is a lot of
machinery to build before anyone outside the team can look at a payload and say "that is not the
field we meant".

There is also an ordering fact hiding in ADR-0006 §2: backfill snapshot pages are
"materialised **on demand** from current domain state and are not written to the outbox at all".
Stated plainly — the target architecture **already contains a synchronous, on-demand,
contract-shaped read endpoint over the domain**, and it needs none of the outbox to work.

## Decision

Deliver in **seven stages**, each independently shippable, each with an exit criterion. Stage 1 is
the on-demand read API described above.

### Stage 1 — Contract-shaped read API

`GET /integration/v1/foos/{id}` and `GET /integration/v1/foos?cursor=&limit=`, serving
the peer's contract shape, built on demand from current domain state, keyset-paged.

| Included | Excluded |
|---|---|
| Integration project + contracts project | **No new tables. No migration.** |
| Flat `FooSnapshot` read model + projection query | No change capture, no interceptor |
| `Snapshot → Contract` mapper (the ACL) with unmapped members as build errors | No workers |
| The page envelope of ADR-0006 §2 (`items`, `nextCursor`, `hasMore`) | No push, no change notification, no deletes |
| Keyset pagination, capped page size | No inbound |
| Authentication, one scope, per-caller rate limit, page cap | No subscriber table |
| **PII redacted unconditionally** (§2.3 Q6 default) | No per-subscriber PII grant |
| Architecture tests, field-coverage test, round-trip tests, schema validation | No outbox retention, no partitioning |

Two consequences of that table are the point of the stage:

1. **It is not throwaway.** Stage 1 *is* ADR-0006 §2's snapshot endpoint, minus the pinned
   watermark that stage 3 adds. The projection, the mapper, the page contract and the tests all
   survive unchanged into the target architecture.
2. **It carries no `aggregateVersion`.** A version needs a write-path hook (ADR-0004 §5.3 option 2,
   defect D1), and stage 1 has no incremental consumer to need one. Stage 1 therefore declares
   **replace-all semantics** explicitly in the consumer contract, and stage 2 adds the field —
   an additive change requiring no version bump (ADR-0004 §4).

**Exit criteria:** a consumer reconstructs the full `Foo` catalogue from the paged endpoint;
`MAPPING_MATRIX.md` is filled in for `Foo` with zero unresolved `GAP-*` rows; the coverage
and round-trip tests are green; no field marked `pii` appears in any response.

### Stages 2–7

| Stage | Ships | First consumer-visible gain | Exit criterion |
|---|---|---|---|
| **2** — Change capture, dark | `integration` schema, `outbox_change_log`, the `SaveChangesInterceptor`, root-resolution map with startup validation, provenance columns, the per-aggregate version counter. **No delivery.** | `aggregateVersion` appears in stage 1's responses, so a consumer can start applying the version rule | Rollback leaves no capture row; capture rows correct in production; no measurable write-latency change |
| **3** — Incremental feed | `outbox_message`, the materialiser, envelope + payload hash, schema validation before the outbox, filter rules and tombstones, `GET …/changes?cursor=` with the `xid8` watermark, `410` on an expired cursor, change-log and message retention | Consumers stop re-reading the catalogue and sync incrementally; **deletions become visible** | The watermark test is green as a release gate; a pilot consumer syncs with no gap over a week |
| **4** — Push delivery | `subscriber`, `outbox_delivery`, the dispatcher with priority claiming and compaction, payload variants and per-subscriber PII grants, response classification, backoff, dead letters, breaker, lock reaper, per-`(subscriber, aggregate)` projection state, admin replay | Latency drops from a poll interval to seconds; consumers who will not build a poller are served | p95 delivery under 10 s; dead letters at zero; a 500 k-row backfill does not delay one live change |
| **5** — Inbound | Ingress endpoint, `inbox_message`, the inbox worker, inbound translators, failure taxonomy, echo suppression, `external_reference`, optional poller | Peer data lands in our domain through the existing command handlers | Peer messages applied; echo suppression verified across a three-system loop |
| **6** — Backfill runs and reconciliation | Pinned-watermark runs over stage 1's endpoint, `backfill_run`, push-mode backfill at the lowest priority with a token bucket, subscriber-scoped checksums, the version-pinned canonical serialiser, the nightly job | Divergence becomes observable daily instead of by a business user | Checksums converge; divergence zero for a week |
| **7** — Hardening and scale | Retention and partitioning per table, the bulk-mutation analyzer, failure injection, load tests, read-replica routing, worker split, runbook, game day | Nothing new; the thing stops needing heroics | On-call trained; alerts verified in a game day |

### Security is a floor, not a stage

Every stage that exposes an endpoint ships authenticated, scope-authorised and rate-limited from
day one (ADR-0005 §3's non-negotiables). Stage 1 needs one scope and the authenticator seam;
stage 4 adds outbound credential handling; stage 5 adds the per-endpoint client allowlist. There is
no stage at which an integration endpoint is unauthenticated because "it is only stage 1".

### The consumer contract grows with the stages

`CONSUMER_CONTRACT.md` is created **in stage 1** — stating replace-all semantics, the page
contract and the rate limit — and is extended in every subsequent stage. A guarantee that changes
without the consumer document changing in the same commit is a guarantee nobody outside the team
knows about.

### Defect gates

Each stage is gated on the defects that bind at that stage
(`openspec/changes/add-cross-system-integration-layer/design.md`):

| Stage | Must be resolved first |
|---|---|
| 1 | none — stage 1 is defined to avoid all fourteen |
| 2 | **D1** (version source), **D6** (provenance columns) |
| 3 | **D2** (version on snapshot pages), **D5** (watermark scope), **D9** (stage-1 retry), **D11** (deletion schema), **D14** (batch isolation) |
| 4 | **D3** (priority claim), **D4** (compaction predicate), **D7** (payload variants), **D12** (projection state), **S5** (delivered-version index) |
| 5 | **D6** (loop suppression, in full), **S1** (identity resolution) |
| 6 | **D8** (subscriber-scoped checksums) |
| 7 | **D10** (retention versus schema) |

`D13` (the consumer contract) is resolved in stage 1 and maintained thereafter.

## Alternatives considered

### A. Capture first, as TECHNICAL_PROPOSAL §12 and IMPLEMENTATION_PLAN order it

*Rejected as the opening move, retained as stage 2.* The argument for it is real: transactional
capture is the load-bearing decision, and a capture bug found late is expensive. But capture is
also the part with **no external dependency** — it can be built at any time, by us, without
waiting for anyone. The mapping layer is the opposite: it is blocked on another team, it is on the
critical path, and it is the only part that can be wrong in a way we cannot detect ourselves.
Building the unblockable part first while the blocked part waits maximises the time spent blocked.

Capture-first also spends two phases producing nothing anyone outside the team can evaluate, which
makes it hard to hold attention on an integration that has to be co-delivered with another team.

### B. Big bang — build the whole design, then integrate

*Rejected.* Every risk in §13 arrives simultaneously, at the point of maximum sunk cost, and the
first contract surprise lands during a joint go-live call. The proposal's own phasing argument
against this stands.

### C. Stage 1 as an internal debug endpoint, not a consumer-facing one

*Rejected.* An endpoint nobody consumes validates the mapper against our own opinion of the
contract, which is exactly what is already written down and exactly what might be wrong. The value
of stage 1 is a second party reading the payload. An internal endpoint costs nearly the same and
buys nothing.

### D. Stage 1 as a nightly file export

*Rejected.* Cheaper for us, worse for everyone: another format, another mapping path, another
security review (object-store credentials, ADR-0005), and a consumer integration that must then be
thrown away when the API arrives. ADR-0006's rejection of dump-based exchange applies here too.

### E. Stage 1 emitting `aggregateVersion` from a domain timestamp

*Rejected.* `UpdatedAtUtc` is monotonic per aggregate only while the clock is; a backwards NTP
correction makes an update look older than the state it replaces, and the consumer's version rule
then silently discards it. ADR-0003 §5 rejects timestamps as a feed cursor for related reasons.
Declaring replace-all semantics for one stage is cheaper and honest.

### F. Skip stage 1 and start at stage 3 (feed without push)

*Rejected as the first increment, though it is a coherent target.* It still requires the outbox,
the materialiser and the watermark before any payload is visible to a peer — the same "contract
contact last" problem as (A), with more machinery.

## Consequences

### Positive

- **The blocked work starts first.** The mapping matrix gets filled against real payloads and a
  real reader in stage 1, so the eight gaps in `MAPPING_MATRIX.md` §7 turn into answers weeks
  earlier than under §12's phasing.
- **Stage 1 is one project, one query, one mapper, no migration.** It can be reviewed in an
  afternoon and reverted by deleting a route.
- **Nothing in stage 1 is rework.** It becomes ADR-0006 §2's snapshot endpoint verbatim, and the
  mapper, the coverage test and the round-trip tests are the same artefacts the target needs.
- **Every stage has a consumer-visible gain**, which is what keeps a two-team integration moving.
- **Risk order is inverted correctly**: the risks we cannot control (R1 peer schema, R2 payload
  shape, R7 PII authorisation) are attacked before the risks we can (R3 bulk SQL, R5 cursor bug,
  R10 table growth).
- **The `xid8` prerequisite (Q5, R4) stops blocking the start.** Stage 1 needs no `xid8`; the
  version check must still happen before stage 3, but a wrong answer no longer stalls day one.

### Negative / costs accepted

- **"No lost changes" — the design's first-priority property — arrives at stage 3, not stage 2.**
  Between stage 1 and stage 3 a consumer holds a snapshot that goes stale silently, and there is no
  mechanism that would tell either side. This is the real price of this ordering and it must be in
  the consumer contract, in those words.
- **Stage 1 has no deletions.** A record removed from our catalogue simply stops appearing in
  pages; a consumer that does not full-replace keeps it forever. Full-replace semantics are
  therefore mandatory in stage 1, not advisory.
- **Read load lands on the primary, on demand and unthrottled by an outbox.** Mitigated by the page
  cap, the per-caller rate limit and a read replica where one exists — but a consumer polling
  aggressively in stage 1 is a load pattern the target architecture is specifically designed to
  avoid.
- **A pull-only endpoint can become the integration.** Once a consumer has a working poller there
  is little pressure to adopt the feed, and the outbox risks being built for nobody. Mitigated by
  documenting stage 1 as time-boxed in the consumer contract and by a rate limit that makes
  catalogue-scale polling uncomfortable — but this is a social risk, and social risks are the ones
  that actually materialise.
- **Two consumer-visible contract iterations** (stage 1 without `aggregateVersion`, stage 2 with).
  Additive and no version bump, but it is still a second conversation with the other team.
- **Stage 1 without a pilot consumer is worth much less** than stated above — hence Q17. If no peer
  can read it, alternative (A) becomes competitive again.

## Compliance / verification

- Stage 1 ships with the architecture test from ADR-0001 already asserting `Domain` and
  `Application` reference neither `Integration` nor `System.Net.Http` — trivially green at that
  point, which is when it is cheapest to add.
- Stage 1 ships with the field-coverage test of ADR-0004 §5(b). "Every field mapped or excluded
  with a reason" is enforced from the first increment, not retrofitted.
- A test asserts no field marked `pii` in `MAPPING_MATRIX.md` appears in any stage 1 response.
- Each stage's exit criterion above is a release gate, not a status report.
