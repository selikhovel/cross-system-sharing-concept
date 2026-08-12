# Implementation Plan — Acme Integration Layer

Companion to [TECHNICAL_PROPOSAL.md](TECHNICAL_PROPOSAL.md). The staging and its rationale are
[ADR-0007](adr/ADR-0007-staged-delivery-and-the-first-increment.md) — this document is the task
detail inside each stage, not the decision about their order.

Stage 1 is broken into ordered steps because it is what starts first. Later stages are listed as
task sets, because their internal order depends on what stage 1 learns about the peer's contract.

**Steps are ordered by dependency, not by calendar.** Each one ends in something demonstrable, so
progress is measured by which step closed rather than by how long it took.

**Critical-path note:** the peer's OpenAPI schema (Q3) is the only true external blocker, and
stage 1 is arranged to hit it in its first two steps rather than at the end. Stage 1 can begin
against a stub contract, but it cannot *exit* without the real one.

---

## Stage 0 — Prerequisites (30 minutes, do this first)

- [ ] Request the peer's OpenAPI schema **today** — longest lead time, and stage 1's exit
      criterion depends on it.
- [ ] Confirm the EF Core / .NET version actually in the solution.
- [ ] Get row counts: `Foo`, `Bar`, `Baz` (sizes stage 1's paging and stage 6's
      backfill).
- [ ] Confirm which auth mechanism is available (Q2) — stage 1 ships authenticated, so this is a
      stage 1 prerequisite, not a week-3 one.
- [x] Peer entitled to receive contact personal data (Q6) — **and it marks some of that data
      `required`**, so blanket redaction is not an option. Still open: the legal basis and the
      peer's retention.
- [x] Pilot consumer identified (Q17): exactly one, and the exchange is one-to-one.
- [x] Production PostgreSQL version (Q5): **17**. `xid8` and `pg_snapshot_xmin` are available and
      risk R4 is closed.
- [ ] Confirm whether aggregates already raise domain events (if yes, ADR-0002's opt-in event path
      in stage 3 gets simpler).
- [ ] Open tickets with owners for the twelve questions still open in proposal §2.3, and **assign an
      owner to each of the eight gaps in `MAPPING_MATRIX.md` §7** — all eight are currently ownerless,
      and a gap without a name against it does not move.

---

## Stage 1 — Contract-shaped read API

**Goal:** a peer can `GET` our `Foo` catalogue in *their* contract shape, page by page, built
on demand from the domain. No new tables, no migration, no workers, no write-path change.

> **[STAGE_1_READ_API.md](STAGE_1_READ_API.md) is the implementation guide for this stage** — the
> discovery checklist for an existing Clean Architecture / CQRS / EF Core service, where each file
> goes and why, the code, the tests, and the twelve EF-specific traps. The steps below are the
> checklist; that document is the detail behind every line of it. Start with its §1: several answers
> there change the code you write.

### Step 1 — projects, projection, and a payload on screen

1. **Projects**
   - [ ] `Acme.Integration` (classlib), `Acme.Integration.Contracts` (stub until
         the schema lands), `Acme.Integration.Tests`.
   - [ ] Reference direction `Integration → Application → Domain`. Add the NetArchTest assertion
         **now**, while it is trivially green: `Domain` and `Application` reference neither
         `Integration`, `Integration.Contracts`, nor `System.Net.Http`.
   - [ ] Packages: `Riok.Mapperly`, `JsonSchema.Net`, `Testcontainers.PostgreSql`,
         `FluentAssertions` **7.x** (pin — v8+ is commercial), `NetArchTest.Rules`.
         Not yet needed: `Microsoft.Extensions.Http.Resilience`, `WireMock.Net`.

2. **Snapshot read model**
   - [ ] `FooSnapshot` — flat, serialisation-friendly, no domain types.
   - [ ] Projection query: `AsNoTracking()`, `.Select(...)` straight into the snapshot.
         **No aggregate rehydration, no `Include` chains, no lazy loading.**
   - [ ] Single-id and keyset-range overloads. **Page by the primary key**, not `(created_at, id)`: the composite key would need an index, and stage 1's whole claim is that it needs no migration (guide В§4.3).

3. **Mapper (the ACL)**
   - [ ] `FooContractMapper` with Mapperly and
         `<WarningsAsErrors>$(WarningsAsErrors);RMG012;RMG020</WarningsAsErrors>`.
   - [ ] Stub contract type until the peer's schema arrives; every ignore carries a written reason
         pointing at `MAPPING_MATRIX.md`.

**Step closes when:** a unit test maps a `FooSnapshot` to a contract object and prints the JSON.

### Step 2 — the endpoints

- [ ] `GET /integration/v1/foos/{id}` → the envelope of ADR-0006 §2 with one item, or `404`.
- [ ] `GET /integration/v1/foos?cursor=&limit=` → `{ items, nextCursor, hasMore }`.
- [ ] **Keyset pagination only** — never `OFFSET`. Opaque cursor over the **primary key**;
      `(created_at, id)` needs an index and is stage 3's business (guide §4.3).
- [ ] `limit` capped (default 100, maximum 500); the cap is documented, not silent.
- [ ] Serialisation: one shared source-generated `JsonSerializerContext`, naming policy matching
      the peer's schema, `DefaultIgnoreCondition = Never`, `NumberHandling = Strict`.
- [ ] **No `aggregateVersion` field** — stage 1 declares replace-all semantics (ADR-0007 §Stage 1).
      Do not improvise one from a timestamp.
- [ ] Feature flag `Integration:ReadApiEnabled`, default `false` in production.

### Step 3 — security floor

- [ ] `IIntegrationAuthenticator` seam so the mechanism is swappable (ADR-0005 §2 ladder).
- [ ] Authentication + one scope (`acme.feed.read`) on both routes.
- [ ] Per-caller rate limiting (`Microsoft.AspNetCore.RateLimiting`, fixed window by caller
      identity) and `MaxRequestBodySize`.
- [ ] TLS-only; no certificate-validation callbacks anywhere.
- [ ] Serilog redaction policy for `Authorization`, `X-Signature`, `X-Api-Key`, `*token*`,
      `*secret*` — with a test asserting a token never reaches the output.
- [ ] **Personal data under a recorded grant** (Q6): the peer is entitled and marks some personal fields `required`, so blanket redaction would produce payloads failing its validation. Emit them, and record the legal basis and retention before shipping.

### Step 4 — the tests that make it a contract, not an endpoint

- [ ] ⭐ **Field-coverage test** (ADR-0004 §5b): reflection walk over `Foo`; every public
      property is mapped or listed in `IntegrationFieldExclusions` with a reason. This is the
      highest-value test in the workstream and it belongs in the first stage, not retrofitted.
- [ ] Round-trip property tests over generated values: money precision, area unit conversion,
      date-only fields, non-ASCII strings, DST boundaries, negative coordinates, null versus empty
      collection.
- [ ] Length guards: a value exceeding the peer's `maxLength` **raises**, naming the field and both
      lengths. Never truncates.
- [ ] Enum exhaustiveness: an unmapped domain value fails the build or the first request, never a
      silent fallback.
- [ ] Schema validation: generated payloads validate against the vendored peer schema (once it
      exists).
- [ ] PII test: fields marked `pii` appear **only** where the recorded grant permits.
- [ ] Testcontainers: keyset pagination over 10 000 rows returns every row exactly once, with no
      duplicate and no gap, including while rows are inserted mid-walk.
- [ ] Architecture tests green.

### Step 5 — contract and handover

- [ ] Vendor `contracts/external/{system}/openapi.v1.yaml`; generate DTOs (Kiota or NSwag);
      **delete the stub**.
- [ ] **Fill `MAPPING_MATRIX.md` for `Foo`** and resolve its §7 gaps with the peer
      counterpart. Zero unresolved `GAP-*` rows is a stage 1 exit criterion.
- [ ] Create `CONSUMER_CONTRACT.md`, stating for stage 1: replace-all semantics, no incremental
      sync, no deletion signal, the page contract, the page cap, the rate limit, and that stage 1
      is time-boxed.
- [ ] Minimal observability: request count and duration by route and caller, plus a log line per
      request carrying caller, page size and cursor — never the payload.
- [ ] Walk the pilot consumer through a full catalogue read.

**Stage 1 exit criteria:** the pilot consumer reconstructs the full `Foo` catalogue from the
paged endpoint; `MAPPING_MATRIX.md` complete for `Foo` with no unresolved gaps; coverage,
round-trip and PII tests green; p95 page latency within the agreed budget; no `pii` field in any
response.

**Deliberately absent from stage 1:** change capture, outbox, workers, push, change notification,
deletions, inbound, subscriber table, retention, partitioning. If any of these creeps in, it is a
different stage.

---

## Stage 2 — Change capture, dark

**Goal:** a domain write produces exactly one `outbox_change_log` row in the same transaction, and
`aggregateVersion` appears in stage 1's responses. Still nothing delivered.

**Gated on:** D1 (version source), D6 (provenance columns) — see
`openspec/changes/add-cross-system-integration-layer/design.md`.

- [ ] EF configuration and migration for `integration.outbox_change_log` (proposal §4.1), including
      `insert_xid xid8 NOT NULL DEFAULT pg_current_xact_id()` and the provenance columns
      (causing source, propagation path). Add them now, not later.
- [ ] Keep the integration entities in the existing `AppDbContext` with
      `.ToTable(..., "integration")` — one context, one transaction, no plumbing (proposal §4.1
      note).
- [ ] Per-aggregate version counter as an EF shadow property (`IntegrationVersion`), bumped on
      write. This is ADR-0004 §5.3 option 2, promoted to primary because option 3 is not
      implementable as ordered (**D1**).
- [ ] `IAggregateRootResolver` + explicit registration map + **startup validation that fails fast**
      on an unregistered entity type.
- [ ] `OutboxChangeCaptureInterceptor` (ADR-0002 §1), registered with `AddInterceptors(...)`.
- [ ] `Deleted` entities capture `OriginalValues` into `snapshot_hint`.
- [ ] Add `aggregateVersion` to stage 1's response envelope (additive, no version bump) and update
      `CONSUMER_CONTRACT.md` to permit incremental application of the version rule.
- [ ] Feature flag `Integration:CaptureEnabled`, default `false` in production.
- [ ] Tests: commit ⇒ exactly one row; **rollback ⇒ zero rows** (the "no dual write" proof); five
      edits to one aggregate in one transaction ⇒ one row; startup fails on an unregistered entity
      type; the version counter is monotonic per aggregate under concurrent writes.

**Exit:** capture rows correct in production for a week; no measurable change in write latency.

---

## Stage 3 — Incremental feed

**Goal:** a consumer stops re-reading the catalogue and syncs incrementally — provably without
gaps.

**Gated on:** D2, D5, D9, D11, D14.

- [ ] `integration.outbox_message` (proposal §4.2), **unpartitioned** in this stage.
- [ ] `MaterializerWorker : BackgroundService` — the loop of proposal §5.2, with three corrections:
      claim by `status = 'Pending'` **without** the `xid8` watermark (**D5**), commit **per
      aggregate** so one poison row cannot roll back its batch (**D14**), and a retry schedule plus
      terminal dead-lettering for stage-1 failures (**D9**).
- [ ] Collapse duplicates by aggregate within the batch.
- [ ] Envelope builder: UUIDv7 `messageId`, `aggregateVersion` from the stage 2 counter,
      `traceparent`, canonical `payload_hash`.
- [ ] Reuse the stage 1 projection and mapper unchanged.
- [ ] Filter rules (`MAPPING_MATRIX.md` §5): drafts, soft deletes, **tombstones on retraction**.
- [ ] Deletion message type and schema selection by `changeKind`, so a tombstone is not validated
      against the full-snapshot schema (**D11**).
- [ ] Hash-based no-op suppression — cheap here, and it is the primary loop defence stage 5 needs
      (**D6**).
- [ ] `GET /integration/v1/foos/changes?cursor=&limit=` (proposal §5.5).
- [ ] ⭐ **`insert_xid < pg_snapshot_xmin(pg_current_snapshot())` in the feed query.** This is the
      line that makes the feed correct.
- [ ] `410 Gone` when the cursor predates retention, with `{"action":"backfill"}`.
- [ ] Retention: change log 7 days after materialisation, messages 30 days — batched deletes in
      this stage, with the partitioning trigger recorded (**D10**, resolved in stage 7).
- [ ] Reframe stage 1's paged endpoint as the snapshot endpoint: same route, plus a pinned
      watermark returned on the first page and echoed on every page.
- [ ] ⭐ **The watermark test** as a release gate: a slow open transaction inserting sequence N, a
      fast committed N+1 — the feed returns *neither*, then *both*. Run it against a read replica
      too if the feed will be served from one.
- [ ] Tests: 20 changes ⇒ one message; delete ⇒ tombstone that validates; draft ⇒ nothing;
      previously published → draft ⇒ tombstone; identical payload ⇒ nothing emitted; one poison
      aggregate does not block its batch; pagination and cursor resumption over 10 k rows.
- [ ] Metrics: `integration_outbox_pending_age_seconds` (the primary SLI) first, plus pending count,
      materialisation duration, and failed-materialisation count with an alert.
- [ ] Update `CONSUMER_CONTRACT.md`: at-least-once, version-based idempotency, the 30-day catch-up
      window, the `410` response, tombstone handling.

**Exit:** the watermark test green as a gate; a pilot consumer syncs incrementally for a week with
no gap and no full re-read.

---

## Stage 4 — Push delivery

**Goal:** latency drops from a poll interval to seconds, and a consumer that will not build a
poller can be served.

**Gated on:** D3, D4, D7, D12, S5.

- [ ] `integration.subscriber` (proposal §4.4) and `integration.outbox_delivery` (§4.3).
- [ ] Durable per-`(subscriber, aggregate)` projection state — last emitted version, last payload
      hash, emitted variant, in-scope flag — which scope-exit tombstones and hash suppression both
      need (**D12**).
- [ ] `payload_variant` on `outbox_message`, part of the fan-out key alongside `contract_version`;
      per-subscriber PII grants generalise stage 1's single recorded grant (**D7**, deferred at one consumer).
- [ ] Fan-out on aggregate type, filter match, contract version and variant.
- [ ] Compaction with the corrected predicate — `message_sequence`, not `sequence` — that can see
      `compactable` and never supersedes an `Event` delivery (**D4**).
- [ ] `DispatcherWorker` with the ADR-0003 §2 claim SQL amended to
      `ORDER BY priority, next_attempt_at, id` (**D3**).
- [ ] Typed `HttpClient` per subscriber: `Microsoft.Extensions.Http.Resilience` retry, 10 s
      per-attempt timeout, **circuit breaker per subscriber**, `AllowAutoRedirect = false`.
- [ ] Headers: `Idempotency-Key = messageId` (**stable across retries** — test it), `traceparent`,
      versioned media type.
- [ ] Response classification (ADR-0003 §6) + persistent backoff with ±20 % jitter.
- [ ] Dead-lettering on permanent failure and on exhaustion; stuck-lock reaper.
- [ ] Delivered-version guard for stage 6's backfill, with its supporting index (**S5**).
- [ ] Outbound credentials in a `DelegatingHandler` with token cache and refresh at 80 % TTL;
      secret-manager wiring; `auth_config_ref` only in the database.
- [ ] Callback-URL allowlist validated at registration **and** at dispatch (SSRF).
- [ ] Admin endpoints (`admin` scope, audit-logged): list dead letters, replay, pause/resume a
      subscriber, re-emit aggregates. Distinguish the two replay mechanisms (**S3**).
- [ ] Tests (WireMock): 500 ⇒ retried; 422 ⇒ dead-lettered **immediately**; 429 ⇒ honours
      `Retry-After`; retry reuses the same `Idempotency-Key`; concurrent workers never
      double-deliver; ⭐ **500 000 priority-9 rows do not delay one priority-0 row**.
- [ ] Remaining metrics and the alert rules of proposal §9.2; Grafana dashboard;
      `/health/integration`.

**Exit:** p95 delivery under 10 s to a pilot subscriber; dead letters at zero; the starvation test
green.

---

## Stage 5 — Inbound

**Gated on:** D6 in full, S1.

- [ ] `POST /integration/v1/inbound/{messageType}` — authenticate, validate structurally, insert,
      `202`. Reject a missing `Idempotency-Key` with `400`.
- [ ] `source_system` from the authenticated principal through the seam, **not** from an
      OAuth-specific `client_id` claim (**S1**).
- [ ] `integration.inbox_message` with the unique dedup index and `ON CONFLICT DO NOTHING`.
- [ ] `InboxWorker`: claim → translate → **existing application command handler** → processed.
- [ ] Inbound ACL translators (peer contract → command).
- [ ] Failure taxonomy: translation permanent, concurrency transient, domain rejection terminal.
- [ ] Echo suppression: propagation path in the envelope and the change log, on top of stage 3's
      hash suppression. Verify across a **three-system** loop, not just a direct echo (**D6**).
- [ ] `integration.external_reference`, with a conflicting binding dead-lettered rather than
      rebound.
- [ ] Per-endpoint client allowlist; JWT validation with JWKS caching; scope policies.
- [ ] Optional `InboundPollerWorker` + `consumer_cursor` if we pull (Q7), advancing the cursor only
      after a durable insert.

**Exit:** peer messages applied through the domain; echo suppression verified across a loop.

---

## Stage 6 — Backfill runs and reconciliation

**Gated on:** D8.

- [ ] Formalise stage 3's pinned-watermark snapshot endpoint as a **run**: `backfill_run` table with
      start, pause, resume and status through the admin API.
- [ ] Push-mode backfill at `priority = 9` with a token-bucket throttle (default 50 msg/s).
- [ ] `GET …/checksums` — 256 buckets, XOR of `sha256(canonical json)`, **scoped to the
      authenticated subscriber's filter, contract version and payload variant** (**D8**).
- [ ] Version-pinned canonical JSON serialiser, shared with `payload_hash`. Changing it is a
      contract change.
- [ ] Nightly reconciliation job, divergence metric, report table.
- [ ] Targeted re-emit endpoint for repair.
- [ ] Route backfill and checksum reads to a read replica.
- [ ] Load test the backfill at the target rate against a WireMock peer.
- [ ] Update `CONSUMER_CONTRACT.md`: double delivery during the handover window, the checksum
      comparison procedure, the post-run staleness rule for deletions.

**Exit:** checksums converge; divergence zero for a week.

---

## Stage 7 — Hardening and scale

**Gated on:** D10.

- [ ] Retention mechanism per table, matching the shipped schema, with the partitioning threshold
      recorded (**D10**). Partition `outbox_message` when it warrants it.
- [ ] Analyzer / banned-symbols rule for `ExecuteUpdateAsync` / `ExecuteDeleteAsync` on integrated
      types, with an explicit `[BypassesIntegrationCapture(reason)]` escape (Risk R3).
- [ ] Failure injection: dispatcher killed mid-flight; database unavailable; peer returning 100 %
      `500`s.
- [ ] Load test the feed over a million rows.
- [ ] Worker/API split by configuration (`WORKERS_ENABLED`, `API_ENABLED`).
- [ ] Runbook (proposal §9.5) + on-call game day.

**Exit:** on-call trained; alerts verified in a game day.

---

## Where to start

1. **Stage 0** — especially requesting the peer's schema and naming a pilot consumer. Both are
   other people's latency; start them before anything you control.
2. **Stage 1, step 1** — projects, the projection, the mapper, the architecture test. It is short,
   and it ends with a real payload on screen.
3. **Stage 1, step 4's coverage test** — bring it forward if there is capacity. It is the test that
   makes "nothing is silently lost" true rather than intended.

Do **not** start with the interceptor or the outbox. They are unblocked work that can be done at
any time, by us, with no external dependency — which is exactly why they should not consume the
window in which the peer's schema is still unanswered.
