# Outbox materialisation

Derived from ADR-0002 §2, ADR-0003 §1, `TECHNICAL_PROPOSAL.md` §5.2–§5.3 and
`MAPPING_MATRIX.md` §5.

## ADDED Requirements

### Requirement: Deferred materialisation with batch collapse

The system SHALL materialise contract payloads in a background worker, separately from capture,
and SHALL collapse multiple pending capture rows for the same aggregate into a single outbound
message carrying the aggregate's current state.

Aggregates SHALL be read with `AsNoTracking()`, in bulk (`WHERE id = ANY(@ids)`), projected
directly into a flat snapshot read model — no aggregate rehydration and no `Include` chains.

#### Scenario: Edit storm collapses

- **WHEN** twenty changes to one `Foo` are pending in one batch
- **THEN** exactly one `outbox_message` row is produced, reflecting the latest state

#### Scenario: Worker crashes mid-batch

- **WHEN** the materialiser is killed after reading a batch and before committing
- **THEN** the capture rows return to `Pending` and are re-materialised
- **AND** no partially emitted message exists

#### Scenario: Contract version two is introduced

- **WHEN** a second contract version is added and the affected capture rows are reset to
  `Pending`
- **THEN** payloads are rebuilt from current domain state with no migration of historical
  payloads

### Requirement: Materialisation is claimed by status, without a visibility watermark

The system SHALL claim pending capture rows with `FOR UPDATE … SKIP LOCKED` filtered on
`status = 'Pending'`, and SHALL NOT gate that claim on the `xid8` visibility watermark.

> Defect **D5**. ADR-0003 §5 applies the watermark to the materialisation scan "for the same
> reason" as the feed. The reason does not transfer: the feed hands out a monotonic cursor a
> consumer advances past invisible rows, whereas status-based claiming cannot skip a row — an
> uncommitted capture row is invisible and is claimed once it commits. Applying the watermark
> here delays all materialisation by the longest concurrent write transaction and inflates
> `integration_outbox_pending_age_seconds`, the primary SLI, for reasons unrelated to pipeline
> health.

#### Scenario: A long-running unrelated write transaction is open

- **WHEN** an unrelated domain transaction stays open for four minutes
- **AND** a `Foo` change commits during that window
- **THEN** the change is materialised without waiting for the long transaction
- **AND** the pending-age metric reflects only genuine pipeline delay

#### Scenario: Concurrent materialiser replicas

- **WHEN** three replicas run the materialiser against the same backlog
- **THEN** no capture row is claimed by more than one replica
- **AND** no aggregate produces duplicate messages for the same claim

### Requirement: `aggregateVersion` is a per-aggregate counter known before serialisation

The system SHALL stamp every outbound message with an `aggregateVersion` that is monotonic per
aggregate, comparable by the consumer, available **before** the message row is inserted, and
defined for every integrated aggregate — including aggregates that have never produced an
outbound message.

The version SHALL come from a dedicated per-aggregate counter maintained as an EF shadow property
on the domain table, not from the generated `outbox_message.sequence`.

> Defects **D1** and **D2**. `TECHNICAL_PROPOSAL.md` §5.2 builds the envelope and hashes it at
> step 6 but inserts the row at step 8, so `sequence` — `GENERATED ALWAYS AS IDENTITY` — is not
> yet assigned; the ordering is circular. And backfill snapshot pages are never written to the
> outbox (ADR-0006 §2), so under the sequence-based option the majority of the catalogue has no
> version at go-live, exactly when the idempotency rule must hold.

#### Scenario: Envelope is built before the insert

- **WHEN** the materialiser builds an envelope and computes `payload_hash`
- **THEN** `aggregateVersion` is already known without reading a generated column
- **AND** the hash covers the final bytes that will be delivered

#### Scenario: Aggregate never changed since capture went live

- **WHEN** a `Foo` that has not been modified since capture was enabled is served through a
  backfill snapshot page
- **THEN** it carries an `aggregateVersion` comparable with the versions of subsequent live
  messages for the same aggregate

#### Scenario: Consumer applies out of order

- **WHEN** a consumer receives version 43 and then version 42 for the same aggregate
- **THEN** applying the documented rule (`ignore if incoming <= stored`) leaves the consumer on
  version 43

### Requirement: Payload variants are modelled per subscriber entitlement

The system SHALL treat the payload variant — full or redacted, and any future variant — as an
explicit dimension of an outbound message, alongside contract version, and SHALL compute a
separate `payload_hash` per variant.

Redaction SHALL happen at materialisation, never at dispatch.

> Defect **D7**. ADR-0005 §3 requires a redacted variant for subscribers without a PII grant, but
> `outbox_message` stores one payload and one hash and the fan-out is keyed on `contract_version`
> only, so the variant has nowhere to live. Rendering at dispatch instead would leave
> `payload_hash` undefined and break reconciliation and the phase 0–1 dark-launch validation.

#### Scenario: Two subscribers differing only in PII grant

- **WHEN** an aggregate carrying contact details is materialised for a subscriber with the
  PII grant and one without
- **THEN** two payload variants exist, each with its own hash
- **AND** the non-granted subscriber's payload contains no field flagged `pii` in
  `MAPPING_MATRIX.md`

#### Scenario: Redaction is asserted, not assumed

- **WHEN** the PII gating test runs over every field flagged `pii`
- **THEN** each has an explicit grant decision and the redacted variant is verified field by field

### Requirement: Publication filters and tombstones

The system SHALL apply publication filters at materialisation time and SHALL emit a tombstone —
never silence — whenever a previously published aggregate stops qualifying.

#### Scenario: Draft is never published

- **WHEN** a `Foo` in `Draft` status is materialised
- **THEN** no message is produced

#### Scenario: Published record is retracted to draft

- **WHEN** a previously published `Foo` moves to `Draft`
- **THEN** a message with `changeKind = 'Deleted'` is produced

#### Scenario: Soft delete

- **WHEN** a published `Foo` is soft-deleted
- **THEN** a tombstone message is produced

#### Scenario: Data-quality failure

- **WHEN** a `Foo` has no `Baz`
- **THEN** materialisation fails that aggregate with a data-quality error and dead-letters it
- **AND** no partial payload is emitted

### Requirement: Subscriber scope transitions are derived from durable projection state

The system SHALL maintain durable per-`(subscriber, aggregate)` projection state — at minimum the
last emitted version, the last emitted payload hash, the emitted variant and an in-scope flag —
and SHALL derive scope-exit tombstones from that state rather than from delivery history.

> Defect **D12**. `MAPPING_MATRIX.md` §5 requires a tombstone when a record leaves a subscriber's
> scope, but the only state recording prior scope membership is `outbox_delivery`, retained 30
> days. For any aggregate not changed within the window — most of a 500 k-row catalogue — "left
> scope" is indistinguishable from "was never in scope", so no tombstone is emitted and the peer
> keeps the record permanently.

#### Scenario: Record leaves a subscriber's region filter

- **WHEN** a `Foo` last changed 90 days ago moves to a region outside a subscriber's filter
- **THEN** a tombstone is emitted to that subscriber only
- **AND** other subscribers receive a normal update

#### Scenario: Record was never in scope

- **WHEN** a `Foo` outside a subscriber's filter is updated
- **THEN** no message and no tombstone is produced for that subscriber

### Requirement: No-op emissions are suppressed by payload hash

The system SHALL compare a newly materialised payload hash against the last emitted hash for the
same `(aggregate, subscriber variant)` and SHALL emit nothing when they are equal.

> Defect **D6**, primary mitigation. This terminates replication loops of any topology — a loop
> converges on a fixed point by definition — without requiring a contract change from the peer,
> and it removes traffic from writes with no externally visible effect.

#### Scenario: Inbound message applied unchanged

- **WHEN** an inbound message from `peer-a` is applied and produces a payload identical to
  the last one emitted for that aggregate
- **THEN** no outbound message is produced for any subscriber

#### Scenario: Internal-only field edited

- **WHEN** a field excluded from the contract is edited
- **THEN** capture occurs but no outbound message is produced

#### Scenario: Three-system loop

- **WHEN** three systems replicate the same aggregate and each applies what it receives
- **THEN** propagation stops within one full circuit, with no version escalation

### Requirement: Per-aggregate transaction boundary and poison isolation

The system SHALL commit materialisation per aggregate — batching only the claim and the read —
so that one failing aggregate can neither roll back nor block its batch peers.

> Defect **D14**. `TECHNICAL_PROPOSAL.md` §5.2 states that steps 7–10 run in one transaction while
> also dead-lettering an invalid payload "now". For a batch of 100 those conflict, and an
> exception aborts the batch, returning the poison row to `Pending` alongside 99 healthy ones —
> reintroducing at materialisation the poison-message retry loop ADR-0003 §6 designs out for
> delivery.

#### Scenario: One poison aggregate in a batch of one hundred

- **WHEN** one aggregate in a claimed batch of 100 fails schema validation
- **THEN** the other 99 messages are emitted and their capture rows marked materialised
- **AND** the failing aggregate is dead-lettered independently
- **AND** the next cycle does not re-attempt the 99

### Requirement: Materialisation failures have a retry schedule and a terminal state

The system SHALL apply a bounded retry schedule with backoff and jitter to capture rows whose
materialisation fails transiently, SHALL move exhausted and permanently failing rows to a
dead-letter record linked to the originating capture row, and SHALL alert on them.

Every status value a capture row can hold SHALL have defined semantics.

> Defect **D9**. `outbox_change_log` has `status`, `attempt` and `last_error` but no
> `next_attempt_at`, and ADR-0003 §6's schedule is specified only for `outbox_delivery` and
> `inbox_message`. A row moved to `Failed` leaves the partial claim index, produces no
> dead-letter row, no metric and no alert — a permanent, unannounced data gap. The `'Skipped'`
> status appears only inside a `CHECK` constraint and is defined nowhere.

#### Scenario: Transient failure

- **WHEN** materialisation fails because the database connection drops
- **THEN** the capture row is retried on the ADR-0003 §6 schedule with jitter

#### Scenario: Attempts exhausted

- **WHEN** a capture row exhausts its attempts
- **THEN** a dead-letter record is written, referencing the capture row, with a failure kind
- **AND** the failed-materialisation counter increases and an alert fires

#### Scenario: Filtered out

- **WHEN** an aggregate is not published because of a filter rule
- **THEN** the capture row reaches a terminal status distinguishable from a failure

### Requirement: Payloads are validated before they enter the outbox

The system SHALL validate every materialised payload against the peer's vendored schema for that
message type before inserting it, and SHALL dead-letter a failing payload immediately with the
offending field named — never queue it for delivery.

#### Scenario: Invalid payload

- **WHEN** a mapper produces a payload violating the peer's schema
- **THEN** it is dead-lettered at materialisation with the field name and the violated constraint
- **AND** no delivery attempt is made

#### Scenario: Tombstone validation

- **WHEN** a tombstone payload is validated
- **THEN** it is validated against the schema for the deletion message type, not against the
  full-snapshot schema

> Defect **D11**. See `contract-mapping` for the requirement this scenario depends on.
