# ADR-0005 — Authentication, authorisation and transport security

- **Status:** Proposed — **requires platform/security team confirmation before implementation**
- **Date:** 2026-07-25
- **Open question:** the company's available service-to-service auth mechanism is not yet
  decided. This ADR recommends one and specifies a fallback ladder.
- **Related:** ADR-0001, ADR-0003

## Context

Both directions of the integration cross a service boundary and carry data that is at least
commercially sensitive (pricing, ownership, addresses) and, depending on the aggregate,
personal data (owner and contact details) subject to GDPR-equivalent handling.

Nothing about "it is inside the corporate network" makes an unauthenticated endpoint
acceptable — a webhook receiver that trusts its caller is a write path into our domain model
available to anything that can reach the pod.

The team has not yet fixed the auth mechanism, so this ADR states a recommendation, a
fallback ladder, and the non-negotiables that hold regardless of which is chosen.

## Decision

### 1. Recommended: OAuth 2.0 client credentials against the corporate IdP

**Primary recommendation: OAuth 2.0 `client_credentials` (RFC 6749 §4.4) with JWT access
tokens issued by the company IdP (Keycloak / Entra ID / equivalent), scoped per capability.**

Why this first:

- **It is the mechanism the rest of the company can already consume.** Every peer service can
  obtain a token; nobody needs a bespoke integration with us.
- **Rotation is solved by the IdP** — short-lived tokens (≤ 1 h), no long-lived shared secret
  sitting in two config stores drifting apart.
- **Authorisation is expressible**, not just authentication: scopes let us grant a peer
  read access to the property feed without granting write access to the inbound endpoint.
- **Revocation is centralised** — disabling a client at the IdP cuts access immediately.
- It is auditable centrally, which is what a security review will ask for first.

Scope design (least privilege, per capability, not per service):

| Scope | Grants |
|---|---|
| `acme.feed.read` | `GET /integration/v1/*/changes`, `GET /integration/v1/*/{id}` |
| `acme.inbound.write` | `POST /integration/v1/inbound/*` |
| `acme.backfill.read` | `GET /integration/v1/backfill/*` |
| `acme.admin` | dead-letter replay, subscriber management — humans/ops only |

Outbound (us → peer): we are the client. Tokens are acquired per subscriber, cached in memory
with refresh at 80 % of lifetime, and never logged. Implementation is a
`DelegatingHandler` on the typed `HttpClient`, so the dispatcher never handles credentials.

Validation on inbound requests: signature via the IdP's JWKS (cached, auto-refreshed),
plus explicit `issuer`, `audience`, `exp`, `nbf` checks and an **allowlist of accepted
`client_id`s per endpoint** — a valid token from an unrelated company service must not be
sufficient to POST into our domain.

### 2. Fallback ladder (if OAuth 2.0 is unavailable)

**Tier 2 — mTLS.** Fully acceptable, and *preferable* if a service mesh (Istio/Linkerd) already
terminates it: identity is the client certificate, no token plumbing at all. Standalone mTLS
without a mesh means running certificate issuance, distribution and rotation ourselves —
that is the part that fails in practice (expired certs at 2 a.m.), so prefer it only with mesh
or an existing internal PKI with automated renewal (cert-manager / Vault PKI).
The application still authorises on the certificate subject/SAN against an allowlist; TLS
termination alone is authentication, not authorisation.

**Tier 3 — API key + HMAC request signing.** The pragmatic fallback if neither of the above
exists. Non-negotiable if chosen:
- Key stored in the corporate secret manager (Vault / Azure Key Vault), injected as an env
  var or mounted file, **never in appsettings, never in git**.
- **HMAC-SHA256 signature over `timestamp + method + path + sha256(body)`**, sent in
  `X-Signature`, with a ±5 minute `X-Timestamp` window and a replay cache of seen signature
  ids. A bare API key in a header is replayable by anything that reads one log line; the
  signature is what makes it defensible.
- Constant-time comparison (`CryptographicOperations.FixedTimeEquals`).
- Documented 90-day rotation with dual-key overlap (accept old and new during the window).

**Tier 4 — bare static API key.** Only acceptable as a time-boxed bridge with a written
expiry date and a ticket. Record it as technical debt in the proposal's risk register.

### 3. Non-negotiables regardless of mechanism

- **TLS 1.2+ everywhere**, including inside the cluster. No plaintext HTTP hop, not even a
  "trusted" one behind the mesh.
- **Certificate validation is never disabled.** No `ServerCertificateCustomValidationCallback`
  returning `true`, in any environment, ever. If a peer has a self-signed cert, add their CA
  to the trust store.
- **Secrets never in source control, appsettings, logs, exception messages or the outbox.**
  A `Serilog` destructuring policy redacts `Authorization`, `X-Signature`, `X-Api-Key` and any
  property named `*token*`, `*secret*`, `*password*`.
- **The inbound endpoint is not a domain write path.** It authenticates, validates against the
  schema, and inserts into `inbox_message`. Domain mutation happens in a worker after the
  request is over — so an authenticated-but-malicious payload gets a validation failure and a
  dead-letter row, not partial domain state.
- **Request limits** on ingress: `MaxRequestBodySize` (1 MB default; explicit larger limit only
  where justified), per-client rate limiting (`Microsoft.AspNetCore.RateLimiting`, fixed window
  by `client_id`), and a hard cap on batch item counts.
- **SSRF protection outbound:** subscriber callback URLs are **not** free-form. They come from
  the `integration.subscriber` table, are validated against an allowlist of internal hosts at
  registration time, and re-validated at dispatch. No redirect following
  (`AllowAutoRedirect = false`) — a 302 to an internal admin service is the classic escalation.
- **PII minimisation.** Fields flagged `pii` in MAPPING_MATRIX.md are emitted only to
  subscribers whose entry carries the `pii` grant. The materializer produces a redacted variant
  otherwise, and the field-coverage test asserts every PII field has an explicit grant decision.
- **Payload retention is bounded** by ADR-0003 §8 — the outbox is not an indefinite copy of
  personal data. A GDPR erasure request must also purge matching outbox/inbox payloads; a
  documented `integration.purge_subject(subjectId)` routine covers this.

### 4. Observability without leaking

Structured logs record `messageId`, `aggregateType`, `aggregateId`, `subscriberId`, status
code and duration — **never the payload body**. Payload access is available on demand via an
admin endpoint requiring `acme.admin`, and that access is itself audit-logged.
W3C `traceparent` is propagated end to end so a message can be followed across services without
anyone needing to read its contents.

## Alternatives considered

- **No auth, network policy only.** Rejected: a single misconfigured NetworkPolicy or a
  compromised neighbouring pod yields full write access to our domain. Defence in depth is
  cheap here.
- **Basic auth over TLS.** Long-lived credentials, replayable, no scoping, appears in any tool
  that logs URLs. Rejected.
- **Shared JWT signing secret between peers (HS256 symmetric).** Every holder can mint tokens
  for every other party. Rejected in favour of asymmetric IdP-issued tokens.
- **Per-subscriber IP allowlisting as the primary control.** Brittle in Kubernetes, and grants
  identity to a network location rather than to a service. Retained only as a secondary control.

## Consequences

### Positive

- Central, revocable, rotatable identity for every party; least-privilege scopes.
- Compromise of one peer does not grant access to the rest of the surface.
- Replay and SSRF — the two attacks this shape of integration actually invites — are addressed
  explicitly rather than assumed away.
- PII exposure is a per-field, per-subscriber decision recorded in a reviewable table.

### Negative / costs accepted

- **Dependency on the IdP's availability** for outbound dispatch. Mitigated by token caching and
  by the outbox itself: IdP downtime delays delivery, it never loses messages.
- **Onboarding a subscriber becomes a process** (IdP client, scope grant, allowlist entry, PII
  decision) rather than a config line. Intentional.
- **Operational work**: JWKS caching, clock skew tolerance, rotation runbooks.
- **The decision is still open**; if the fallback ladder lands on Tier 3/4, the HMAC and
  rotation work is roughly 2–3 extra days and must be scheduled.
