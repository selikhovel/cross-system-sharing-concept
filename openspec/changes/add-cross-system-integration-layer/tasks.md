# Tasks

Milestone 0 is documentation work in **this** repository. Milestones 1–10 are implementation work in
the service's repository, ordered as `IMPLEMENTATION_PLAN.md` orders it, with the corrections from
`design.md` folded into the step where they apply.

> Stage 1 of ADR-0007 lives in [`add-foo-read-api`](../add-foo-read-api/tasks.md) and is
> not repeated here. Milestone 2 onward corresponds to ADR-0007 stages 2–7; the defect ids in
> parentheses match ADR-0007's defect-gate table, so a milestone cannot start before its gates are
> resolved in milestone 0.

## 0. Resolve the design defects before code depends on them

- [ ] 0.1 Amend ADR-0003 §2: `ORDER BY priority, next_attempt_at, id` in the claim SQL, and reconcile
      the claim index name and columns with `TECHNICAL_PROPOSAL.md` §4.3 (**D3**)
- [ ] 0.2 Amend ADR-0003 §4: fix the compaction predicate's column name and make `compactable`
      resolvable from `outbox_delivery` (**D4**)
- [ ] 0.3 Amend ADR-0003 §5: remove the watermark from the stage-1 claim and record why cursor-based
      and status-based claiming differ (**D5**)
- [ ] 0.4 Amend ADR-0004 §3 / `TECHNICAL_PROPOSAL.md` §5.3: make the per-aggregate counter the primary
      `aggregateVersion` source, and state how snapshot pages obtain a version (**D1**, **D2**)
- [ ] 0.5 Amend ADR-0002 and ADR-0004 §3: add capture provenance and the envelope propagation path;
      specify hash-based no-op suppression as the primary loop defence (**D6**)
- [ ] 0.6 Amend ADR-0005 §3 and `TECHNICAL_PROPOSAL.md` §4.2/§5.2: model `payload_variant` as a
      first-class dimension of a message and of the fan-out key (**D7**)
- [ ] 0.7 Amend ADR-0006 §4: scope the checksum endpoint to the authenticated subscriber's filter,
      contract version and variant (**D8**)
- [ ] 0.8 Amend ADR-0002 / ADR-0003 §6: give stage-1 failures `next_attempt_at`, the retry schedule, a
      dead-letter link, a metric and an alert; define or drop the `Skipped` status (**D9**, **D14**)
- [ ] 0.9 Amend ADR-0003 §8: state the phase-1 retention mechanism per table so it matches the DDL,
      with the partitioning trigger recorded (**D10**)
- [ ] 0.10 Add the deletion-representation gap to `MAPPING_MATRIX.md` §7 and specify schema selection
      by `changeKind` in ADR-0004 (**D11**)
- [ ] 0.11 Add durable per-`(subscriber, aggregate)` projection state to `TECHNICAL_PROPOSAL.md` §4 and
      decide `filter_expression`'s fate — specified grammar, or named code-defined filters (**D12**)
- [ ] 0.12 Write `CONSUMER_CONTRACT.md`, add its README row, and move every consumer obligation out of
      the ADRs into it (**D13**)
- [ ] 0.13 Add open questions Q9–Q16 to `TECHNICAL_PROPOSAL.md` §2.3 and the field-level gaps to
      `MAPPING_MATRIX.md` §7, each with an owner and a default
- [ ] 0.14 Correct `TECHNICAL_PROPOSAL.md` §6.1's identity resolution to use the authentication seam
      (**S1**); fix the delivered-version guard's index (**S5**); state the push-versus-pull stream
      difference (**S4**); price or defer `Event` stream locking (**S2**); split the two replay
      mechanisms in §9.5 (**S3**)
- [ ] 0.15 Re-report the estimate as engineering effort and calendar separately, with the peer
      round-trips on `MAPPING_MATRIX.md` §7 named as the critical path (**S7**)

## 1. Prerequisites

- [ ] 1.1 Confirm the production PostgreSQL major version is 13 or higher; if not, stop and re-plan
      the feed watermark (Q5, risk R4)
- [ ] 1.2 Confirm the .NET and EF Core versions actually in the solution
- [ ] 1.3 Confirm whether aggregates already raise domain events
- [ ] 1.4 Get row counts per integrated aggregate to size the backfill (Q8)
- [ ] 1.5 Request the peer's schema — the longest-lead item and the only true blocker (Q3)
- [ ] 1.6 Open tickets with owners for Q1–Q8 and Q9–Q16

## 2. Foundation and transactional capture

- [ ] 2.1 Create the integration, contracts and test projects; assert the reference direction with an
      architecture test while it is trivially green
- [ ] 2.2 Add packages, pinning the assertion library below its commercial major version
- [ ] 2.3 EF configurations and migration for the integration schema, unpartitioned, including the
      transaction-id column on the change log and the message table
- [ ] 2.4 Add the columns the corrections require: capture provenance, payload variant, stage-1
      `next_attempt_at`, per-`(subscriber, aggregate)` projection state
- [ ] 2.5 Aggregate root resolution map with startup validation that fails fast on an unregistered
      entity type
- [ ] 2.6 The capture interceptor, registered on the existing context; original values captured for
      deletes
- [ ] 2.7 Feature flag for capture, defaulting off in production
- [ ] 2.8 Tests: commit yields exactly one row; **rollback yields none**; five edits yield one row;
      startup fails on an unregistered type

## 3. Materialisation

- [ ] 3.1 Flat snapshot read models and non-tracking bulk projections — no rehydration, no include
      chains
- [ ] 3.2 The materialiser loop with status-based claiming and no visibility watermark (**D5**)
- [ ] 3.3 Per-aggregate transaction boundary with poison isolation (**D14**)
- [ ] 3.4 Stage-1 retry schedule and terminal dead-lettering (**D9**)
- [ ] 3.5 Per-aggregate version counter, read before the envelope is built (**D1**)
- [ ] 3.6 Envelope builder: stable message id, version, trace context, canonical payload hash
- [ ] 3.7 Payload variants per subscriber entitlement, each hashed separately (**D7**)
- [ ] 3.8 Hash-based no-op suppression (**D6**)
- [ ] 3.9 Publication filters and the three tombstone paths, driven by durable projection state
      (**D12**)
- [ ] 3.10 Stub mapper until the peer's schema lands
- [ ] 3.11 Tests: twenty changes yield one message; delete yields a tombstone; draft yields nothing;
      retraction yields a tombstone; scope exit after 90 days of no changes still yields a tombstone;
      identical payload yields nothing; one poison aggregate does not block its batch

## 4. Fan-out and push delivery

- [ ] 4.1 Subscriber seed and fan-out keyed on contract version **and** payload variant
- [ ] 4.2 Compaction with the corrected predicate, never superseding a non-compactable delivery
      (**D4**)
- [ ] 4.3 Dispatcher claim with priority ordering (**D3**)
- [ ] 4.4 Typed client per subscriber: retry, per-attempt timeout, breaker, no redirect following
- [ ] 4.5 Stable idempotency key across retries, trace header, versioned media type
- [ ] 4.6 Response classification and the persistent backoff schedule with jitter
- [ ] 4.7 Dead-lettering on permanent failure and on exhaustion
- [ ] 4.8 Stuck-lock reaper
- [ ] 4.9 Delivered-version guard with its supporting index (**S5**)
- [ ] 4.10 Tests: transient retried, permanent dead-lettered on the first attempt, rate limit honoured,
      idempotency key stable, concurrent workers never double-deliver, **500 000 backfill rows do not
      delay one live row**

## 5. Pull feed

- [ ] 5.1 The feed endpoint with an opaque cursor and a capped page size
- [ ] 5.2 The visibility watermark filter — the line that makes the feed correct
- [ ] 5.3 Expired cursor answers `410` with a backfill action
- [ ] 5.4 Subscriber-scoped payload variant on the feed
- [ ] 5.5 The watermark test as a release gate: slow transaction N, committed N+1, neither then both
- [ ] 5.6 The same watermark test against a read replica if the feed will be served from one
- [ ] 5.7 Tests: pagination over ten thousand rows, cursor resumption, tombstones present

## 6. Observability and operations

- [ ] 6.1 Meters, starting with the oldest-pending-age gauge per stage and subscriber
- [ ] 6.2 Trace propagation from request to peer, with span links across the async hop
- [ ] 6.3 Log redaction, verified by a test asserting a token never appears in output
- [ ] 6.4 Health endpoint reporting oldest pending age and breaker states
- [ ] 6.5 Admin endpoints, admin-scoped and audit-logged, with the two replay mechanisms distinguished
      (**S3**)
- [ ] 6.6 Dashboard and alert rules

## 7. Contract and mapping — blocked on the peer's schema

- [ ] 7.1 Vendor the schema; generate contract types
- [ ] 7.2 Fill `MAPPING_MATRIX.md` completely and resolve its §7 gaps with the peer counterpart —
      before writing mapper code
- [ ] 7.3 Mappers with unmapped-member diagnostics promoted to errors
- [ ] 7.4 Enum tables, exhaustive switches, fail-closed
- [ ] 7.5 Value converters: money, units, dates, phone, coordinates, collection ordering
- [ ] 7.6 Length guards that raise rather than truncate
- [ ] 7.7 Deletion schema selection by change kind (**D11**)
- [ ] 7.8 Schema validation at materialisation, dead-lettering with the field named
- [ ] 7.9 The three enforcement tests: reflection coverage, round-trip property tests, boundary cases
- [ ] 7.10 Scheduled schema-drift job
- [ ] 7.11 Replace and delete the stub contract

## 8. Security — blocked on the mechanism decision

- [ ] 8.1 The authentication seam
- [ ] 8.2 Outbound token acquisition in a delegating handler with caching and early refresh
- [ ] 8.3 Inbound token validation: signature, issuer, audience, lifetime, per-endpoint client
      allowlist
- [ ] 8.4 Scope policies per capability
- [ ] 8.5 The signing fallback path behind the same seam
- [ ] 8.6 Secret-manager wiring; only references in the database
- [ ] 8.7 Rate limits, body size limits, callback allowlist validated at registration and at dispatch
- [ ] 8.8 Personal-data gating per subscriber, with the redaction test and the subject-purge routine

## 9. Inbound

- [ ] 9.1 The ingress endpoint: authenticate, validate structurally, insert, answer `202`
- [ ] 9.2 Reject a missing idempotency key with `400`
- [ ] 9.3 Source identity from the authenticated principal, not an OAuth-specific claim (**S1**)
- [ ] 9.4 Unique dedup index and conflict-ignoring insert
- [ ] 9.5 The inbox worker: claim, translate, execute the **existing** command handler, mark processed
- [ ] 9.6 Inbound translators
- [ ] 9.7 Failure taxonomy: translation permanent, concurrency transient, domain rejection terminal
- [ ] 9.8 Echo suppression by propagation path plus hash suppression, verified across a three-system
      loop (**D6**)
- [ ] 9.9 External-id correlation, with a conflicting binding dead-lettered rather than rebound
- [ ] 9.10 Optional poller advancing its cursor only after a durable insert

## 10. Backfill, reconciliation, hardening and rollout

- [ ] 10.1 Snapshot pages: keyset paged, watermark pinned on the first page and echoed, replica-served
- [ ] 10.2 Snapshot page items carry a comparable version (**D2**)
- [ ] 10.3 Backfill run record with start, pause, resume and status
- [ ] 10.4 Pushed backfill at the lowest priority with a token-bucket throttle
- [ ] 10.5 Subscriber-scoped checksum endpoint with per-id drill-down (**D8**)
- [ ] 10.6 Version-pinned canonical serialiser shared with the payload hash
- [ ] 10.7 Nightly reconciliation job, divergence metric and report table
- [ ] 10.8 Targeted re-emit endpoint
- [ ] 10.9 Retention job matching the shipped schema, per table (**D10**)
- [ ] 10.10 Analyzer or banned-symbols rule for bulk mutations on integrated types
- [ ] 10.11 Failure injection: dispatcher killed mid-flight, database unavailable, peer failing every
      request
- [ ] 10.12 Load test the backfill at the target rate and the feed over a million rows
- [ ] 10.13 Runbook and an on-call game day
- [ ] 10.14 Phased rollout, each phase behind its own flag, phases 0 and 1 dark
