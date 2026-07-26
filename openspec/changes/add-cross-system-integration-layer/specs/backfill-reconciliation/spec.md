# Backfill and reconciliation

Derived from ADR-0006 and `TECHNICAL_PROPOSAL.md` §5.6, §9.4.

## ADDED Requirements

### Requirement: Snapshot-then-stream handover needs no write freeze

The system SHALL bootstrap a consumer by pinning a safe watermark, serving the full catalogue as
snapshot pages, and having the consumer resume the incremental feed from that watermark — with no
maintenance window, no write freeze and no coordinated cutover.

The watermark SHALL be the highest sequence below the feed's visibility watermark at the start of
the run, SHALL be returned on the first page and SHALL be echoed on every page.

#### Scenario: Record changed during the run

- **WHEN** an aggregate is modified after its snapshot page was served and before the run completes
- **THEN** the consumer receives it twice — once in the page, once through the feed after the
  watermark
- **AND** the version rule leaves it converged on the later state

#### Scenario: Consumer loses the watermark

- **WHEN** a consumer discards the first page's watermark
- **THEN** it recovers the value from any subsequent page

#### Scenario: Handover completes

- **WHEN** the last page reports no more data
- **THEN** the consumer resumes the incremental feed from the pinned watermark with no gap

### Requirement: Snapshot pages are keyset-paged and resumable

Snapshot pagination SHALL use keyset pagination over a stable key, SHALL NOT use offset pagination,
and SHALL be resumable from the cursor alone.

#### Scenario: Crash mid-run

- **WHEN** a consumer crashes on page 1 400 of 2 000
- **THEN** it resumes at page 1 400 from its stored cursor with no repeated or skipped record

#### Scenario: Data shifts under a long run

- **WHEN** records are inserted and deleted while a run is in progress
- **THEN** no record is silently skipped and no record is returned twice within one page sequence

#### Scenario: Pages leave no queue footprint

- **WHEN** a snapshot page is served
- **THEN** it is materialised on demand from current state and no outbox row is written

### Requirement: Snapshot pages carry a comparable version

Every snapshot page item SHALL carry an `aggregateVersion` comparable with the versions of live
messages for the same aggregate, including for aggregates that have never produced an outbound
message.

> Defect **D2**. Pages are never written to the outbox, so under `aggregateVersion = outbox
> sequence` a page item has no version — and at go-live most of the catalogue has never produced a
> message, so no sequence exists for it in principle. The idempotency rule the handover depends on
> would be undefined for exactly the messages that bootstrap the peer. See
> `outbox-materialization` for the corrected version source.

#### Scenario: Never-changed aggregate served in a page

- **WHEN** an aggregate untouched since capture was enabled is served in a snapshot page
- **THEN** it carries a version the consumer can compare against later live messages

#### Scenario: Page item versus a later live message

- **WHEN** the same aggregate is later updated and delivered through the feed
- **THEN** the live message's version is strictly greater and the consumer applies it

### Requirement: Push-mode backfill cannot starve or overwrite live traffic

For peers that do not pull, the system SHALL support pushed backfill that is separately queued at
the lowest priority, throttled by a configurable token bucket, pausable and resumable through a run
record, and never able to supersede or overwrite a fresher live message.

#### Scenario: Live change during a two-day backfill

- **WHEN** a live change is queued while a backfill is draining
- **THEN** the live change is delivered before any further backfill message

#### Scenario: Backfill message older than delivered live state

- **WHEN** a backfill message's version is lower than a version already delivered for that
  aggregate
- **THEN** it is not sent

#### Scenario: Run paused and resumed

- **WHEN** an operator pauses a run and resumes it later
- **THEN** it continues from its stored cursor with its completed count intact

#### Scenario: Throttle protects the peer

- **WHEN** a run's rate limit is set
- **THEN** dispatch to that peer stays at or below it, independently of live traffic

### Requirement: Reconciliation compares bucketed checksums, scoped to the subscriber

The system SHALL expose bucketed checksums — each aggregate contributing a hash of its canonical
contract payload, combined order-independently per bucket — so that a full consistency check costs
one small response, with drill-down to per-id hashes for mismatching buckets only.

Checksums SHALL be scoped to the authenticated subscriber: its filters, its contract version and its
payload variant.

> Defect **D8**. ADR-0006 §4 defines checksums globally, but subscriber filters mean a subscriber
> legitimately holds a subset and redaction (**D7**) means it holds different content. A filtered
> subscriber would mismatch on approximately every bucket, every night — noise that trains everyone
> to ignore the divergence metric. The internal fallback comparison inherits the same problem,
> because last-delivered hashes are per variant.

#### Scenario: Subscriber with a region filter

- **WHEN** a subscriber restricted to one region requests checksums
- **THEN** the buckets cover only records within its scope
- **AND** a correctly synchronised subscriber sees zero mismatches

#### Scenario: Subscriber without a personal-data grant

- **WHEN** such a subscriber requests checksums
- **THEN** the hashes are computed over its redacted payload variant

#### Scenario: Genuine divergence

- **WHEN** one record differs between us and the subscriber
- **THEN** exactly one bucket mismatches, drill-down names the id, and only that record is
  re-fetched

#### Scenario: Divergence is visible on our side too

- **WHEN** the nightly job finds a mismatch
- **THEN** the divergence gauge rises, the divergent ids are recorded, and a ticket alert fires —
  because a mismatch usually means messages were dead-lettered and nobody looked

### Requirement: Divergence has a targeted repair path

The system SHALL provide an admin-scoped operation that re-materialises and re-queues a named set of
aggregates for a given subscriber, and SHALL audit-log every use.

#### Scenario: Repair a handful of ids

- **WHEN** an operator submits the divergent ids for a subscriber
- **THEN** those aggregates are re-materialised and queued for that subscriber only
- **AND** no full re-export is required

#### Scenario: Repair is audited

- **WHEN** a re-emit runs
- **THEN** the actor, the subscriber, the ids and the time are recorded

### Requirement: Deletions are reconcilable after a backfill

The system SHALL state, in the consumer-facing contract, how a consumer detects records that no
longer exist on our side after a backfill run, and SHALL retain deletion tombstones in the
incremental feed for the full retention window.

#### Scenario: Record deleted before the run

- **WHEN** an aggregate was deleted before a backfill run started
- **THEN** it is absent from every page and absent from the subscriber-scoped checksums
- **AND** the documented rule lets the consumer identify its local copy as stale

#### Scenario: Record deleted during the window

- **WHEN** an aggregate is deleted after the run's watermark
- **THEN** a tombstone reaches the consumer through the incremental feed

### Requirement: Backfill reads do not endanger the primary

Snapshot pages and checksum computation SHALL be servable from a read replica, and the pinned
watermark SHALL make replica lag harmless to correctness.

#### Scenario: Large run against a replica

- **WHEN** a 500 000-record run is served from a replica under lag
- **THEN** the primary's write latency is unaffected
- **AND** the handover remains correct because the watermark was pinned at the start
