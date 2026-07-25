# Implementation Plan — ProReel Estate Integration Layer

Companion to [TECHNICAL_PROPOSAL.md](TECHNICAL_PROPOSAL.md). Ordered so that **something
demonstrable exists at the end of every day**, and so that the parts blocked on the peer's
schema come last.

**Critical-path note:** the peer's OpenAPI schema (Q3) blocks only the mapper and the contract
project. Everything in Day 1–4 is independent of it — start there today and chase the schema
in parallel.

---

## Day 0 — Prerequisites (30 minutes, do this first)

- [ ] `SELECT version();` on production — **must be 13+** for `xid8` / `pg_snapshot_xmin`
      (Risk R4). If lower, stop and re-plan the feed watermark.
- [ ] Confirm the EF Core / .NET version actually in the solution.
- [ ] Confirm whether aggregates already raise domain events (if yes, ADR-0002's opt-in event
      path gets simpler).
- [ ] Get row counts: `Property`, `Building`, `Location` (sizes the backfill).
- [ ] Open tickets for Q1–Q8 (proposal §2.3) and assign owners.
- [ ] Request the peer's OpenAPI schema **today** — it is the longest lead-time item.

---

## Day 1 — Foundation and transactional capture

**Goal at end of day:** a domain write produces exactly one `outbox_change_log` row, in the
same transaction, with a test proving rollback leaves nothing.

1. **Projects**
   - [ ] `ProReelEstate.Integration` (classlib), `ProReelEstate.Integration.Contracts` (stub),
         `ProReelEstate.Integration.Tests`.
   - [ ] Reference direction: `Integration → Application → Domain`. Add the NetArchTest
         assertion *now*, while it is trivially green.
   - [ ] Packages: `Riok.Mapperly`, `Microsoft.Extensions.Http.Resilience`, `JsonSchema.Net`,
         `Testcontainers.PostgreSql`, `WireMock.Net`, `FluentAssertions` **7.x** (pin — v8+ is
         commercial), `NetArchTest.Rules`.

2. **Schema**
   - [ ] EF configurations for `outbox_change_log`, `outbox_message`, `outbox_delivery`,
         `subscriber`, `inbox_message`, `dead_letter` (proposal §4). Ship **unpartitioned**.
   - [ ] `HasDefaultSchema("integration")` for these entities; keep them in the existing
         `AppDbContext` (proposal §4.1 note) — one context, one transaction, no plumbing.
   - [ ] Migration + apply. `insert_xid xid8 NOT NULL DEFAULT pg_current_xact_id()` on
         `outbox_message` and `outbox_change_log` — add it now, not later.

3. **Change capture**
   - [ ] `IAggregateRootResolver` + explicit registration map + **startup validation that fails
         fast** on an unregistered entity type.
   - [ ] `OutboxChangeCaptureInterceptor` (ADR-0002 §1). Register with
         `AddInterceptors(...)` on the existing `DbContext`.
   - [ ] `Deleted` entities capture `OriginalValues` into `snapshot_hint`.
   - [ ] Feature flag `Integration:CaptureEnabled` (default `false` in prod).

4. **Tests (write these, they are the day's deliverable)**
   - [ ] Testcontainers: commit a `Property` change ⇒ exactly one change-log row.
   - [ ] Testcontainers: **roll back** ⇒ zero change-log rows. *(the "no dual write" proof)*
   - [ ] Five edits to one aggregate in one transaction ⇒ one row.
   - [ ] Startup validation fails when an entity type is unregistered.

---

## Day 2 — Materialisation

**Goal:** change-log rows become `outbox_message` rows carrying a stub contract payload.

- [ ] `PropertySnapshot` / `BuildingSnapshot` / `LocationSnapshot` flat read models.
- [ ] Projection queries: `AsNoTracking()`, `WHERE id = ANY(@ids)`, `.Select(...)` straight into
      the snapshot — **no aggregate rehydration, no `Include` chains**.
- [ ] `MaterializerWorker : BackgroundService` — the 10-step loop in proposal §5.2.
- [ ] Batch claim with `FOR UPDATE SKIP LOCKED` **below the `xid8` watermark**.
- [ ] Collapse duplicates by aggregate within the batch.
- [ ] Envelope builder: UUIDv7 `messageId`, `aggregateVersion` = outbox `sequence` (proposal
      §5.3 option 3), `traceparent`, `payload_hash`.
- [ ] Stub mapper `Snapshot → StubContract` until the peer schema arrives.
- [ ] Filter rules (MAPPING_MATRIX §5): drafts, soft deletes, **tombstones on retraction**.
- [ ] Tests: 20 changes to one aggregate ⇒ one message; delete ⇒ tombstone; draft ⇒ nothing;
      previously-published → draft ⇒ tombstone.

---

## Day 3 — Fan-out and push delivery ⭐ end-to-end slice

**Goal:** a domain write reaches a WireMock "peer" as an HTTP POST. This is the day the
architecture becomes real.

- [ ] `subscriber` seed + fan-out into `outbox_delivery` (aggregate-type + filter match).
- [ ] Compaction: supersede pending deliveries for the same `(subscriber, aggregate_key)`.
- [ ] `DispatcherWorker` with the claim SQL from ADR-0003 §2, priority-ordered.
- [ ] Typed `HttpClient` per subscriber: `Microsoft.Extensions.Http.Resilience` with retry,
      10 s per-attempt timeout, **circuit breaker per subscriber**,
      `AllowAutoRedirect = false`.
- [ ] Headers: `Idempotency-Key = messageId` (**stable across retries** — test it),
      `traceparent`, versioned media type.
- [ ] Response classification table (ADR-0003 §6) + persistent backoff schedule with jitter.
- [ ] Dead-letter on permanent failures and on attempt exhaustion.
- [ ] Stuck-lock reaper (`locked_until < now()` on `InFlight`).
- [ ] Tests (WireMock): 500 ⇒ retried with backoff; 422 ⇒ dead-lettered **immediately, not
      retried**; 429 ⇒ honours `Retry-After`; retry reuses the same `Idempotency-Key`;
      concurrent workers never double-deliver (Testcontainers, N workers).

---

## Day 4 — Pull feed and the watermark

**Goal:** a consumer can walk our change log without gaps — provably.

- [ ] `GET /integration/v1/{aggregate}/changes?cursor=&limit=` (proposal §5.5).
- [ ] **`insert_xid < pg_snapshot_xmin(pg_current_snapshot())` in the query.** This is the line
      that makes the feed correct.
- [ ] `limit` cap 1 000; `AsNoTracking()`; opaque cursor.
- [ ] **410 Gone** when the cursor predates retention, with `{"action":"backfill"}`.
- [ ] ⭐ **The watermark test** (proposal §10): a slow open transaction inserting sequence N,
      a fast committed N+1 — the feed must return *neither*, then *both*. Make this a release
      gate; it is the bug that otherwise ships silently.
- [ ] Tests: pagination over 10 k rows; cursor resumption; tombstones present.

---

## Day 5 — Observability and operations

**Goal:** you can tell whether the pipeline is healthy from one dashboard.

- [ ] OpenTelemetry meters — full list in proposal §9.1. `integration_outbox_pending_age_seconds`
      first; it is the primary SLI.
- [ ] Trace propagation: request → change log → materializer (span link) → dispatcher → peer.
- [ ] Serilog redaction policy (ADR-0005 §4) — verify with a test that a token never appears
      in output.
- [ ] Health checks: `/health/integration` reporting oldest pending age + breaker states.
- [ ] Admin endpoints (scope `admin`, audit-logged): list dead letters, replay,
      pause/resume subscriber, re-emit aggregates.
- [ ] Grafana dashboard + the alert rules from proposal §9.2.

---

## Week 2 — Contract and mapping *(needs the peer schema)*

- [ ] Vendor `contracts/external/{system}/openapi.v1.yaml`; generate DTOs (Kiota/NSwag).
- [ ] **Fill in MAPPING_MATRIX.md completely** — before writing mapper code. Resolve §7 gaps
      with the peer counterpart.
- [ ] Mapperly mappers; `<WarningsAsErrors>$(WarningsAsErrors);RMG012;RMG020</WarningsAsErrors>`.
- [ ] Enum lookup tables, exhaustive switches, fail-closed.
- [ ] Value converters: money, area/units, dates, phone E.164, coordinates, collection ordering.
- [ ] Length guards — **truncation raises, never truncates**.
- [ ] JSON Schema validation at materialisation; failures dead-letter with the field named.
- [ ] The three enforcement tests: field coverage (reflection), round-trip property tests,
      boundary/edge cases.
- [ ] Daily CI job diffing the peer's live schema against the vendored copy.
- [ ] Replace the stub contract; delete the stub.

## Week 2–3 — Security *(needs Q2)*

- [ ] `IIntegrationAuthenticator` seam so the mechanism is swappable.
- [ ] OAuth2 client-credentials `DelegatingHandler` + token cache (refresh at 80 % TTL).
- [ ] JWT bearer validation, JWKS caching, scope policies, per-endpoint `client_id` allowlist.
- [ ] HMAC signing/verification path (fallback ladder, ADR-0005 §2) behind the same seam.
- [ ] Secret-manager wiring; `auth_config_ref` only in the DB.
- [ ] Rate limiting, body size limits, callback-URL allowlist (SSRF).
- [ ] PII gating per subscriber + redaction test.

## Week 3 — Inbound

- [ ] `POST /integration/v1/inbound/{messageType}` — authn, schema validation, insert, 202.
      Reject a missing `Idempotency-Key` with 400.
- [ ] Unique dedup index + `ON CONFLICT DO NOTHING`.
- [ ] `InboxWorker`: claim → translate → **existing application command handler** → processed.
- [ ] Inbound ACL translators (peer contract → command).
- [ ] Error taxonomy: translation = permanent, concurrency = transient, domain rejection = terminal.
- [ ] **Echo suppression** (`causationSource`) — verify a peer update does not bounce back.
- [ ] `external_reference` correlation table.
- [ ] Optional `InboundPollerWorker` + `consumer_cursor` if we pull (Q7).

## Week 3–4 — Backfill and reconciliation

- [ ] `GET …/snapshot?cursor=&limit=` — keyset pagination, pinned watermark, replica-served.
- [ ] `backfill_run` table + admin control (start/pause/resume/status).
- [ ] Push-mode backfill at `priority = 9` with a token-bucket throttle.
- [ ] `GET …/checksums` — 256 buckets, XOR of `sha256(canonical json)`.
- [ ] Canonical JSON serialiser, **version-pinned** (changing it invalidates every hash).
- [ ] Nightly reconciliation job + divergence metric + report table.
- [ ] Load test the backfill at target rate against a WireMock peer.

## Week 4 — Hardening and rollout

- [ ] Retention job; partitioning if warranted.
- [ ] Analyzer/banned-symbols rule for `ExecuteUpdateAsync`/`ExecuteDeleteAsync` on integrated
      types (Risk R3).
- [ ] Failure-injection tests: kill the dispatcher mid-flight; DB unavailable; peer 100 % 500s.
- [ ] Runbook (proposal §9.5) + on-call game day.
- [ ] Phased rollout 0 → 6 (proposal §12), each behind its own flag.

---

## If you only have today

The highest-value four hours, in order:

1. **Day 0 checks** — especially the PostgreSQL version. A wrong answer changes the design.
2. **Day 1 in full** — schema + interceptor + the two Testcontainers tests. This is the load-
   bearing part: transactional capture with the domain untouched. Everything else is
   plumbing on top of it.
3. **Day 2's materializer skeleton** with a stub contract, so the two-stage shape is real in
   code before the peer schema arrives and starts applying pressure.

Do **not** start with the mappers. Without the vendored schema they will be rewritten, and
the mapping layer is the one part that is genuinely blocked.
