# Outbox delivery (push)

Derived from ADR-0003 §2–§4, §6 and `TECHNICAL_PROPOSAL.md` §5.4.

## ADDED Requirements

### Requirement: Concurrent claiming without coordination

The system SHALL claim pending deliveries with `FOR UPDATE … SKIP LOCKED`, backed by a partial
index matching the claim's `ORDER BY` exactly, and SHALL require no leader election, distributed
lock or external coordination service.

`locked_until` SHALL serve only as a crash-recovery reaper; the row lock SHALL be the primary
safety mechanism.

#### Scenario: Several dispatcher replicas

- **WHEN** four replicas claim from the same subscriber's backlog simultaneously
- **THEN** no delivery row is claimed by more than one replica

#### Scenario: Worker dies holding a claim

- **WHEN** a dispatcher pod is killed after claiming and before the HTTP call completes
- **THEN** the claim transaction aborts and the row is immediately available as `Pending`

#### Scenario: Lock left behind by a hung pod

- **WHEN** a delivery row stays `InFlight` past `locked_until`
- **THEN** the reaper returns it to `Pending` and the event is recorded for investigation

#### Scenario: Adaptive polling

- **WHEN** the previous claim returned a full batch
- **THEN** the next claim runs immediately
- **AND WHEN** it returned nothing, the worker waits approximately one second

### Requirement: Priority ordering guarantees live traffic is never starved

The claim query SHALL order by `priority` before `next_attempt_at`, so that live deliveries
(`priority = 0`) are always drained ahead of backfill deliveries (`priority = 9`).

> Defect **D3**. ADR-0006 §3 promises this, `TECHNICAL_PROPOSAL.md` §5.4 restates it, and §4.3
> builds `ix_delivery_claim (subscriber_id, priority, next_attempt_at, id)` for it — but the claim
> SQL in ADR-0003 §2, which both documents point implementers at, orders by
> `next_attempt_at, id` only. As written, a 500 k-row backfill queued before a live change delays
> every live change behind the whole backfill.

#### Scenario: Backfill queued ahead of a live change

- **WHEN** 500 000 `priority = 9` rows are pending with `next_attempt_at <= now()`
- **AND** one `priority = 0` row is created afterwards
- **THEN** the next claim returns the live row
- **AND** the live row is delivered before any further backfill row

#### Scenario: Index supports the claim

- **WHEN** the claim query is explained
- **THEN** it uses the partial index without a sort step

### Requirement: At-least-once delivery with a stable idempotency key

The system SHALL deliver at least once, SHALL carry a `messageId` (UUIDv7) that is stable across
every retry of the same message, and SHALL send it both in the envelope and as the
`Idempotency-Key` header. Exactly-once SHALL NOT be claimed.

#### Scenario: Retry after a timeout

- **WHEN** a delivery times out and is retried
- **THEN** the retry carries the same `messageId` and the same `Idempotency-Key`

#### Scenario: Peer reports a duplicate

- **WHEN** the peer answers `409` or `200` with an already-processed marker
- **THEN** the delivery is marked delivered, not retried

#### Scenario: Dispatcher killed mid-flight

- **WHEN** the dispatcher is killed after the peer received the request but before the response
  was recorded
- **THEN** the message is redelivered with the same idempotency key and is never lost

### Requirement: Compaction supersedes pending deliveries, and never supersedes a transition message

When a newer message is produced for an aggregate, the system SHALL supersede still-`Pending`
deliveries of older messages for the same `(subscriber, aggregate)` — and SHALL NOT supersede a
delivery whose message is marked non-compactable.

The supersession predicate SHALL reference columns that exist on the delivery table, and
compactability SHALL be resolvable without a cross-partition join at compaction time.

> Defect **D4**. ADR-0003 §4's statement filters on `sequence`, a column `outbox_delivery` does not
> have (it has `message_sequence`), so it does not compile. It also filters on `compactable`,
> which lives on `outbox_message` — so as written it would supersede pending `Event` deliveries,
> losing transition messages the design guarantees are always delivered. The index name also
> differs between ADR-0003 §2 and `TECHNICAL_PROPOSAL.md` §4.3.

#### Scenario: New state supersedes an undelivered older one

- **WHEN** a new message for a `Foo` is fanned out and an older delivery for the same
  subscriber is still `Pending`
- **THEN** the older delivery becomes `Superseded` with a completion timestamp
- **AND** exactly one HTTP request is made for the pair

#### Scenario: In-flight delivery is not superseded

- **WHEN** the older delivery is already `InFlight`
- **THEN** it is not superseded and may arrive after the newer message
- **AND** the consumer's version rule leaves it converged on the newer state

#### Scenario: Transition message is never compacted

- **WHEN** a non-compactable `Event` delivery is pending and a newer state message arrives
- **THEN** the `Event` delivery remains `Pending` and is delivered

### Requirement: Responses are classified into transient, permanent and success

The system SHALL classify every delivery outcome, retry only transient failures on a persistent
backoff schedule with jitter, and quarantine permanent failures immediately.

| Response | Classification | Action |
|---|---|---|
| 2xx | success | mark delivered |
| 409, or 200 with an already-processed marker | success (duplicate) | mark delivered |
| 400, 422 | permanent | dead-letter without retry, alert |
| 401, 403 | permanent | dead-letter, credential alert |
| 404 | permanent | dead-letter, configuration alert |
| 408, 429, 5xx, timeout, DNS/TCP failure | transient | persistent backoff schedule |

#### Scenario: Schema rejection

- **WHEN** the peer answers `422`
- **THEN** the delivery is dead-lettered on the first attempt and the permanent-4xx alert
  condition advances

#### Scenario: Rate limited

- **WHEN** the peer answers `429` with `Retry-After`
- **THEN** the next attempt is scheduled no earlier than that value

#### Scenario: Peer outage

- **WHEN** the peer returns `503` for six hours
- **THEN** attempts follow the persistent schedule across pod restarts and no message is lost

#### Scenario: Attempts exhausted

- **WHEN** the last scheduled attempt fails
- **THEN** the delivery is dead-lettered with its attempt count and first failure time, and an
  alert fires

### Requirement: Per-subscriber isolation

The system SHALL isolate subscribers from one another: a circuit breaker per subscriber, bounded
in-flight requests per subscriber, and a rate limit taken from the subscriber's configuration.

#### Scenario: One subscriber is down

- **WHEN** one subscriber's breaker is open
- **THEN** the dispatcher skips that subscriber without consuming its persistent attempts
- **AND** other subscribers continue to be served at full rate

#### Scenario: Breaker state is observable

- **WHEN** a breaker opens
- **THEN** the circuit-state metric reflects it per subscriber, and a ticket alert fires if it
  stays open beyond the configured window

### Requirement: Subscribers are configuration data, not code

The system SHALL onboard a subscriber as a data row — callback URL, contract version, auth mode,
auth configuration reference, aggregate types, filters, PII grant, rate limit, enabled flag —
requiring no redeployment.

#### Scenario: New subscriber onboarded

- **WHEN** a subscriber row is inserted and enabled
- **THEN** subsequent messages for its aggregate types are fanned out to it with no code change

#### Scenario: Pull-only subscriber

- **WHEN** a subscriber has no callback URL
- **THEN** no delivery rows are created for it and it consumes the change feed instead

#### Scenario: Subscriber paused

- **WHEN** a subscriber is disabled
- **THEN** no new delivery rows are created and existing pending rows are not attempted

### Requirement: Backfill deliveries never overwrite fresher live state

The system SHALL refuse to send a backfill delivery whose `aggregateVersion` is lower than a
version already delivered to that subscriber for the same aggregate, and SHALL support that check
with an index over delivered rows.

> Defect **S5**. ADR-0006 §3 requires this guard, but `ix_delivery_compaction` is partial on
> `status = 'Pending'`, leaving the delivered-row lookup unindexed — 500 000 unindexed lookups
> during a backfill.

#### Scenario: Stale backfill page after a live update

- **WHEN** a live update for an aggregate has been delivered at version 57
- **AND** a backfill delivery for the same aggregate carries version 42
- **THEN** the backfill delivery is completed without being sent

#### Scenario: Guard is indexed

- **WHEN** the guard query is explained
- **THEN** it resolves through an index on `(subscriber_id, aggregate_key)` over delivered rows

### Requirement: The push stream is a compacted subsequence of the feed

The system SHALL document that push and pull deliver the same converged state but not the same
message sequence: push omits superseded messages, while the change feed exposes every message.

> Defect **S4**. ADR-0001's Consequences claim push and pull "cannot diverge because they read the
> same rows". They read the same table, not the same rows. A consumer migrating between the two
> paths needs to know this.

#### Scenario: Consumer migrates from push to pull

- **WHEN** a subscriber switches from push delivery to the change feed at its last received
  version
- **THEN** it receives intermediate messages it never saw over push
- **AND** applying the documented version rule leaves it in the same converged state
