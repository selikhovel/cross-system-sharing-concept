# Integration security

Derived from ADR-0005 and `TECHNICAL_PROPOSAL.md` §8.

## ADDED Requirements

### Requirement: Every integration endpoint is authenticated and scope-authorised

The system SHALL authenticate every integration endpoint in both directions and SHALL authorise by
capability scope, not by service identity alone. Network location SHALL NOT be treated as
authentication.

| Capability | Grants |
|---|---|
| feed read | reading the change feed and single-aggregate reads |
| inbound write | posting inbound messages |
| backfill read | reading snapshot pages and checksums |
| admin | dead-letter replay, subscriber management, payload inspection |

#### Scenario: Unauthenticated request

- **WHEN** a request arrives without credentials
- **THEN** it is rejected with `401` and no work is performed

#### Scenario: Valid token, wrong scope

- **WHEN** a caller holding only the feed-read scope posts an inbound message
- **THEN** the request is rejected with `403`

#### Scenario: Valid token from an unrelated service

- **WHEN** a token valid for the company but issued to a client not on the endpoint's allowlist is
  presented
- **THEN** the request is rejected

### Requirement: The authentication mechanism is swappable behind a seam

The system SHALL implement authentication behind a single abstraction so the mechanism can change
without touching the dispatcher, the ingress handler or the workers.

The recommended mechanism SHALL be OAuth 2.0 client credentials with IdP-issued JWTs; the fallback
ladder SHALL be mutual TLS, then API key with HMAC request signing, then — time-boxed only — a bare
static key recorded as technical debt.

#### Scenario: Mechanism changes

- **WHEN** the mechanism moves from OAuth 2.0 to mutual TLS
- **THEN** only the seam's implementation and configuration change
- **AND** no dispatcher, ingress or worker code changes

#### Scenario: HMAC fallback in use

- **WHEN** the HMAC tier is selected
- **THEN** the signature covers timestamp, method, path and a body digest
- **AND** a timestamp outside the accepted window is rejected
- **AND** a replayed signature identifier is rejected
- **AND** comparison is constant-time

#### Scenario: Bare static key in use

- **WHEN** the bare-key tier is selected
- **THEN** it carries a written expiry date and a tracked ticket

### Requirement: Outbound credentials never reach the dispatcher

The system SHALL acquire and cache peer credentials in a delegating handler, refresh them before
expiry, and keep them out of dispatcher code, out of the database and out of logs. The subscriber
row SHALL hold a reference to a secret, never a secret.

#### Scenario: Token refresh

- **WHEN** a cached token approaches its expiry
- **THEN** it is refreshed before use without failing an in-flight delivery

#### Scenario: IdP unavailable

- **WHEN** the identity provider is unreachable
- **THEN** deliveries are delayed and retried, and no message is lost

#### Scenario: Secret in the database

- **WHEN** the subscriber table is inspected
- **THEN** it contains only a secret-manager reference

### Requirement: Transport security is never relaxed

The system SHALL use TLS 1.2 or higher on every hop, including inside the cluster, and SHALL NOT
disable certificate validation in any environment.

#### Scenario: Peer presents a self-signed certificate

- **WHEN** a peer's certificate is not chained to a trusted root
- **THEN** the connection fails and the remedy is adding the peer's CA to the trust store
- **AND** no validation callback is introduced

#### Scenario: Plaintext endpoint configured

- **WHEN** a subscriber callback URL uses a non-TLS scheme
- **THEN** registration is rejected

### Requirement: Secrets and payloads never reach logs

The system SHALL redact authorisation headers, signatures, API keys and any property whose name
suggests a token, secret or password; and SHALL NOT log payload bodies.

Payload inspection SHALL be available only through an admin-scoped endpoint, and that access SHALL
itself be audit-logged.

#### Scenario: Failed delivery logged

- **WHEN** a delivery fails
- **THEN** the log records message id, aggregate identity, subscriber, status code and duration
- **AND** contains neither the payload nor any credential

#### Scenario: Redaction verified

- **WHEN** the redaction test runs with a token present in a logged object graph
- **THEN** the token does not appear in the output

#### Scenario: Operator inspects a payload

- **WHEN** an operator with the admin scope reads a dead-lettered payload
- **THEN** the access is recorded in the audit log

### Requirement: Ingress is bounded

The system SHALL bound request body size, batch item counts and per-caller request rate on every
inbound endpoint.

#### Scenario: Oversized body

- **WHEN** a body exceeds the configured limit
- **THEN** the request is rejected before the body is fully buffered

#### Scenario: Caller exceeds its rate

- **WHEN** a caller exceeds its configured rate
- **THEN** it receives `429` with a retry hint, and other callers are unaffected

### Requirement: Callback URLs cannot be turned into a request forgery primitive

The system SHALL take callback URLs only from subscriber configuration, validate them against an
allowlist of internal hosts at registration **and** at dispatch, and SHALL NOT follow redirects.

#### Scenario: Callback outside the allowlist

- **WHEN** a subscriber row is written with a host outside the allowlist
- **THEN** the write is rejected

#### Scenario: Allowlist changed after registration

- **WHEN** a previously valid host is removed from the allowlist
- **THEN** dispatch to it fails validation rather than proceeding on the stale value

#### Scenario: Peer responds with a redirect

- **WHEN** a peer answers `302` pointing at an internal service
- **THEN** the redirect is not followed and the response is classified as a failure

### Requirement: Personal data is gated per subscriber and bounded in time

The system SHALL emit fields marked as personal data only to subscribers holding an explicit grant,
SHALL assert that every such field has a recorded grant decision, SHALL bound how long payloads
containing personal data are retained, and SHALL provide a documented routine that purges a
subject's data from integration payloads on an erasure request.

#### Scenario: Subscriber without a grant

- **WHEN** an aggregate carrying contact details is delivered to a subscriber without the
  grant
- **THEN** the payload contains no personal field, verified by test

#### Scenario: New personal field added

- **WHEN** a field marked as personal data is added and no grant decision is recorded
- **THEN** the coverage test fails

#### Scenario: Erasure request

- **WHEN** an erasure request is executed for a subject
- **THEN** matching outbox, inbox and dead-letter payloads are purged
- **AND** the operation is audit-logged

#### Scenario: Retention respected

- **WHEN** the retention job runs
- **THEN** payloads past their window are removed, so the outbox is not an indefinite copy of
  personal data
