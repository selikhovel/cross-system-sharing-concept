# Integration operations

Derived from `TECHNICAL_PROPOSAL.md` §9–§12 and ADR-0003 §8.

## ADDED Requirements

### Requirement: One metric answers "is the pipeline healthy"

The system SHALL expose the age of the oldest unprocessed work item, per stage and per subscriber,
as the primary service level indicator, alongside queue depth, delivery attempts by status class,
delivery duration, dead letters by direction and failure kind, materialisation duration, unmapped
value counts by field, feed requests by consumer, divergence counts and circuit state per subscriber.

#### Scenario: Dispatcher stopped

- **WHEN** the dispatcher stops while work is pending
- **THEN** the pending-age gauge rises continuously and the stall alert pages after the configured
  window

#### Scenario: Silent field loss

- **WHEN** a mapper starts dropping values for a field
- **THEN** the unmapped-value counter rises for that field and the daily threshold alert fires

#### Scenario: Inbound stall

- **WHEN** inbound processing stops
- **THEN** the inbound pending-age gauge rises independently of the outbound one

### Requirement: Alerts distinguish "we are broken" from "they are broken"

The system SHALL alert on: pipeline stall, queue growth without deliveries, any new dead letter,
a subscriber breaker open beyond a window, a spike of permanent client errors, any outbound
authentication failure, non-zero divergence, unmapped value volume, and schema drift — with paging
reserved for conditions that mean data is not moving or a contract has broken.

#### Scenario: Contract break

- **WHEN** the peer starts rejecting payloads with a schema error
- **THEN** the permanent-client-error alert pages, because a deploy or a schema change is the usual
  cause

#### Scenario: Credential expiry

- **WHEN** outbound requests start failing authentication
- **THEN** the alert pages rather than being absorbed by retries

#### Scenario: First dead letter

- **WHEN** the first dead letter of the day appears
- **THEN** a ticket is raised, escalating to a page above the configured hourly rate

### Requirement: One trace spans a change end to end

The system SHALL propagate W3C trace context from the originating HTTP request through the capture
row, the materialiser, the dispatcher and the peer request, so that "where did this update go?" is
answerable from one trace.

#### Scenario: Following a single change

- **WHEN** an operator searches by correlation id or aggregate id
- **THEN** one trace shows the write, the capture, the materialisation, every delivery attempt and
  the peer's response codes

#### Scenario: Message inspected without reading its payload

- **WHEN** an operator diagnoses a delivery
- **THEN** identity, timing and status are available in the trace without exposing payload content

### Requirement: Every symptom has a first check and an action

The system SHALL ship a runbook mapping each observable symptom to its first diagnostic step and its
remediation, covering at minimum: rising pending age, blanket authentication failures, blanket
schema rejections, one aggregate always failing, a consumer reporting missing data, an expired feed
cursor, a growing dead-letter queue, and stale worker locks.

#### Scenario: On-call receives a stall page

- **WHEN** the pending-age alert pages
- **THEN** the runbook's first check identifies whether the dispatcher is running and whether a
  breaker is open

#### Scenario: Consumer reports missing data

- **WHEN** a consumer reports gaps
- **THEN** the runbook decides between replay and backfill by comparing their cursor against the
  retained window

### Requirement: Replay distinguishes its two mechanisms

The system SHALL define two distinct replay operations and SHALL state which applies when: resetting
a delivery to pending — valid only while the message payload is still retained — and re-sending from
a retained dead-letter payload, valid for the longer dead-letter window.

Every replay SHALL require the admin scope and SHALL be audit-logged.

> Defect **S3**. ADR-0003 §6 and `TECHNICAL_PROPOSAL.md` §9.5 both call this "replay", but the
> delivery reset needs `outbox_message` (retained 30 days) while the dead-letter record keeps its own
> payload for 180 days. A dead letter older than the outbox window cannot be replayed by the
> mechanism the runbook describes.

#### Scenario: Recent dead letter

- **WHEN** a dead letter younger than the message retention window is replayed
- **THEN** the delivery is reset to pending with its attempt count cleared, and the retained message
  is sent

#### Scenario: Old dead letter

- **WHEN** a dead letter older than the message retention window is replayed
- **THEN** it is re-sent from the dead-letter payload, or re-materialised from current state, and the
  operator is told which happened

#### Scenario: Replay audited

- **WHEN** any replay runs
- **THEN** the actor, the affected ids and the time are recorded

### Requirement: Retention has one defined mechanism per table, matching the schema

Each integration table SHALL have a stated retention window and a retention mechanism that its actual
schema supports, and the condition for moving a table from batched deletion to partition rotation
SHALL be recorded.

> Defect **D10**. ADR-0003 §8 assigns `outbox_delivery` to "its parent partition" and `inbox_message`
> to monthly partitions, but the DDL defines both as unpartitioned tables — and `outbox_delivery`'s
> single-column identity primary key precludes partitioning by creation time without changing it.
> `outbox_message` is specified as partitioned and then recommended unpartitioned for phase 1. The
> phase-1 reality is batched deletion on three hot queue tables, which is what §8 argues against.

#### Scenario: Retention job runs in phase one

- **WHEN** the retention job runs against the shipped schema
- **THEN** each table is pruned by the mechanism its schema supports
- **AND** deletion runs in bounded batches with the autovacuum impact acknowledged

#### Scenario: Growth threshold reached

- **WHEN** a table crosses its recorded partitioning threshold
- **THEN** the migration to partition rotation is triggered as planned work, not discovered under
  load

#### Scenario: Consumer catch-up window

- **WHEN** retention is changed
- **THEN** the consumer-facing maximum catch-up window is updated in the same change

### Requirement: Every phase is independently reversible without a deploy

The system SHALL gate capture, materialisation, dispatch, feed and inbound behind separate flags,
plus a per-subscriber enable flag, so that each rollout phase can be switched off at runtime.

Phases that only accumulate and validate data SHALL ship dark, before any message is delivered.

#### Scenario: Dark launch

- **WHEN** capture and materialisation are enabled and dispatch and feed are not
- **THEN** payloads accumulate and are validated against the peer's schema with nothing delivered

#### Scenario: Emergency stop

- **WHEN** dispatch is disabled at runtime
- **THEN** deliveries stop, no message is lost, and re-enabling drains the backlog

#### Scenario: Single subscriber isolated

- **WHEN** one subscriber is disabled
- **THEN** other subscribers are unaffected

### Requirement: Scaling levers are ordered, and re-platforming is last

The system SHALL document the order in which capacity is added: dispatcher batch size and
per-subscriber concurrency, then API replicas, then a separate worker deployment, then read-replica
reads for feed, backfill and checksums, then partitioning — and only then a change of data-capture
or transport technology.

#### Scenario: Backlog grows

- **WHEN** the queue is not draining fast enough
- **THEN** the first response is configuration, not architecture

#### Scenario: Worker load affects API latency

- **WHEN** dispatch load measurably affects request latency
- **THEN** workers are moved to their own deployment by configuration, without a code change

### Requirement: The obligations placed on consumers are written down

The system SHALL publish a consumer-facing contract stating every obligation its correctness depends
on: idempotency by version or message id, tolerance of at-least-once delivery and of double delivery
during a backfill handover, the maximum catch-up window and the expired-cursor response, ignoring
unknown fields, tombstone handling, the difference between the pushed and pulled message streams, the
checksum comparison procedure, and the loop-prevention field if one is agreed.

> Defect **D13**. ADR-0001, ADR-0003 §3 and §8, and ADR-0006 §5 all refer to this document as though
> it exists. It does not. A design whose correctness depends on consumer behaviour and states that
> behaviour nowhere the consumer can read is not finished.

#### Scenario: New consumer onboards

- **WHEN** a team integrates against the feed
- **THEN** every rule they must implement is in one document, with no reference to our internal ADRs

#### Scenario: Guarantee changes

- **WHEN** a retention window or a delivery guarantee changes
- **THEN** the consumer-facing contract is updated in the same change as the ADR
