# Inbound ingestion

Derived from ADR-0001 §2, ADR-0003 §7 and `TECHNICAL_PROPOSAL.md` §6.

## ADDED Requirements

### Requirement: The ingress endpoint is not a domain write path

The inbound endpoint SHALL perform only authentication, authorisation, structural schema
validation and one durable insert into `integration.inbox_message`. It SHALL NOT touch the domain,
open a business transaction, or run mapping logic.

#### Scenario: Valid message accepted

- **WHEN** an authenticated peer posts a schema-valid message
- **THEN** one inbox row is written and the response is `202 Accepted`
- **AND** no domain aggregate is loaded during the request

#### Scenario: Mapping bug downstream

- **WHEN** the message is valid but our translator cannot map it
- **THEN** the request still returned `202`
- **AND** the failure surfaces as a dead-lettered inbox row, never as a `5xx` to the peer

#### Scenario: Constant-time work

- **WHEN** a message referencing an aggregate with thousands of related rows is posted
- **THEN** request handling cost is independent of that aggregate's size

### Requirement: Inbound deduplication on the peer's message identity

The system SHALL deduplicate on `(source_system, source_message_id)` with a unique index and an
insert that ignores conflicts, and SHALL answer `202` for both a new message and a duplicate.

A message without an identifier SHALL be rejected rather than assigned one.

#### Scenario: Peer retries a message it already sent

- **WHEN** the same `Idempotency-Key` is posted twice by the same source
- **THEN** exactly one inbox row exists and both responses are `202`

#### Scenario: Missing idempotency key

- **WHEN** a message arrives without an `Idempotency-Key`
- **THEN** the response is `400`
- **AND** no inbox row is written

#### Scenario: Same key from a different source

- **WHEN** two different source systems post the same key value
- **THEN** both are stored as distinct messages

### Requirement: Source identity comes from the authenticated principal

The system SHALL derive `source_system` from the authenticated principal through the
authentication seam, and SHALL NOT depend on a claim specific to one authentication mechanism.

> Defect **S1**. `TECHNICAL_PROPOSAL.md` §6.1 reads
> `ctx.User.FindFirst("client_id")!.Value`. Under ADR-0005's Tier 2 (mTLS) and Tier 3 (HMAC) there
> is no `client_id` claim, so a correctly authenticated request throws and returns `500` — which
> the peer reads as a delivery failure and retries indefinitely.

#### Scenario: mTLS-authenticated request

- **WHEN** a peer authenticates with a client certificate and posts a message
- **THEN** `source_system` is resolved from the certificate subject through the seam
- **AND** the response is `202`

#### Scenario: HMAC-authenticated request

- **WHEN** a peer authenticates with a signed request and posts a message
- **THEN** `source_system` is resolved from the key identity
- **AND** the response is `202`

#### Scenario: Unresolvable identity

- **WHEN** the principal cannot be resolved to a known source system
- **THEN** the response is `403` and no unhandled exception is thrown

### Requirement: Structural validation failures are attributed to the caller

The system SHALL validate an inbound payload against the vendored schema for its message type and
SHALL answer `400` with the validation errors, so that a contract violation is unambiguously
attributed to the peer rather than to us.

#### Scenario: Schema-invalid payload

- **WHEN** a payload is missing a field the schema requires
- **THEN** the response is `400` naming the offending path
- **AND** no inbox row is written

#### Scenario: Unknown fields present

- **WHEN** a payload carries fields we do not know
- **THEN** they are ignored and the message is accepted

#### Scenario: Oversized body

- **WHEN** a body exceeds the configured maximum
- **THEN** the request is rejected before the payload is buffered into the inbox

### Requirement: Inbound messages are applied through existing application command handlers

The inbox worker SHALL translate a peer payload into an application command through the inbound
anti-corruption layer and SHALL execute it through the existing command handler, so that all
domain invariants apply to external data exactly as to user input. It SHALL NOT mutate the domain
through direct `DbContext` writes.

#### Scenario: Message applied

- **WHEN** the worker claims a pending inbox row and its command succeeds
- **THEN** the row is marked processed and the domain change is captured for outbound emission

#### Scenario: Direct mutation is prevented

- **WHEN** the architecture tests run
- **THEN** no inbound translator writes to a domain entity set directly

### Requirement: Inbound failure taxonomy

The system SHALL classify inbound processing failures and SHALL NOT retry a failure that cannot
succeed on retry.

| Failure | Classification | Action |
|---|---|---|
| Translation / mapping error | permanent | dead-letter immediately, alert |
| Concurrency conflict | transient | retry with backoff |
| Domain rule rejection | terminal, expected | dead-letter with a domain-rejected kind, no retry |
| Infrastructure error | transient | retry with backoff |

#### Scenario: Payload cannot be translated

- **WHEN** translation throws for a structurally valid payload
- **THEN** the row is dead-lettered on the first attempt and an alert fires

#### Scenario: Aggregate busy

- **WHEN** the command fails with a concurrency conflict
- **THEN** the row is retried on the backoff schedule

#### Scenario: Domain says no

- **WHEN** the command is rejected by a domain invariant
- **THEN** the row is dead-lettered as domain-rejected without retry
- **AND** a negative acknowledgement is sent to the peer if their contract supports one

### Requirement: Applied inbound changes do not echo back to their origin

The system SHALL NOT deliver, to any system already on a change's propagation path, an outbound
message produced by applying that system's inbound message.

> Defect **D6**. `TECHNICAL_PROPOSAL.md` §6.2 specifies suppression by `causationSource`, a field
> that exists in no table and in no envelope; and suppression by immediate source cannot break a
> loop across three or more systems, because every hop is a local write that raises the version, so
> the consumer-side version check never terminates it. See `change-capture` for provenance
> recording and `outbox-materialization` for hash-based no-op suppression.

#### Scenario: Direct echo

- **WHEN** a message from `peer-a` is applied and the resulting change is materialised
- **THEN** no delivery row is created for `peer-a`
- **AND** other subscribers receive the message normally

#### Scenario: Indirect loop

- **WHEN** three systems replicate the same aggregate in a cycle
- **THEN** propagation terminates within one circuit

#### Scenario: Genuinely divergent absorption

- **WHEN** applying an inbound message produces a state different from what the peer sent — because
  a domain rule adjusted it
- **THEN** the correction is still communicated back to the originating system

### Requirement: External identities are correlated, never substituted

The system SHALL store the mapping between our aggregate id and each peer's identifier in a
dedicated correlation table, unique in both directions, and SHALL NOT overwrite our primary key
with a foreign one.

#### Scenario: First message for an unknown external id

- **WHEN** a peer sends a record whose external id we have never seen
- **THEN** a correlation row is created against the aggregate the translator resolves or creates

#### Scenario: External id already bound elsewhere

- **WHEN** an external id already maps to a different aggregate
- **THEN** the message is dead-lettered as a correlation conflict rather than rebinding silently

### Requirement: Polling a peer advances its cursor only after durable insertion

If we poll a peer's feed, the system SHALL insert the fetched batch into the inbox durably and
SHALL advance the stored cursor only after that insert commits.

#### Scenario: Crash after fetch, before insert

- **WHEN** the poller crashes between the HTTP response and the insert
- **THEN** the cursor is unchanged and the batch is re-fetched

#### Scenario: Crash after insert, before processing

- **WHEN** the poller crashes after the insert commits
- **THEN** the cursor has advanced and the inbox worker processes the batch independently
