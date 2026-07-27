# Foo read API

Stage 1 of ADR-0007. Derived from ADR-0004 (the anti-corruption layer), ADR-0005 §3 (the security
rank), ADR-0006 §2 (the page contract) and `MAPPING_MATRIX.md`.

## ADDED Requirements

### Requirement: Foos are readable in the peer's contract shape

The system SHALL expose a single-aggregate read and a paged catalogue read that return `Foo`
data in the external contract shape, built on demand from current domain state.

Responses SHALL use the page envelope the target snapshot endpoint uses — an item collection, a next
cursor and a more-available flag — so that the consumer's parser does not change when the endpoint
gains a pinned watermark in stage 3.

The system SHALL NOT write any row, create any table, or require any migration to serve these
endpoints.

#### Scenario: Single Foo read

- **WHEN** an authorised caller requests an existing property by id
- **THEN** the response carries one item in the contract shape

#### Scenario: Unknown id

- **WHEN** an authorised caller requests an id that does not exist
- **THEN** the response is `404` and no partial payload is returned

#### Scenario: Catalogue read

- **WHEN** an authorised caller requests a page without a cursor
- **THEN** the response carries a page of items, a next cursor and a more-available flag

#### Scenario: No write side effects

- **WHEN** either endpoint is called
- **THEN** no integration table is written and no domain row is modified

### Requirement: Reads project directly into a flat snapshot

The system SHALL read domain data through a non-tracking projection into a flat snapshot read model,
and SHALL NOT rehydrate aggregates, use eager-loading chains, or trigger lazy loading.

#### Scenario: Page of one hundred Foos

- **WHEN** a page of one hundred Foos is served
- **THEN** the data is fetched by a bounded number of queries independent of page size
- **AND** no aggregate instance is materialised

#### Scenario: Projection shape is reused

- **WHEN** later stages materialise the same aggregate for the outbox
- **THEN** they use the same snapshot read model and the same projection query

### Requirement: Pagination is keyset-based and capped

The system SHALL paginate by keyset over a stable ordered key, exposed to the caller as an opaque
cursor, and SHALL NOT use offset pagination.

The page size SHALL have a documented default and a documented maximum, and a request above the
maximum SHALL be capped rather than rejected silently.

#### Scenario: Full catalogue walk

- **WHEN** a caller walks the whole catalogue page by page
- **THEN** every property is returned exactly once, with no duplicate and no skipped row

#### Scenario: Rows inserted mid-walk

- **WHEN** properties are inserted and deleted while a caller is walking pages
- **THEN** no previously unseen existing row is skipped and no row is returned twice within the walk

#### Scenario: Oversized page requested

- **WHEN** a caller requests a page size above the maximum
- **THEN** the response is capped at the maximum, and the cap is discoverable in the consumer
  contract

#### Scenario: Cursor is opaque

- **WHEN** a caller inspects a returned cursor
- **THEN** it carries no meaning the caller is invited to construct or increment

### Requirement: The mapping is total, and its totality is enforced by the build

The system SHALL map the snapshot to the contract through a compile-time mapper with unmapped source
and target members promoted to build errors, and SHALL account for every public domain field as
either mapped or explicitly excluded with a written reason.

> This is ADR-0004 §5 enforced from the first increment rather than retrofitted: the mapper, the
> exclusion registry and the coverage test are the artefacts every later stage reuses unchanged.

#### Scenario: Snapshot field left unmapped

- **WHEN** a property is added to the snapshot type and not mapped
- **THEN** the build fails with the unmapped-member diagnostic

#### Scenario: Domain field left unaccounted

- **WHEN** a property is added to `Foo` and is neither mapped nor listed as an exclusion
- **THEN** the coverage test fails, naming the field path

#### Scenario: Exclusion without a reason

- **WHEN** a field is excluded without a recorded reason in the mapping matrix
- **THEN** the coverage test fails

### Requirement: Values are translated without silent loss

The system SHALL apply the normative value rules — money as a decimal representation with an
explicit currency, measurements with an explicit unit and a documented rounding rule, timestamps as
ISO-8601 UTC with date-only fields kept date-only, table-driven fail-closed enums, deterministic
collection ordering, fixed coordinate precision, normalised phone numbers, and our own id as the
aggregate id — and SHALL fail a request rather than degrade a value.

#### Scenario: Value exceeds the peer's maximum length

- **WHEN** a title exceeds the peer's declared maximum length
- **THEN** the request fails for that property, naming the field, the actual length and the limit
- **AND** no truncated value is returned

#### Scenario: Unmapped enum value

- **WHEN** a domain enum value has no external mapping
- **THEN** the build fails, or the first request carrying it fails explicitly
- **AND** no fallback value is substituted

#### Scenario: Round-trip over generated values

- **WHEN** the round-trip property tests run over generated boundary values — money precision, unit
  conversion, non-ASCII text, daylight-saving boundaries, negative coordinates, null versus empty
  collection
- **THEN** every symmetric field round-trips exactly, and every registered asymmetric mapping
  converges

#### Scenario: Date-only field

- **WHEN** a date-only field is serialised
- **THEN** it carries no time component and no offset that could shift the date

### Requirement: Payloads validate against the vendored peer schema

The system SHALL validate generated payloads against the peer's vendored schema in the test suite,
and SHALL keep the vendored schema in the repository so that a partner change is reviewable as a
diff.

#### Scenario: Vendored schema present

- **WHEN** the peer's schema has been vendored
- **THEN** every generated payload in the test suite validates against it

#### Scenario: Peer removes a field we populate

- **WHEN** the vendored schema is updated and a field we map no longer exists
- **THEN** the build fails at the mapper

### Requirement: The endpoints ship authenticated, authorised and bounded

The system SHALL authenticate both endpoints, authorise them by a read scope, apply a per-caller
rate limit, and serve them over TLS only — from the first increment. Network location SHALL NOT be
treated as authentication.

Authentication SHALL sit behind a seam so the mechanism can change without touching the endpoints.

#### Scenario: Unauthenticated request

- **WHEN** a request arrives without credentials
- **THEN** it is rejected with `401` and no query is executed

#### Scenario: Wrong scope

- **WHEN** a caller without the read scope requests a page
- **THEN** the request is rejected with `403`

#### Scenario: Caller exceeds its rate

- **WHEN** a caller exceeds its configured rate
- **THEN** it receives `429` and other callers are unaffected

#### Scenario: Mechanism changes

- **WHEN** the authentication mechanism changes
- **THEN** only the seam's implementation and configuration change

#### Scenario: Credentials never logged

- **WHEN** a request is logged
- **THEN** the log records caller, route, page size, cursor, status and duration
- **AND** contains neither credentials nor payload content

### Requirement: Personal data is not served in stage 1

The system SHALL omit every field marked as personal data in the mapping matrix from every stage 1
response, unconditionally, because no subscriber registry exists yet to hold a grant.

#### Scenario: Foo with contact details

- **WHEN** a property whose contact has a name, phone and email is returned
- **THEN** none of those fields appear in the response

#### Scenario: New personal field added

- **WHEN** a field marked as personal data is added to the mapping matrix
- **THEN** the redaction test fails until it is omitted

### Requirement: The endpoints are reversible without a deploy

The system SHALL gate both endpoints behind a feature flag that defaults to off in production.

#### Scenario: Flag disabled

- **WHEN** the flag is off
- **THEN** both routes return `404` and no query is executed

#### Scenario: Emergency stop

- **WHEN** the flag is switched off at runtime
- **THEN** serving stops without a deployment and without affecting any other route

### Requirement: The domain stays unaware of the integration

The system SHALL keep the dependency direction one-way, verified by architecture tests added in this
stage while they are trivially green.

#### Scenario: Architecture tests run

- **WHEN** the architecture tests run
- **THEN** no type in the domain assembly references the integration assemblies or `System.Net.Http`
- **AND** no external contract type appears in a domain signature, an application command, an EF
  entity configuration or a public API response outside the integration routes

### Requirement: Stage 1's guarantees and non-guarantees are written down for the consumer

The system SHALL publish a consumer-facing contract before the endpoints are used, stating that
stage 1 offers **replace-all semantics only**: no version field, no incremental synchronisation, no
change notification, and no signal that a record has been deleted — together with the page contract,
the page cap, the rate limit and the fact that stage 1 is time-boxed.

> Defect **D13**. Four places in the ADRs refer to a consumer-facing contract that does not exist.
> Stage 1 is the first moment we make a promise to another team, so it is the moment the document
> starts.

#### Scenario: Record deleted on our side

- **WHEN** a property is deleted and a consumer re-reads only changed pages
- **THEN** nothing in the response indicates the deletion
- **AND** the consumer contract states that only a full replace is correct in stage 1

#### Scenario: Consumer asks for incremental sync

- **WHEN** a consumer asks how to fetch only what changed
- **THEN** the contract states that stage 1 does not offer it and names the stage that will

#### Scenario: Version field added in stage 2

- **WHEN** `aggregateVersion` is added to the response envelope
- **THEN** it is an additive change requiring no contract version bump
- **AND** the consumer contract is updated in the same change
