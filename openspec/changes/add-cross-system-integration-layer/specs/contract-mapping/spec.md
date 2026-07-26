# Contract mapping (anti-corruption layer)

Derived from ADR-0004, `MAPPING_MATRIX.md` and `TECHNICAL_PROPOSAL.md` §7.

## ADDED Requirements

### Requirement: Three isolated model layers translated only at the boundary

The system SHALL keep three separate model layers — domain aggregates, internal flat snapshots, and
contract types generated from the peer's schema — and SHALL translate only at the boundaries
between them.

External contract types SHALL NOT appear in a domain signature, an application command, an EF
entity or a public API response.

#### Scenario: Architecture tests run

- **WHEN** the architecture tests run
- **THEN** the domain assembly references nothing from the integration assemblies
- **AND** the application assembly references nothing from the external contracts assembly
- **AND** no external contract type appears in any EF entity configuration or public API response

#### Scenario: The peer ships schema version two

- **WHEN** a second external contract version is added
- **THEN** only the snapshot-to-contract mappers change
- **AND** the domain-to-snapshot projections are untouched

### Requirement: Contract types are generated from a vendored schema

The system SHALL vendor the peer's schema into the repository, generate contract types from it at
build time, and fail a scheduled job when the peer's live schema differs from the vendored copy.

#### Scenario: Peer removes a field

- **WHEN** the vendored schema is updated and a field we populate no longer exists
- **THEN** the build fails at the mapper

#### Scenario: Peer changes the schema without telling us

- **WHEN** the drift job fetches the live schema and finds a difference
- **THEN** the job fails and raises a ticket alert
- **AND** the difference is reviewable as a diff against the vendored file

### Requirement: Snapshot envelope as the default payload shape

The system SHALL emit, by default, the full current state of an aggregate inside an envelope
carrying at least: `messageId`, `messageType`, `contractVersion`, `occurredAt`, `producedAt`,
`source`, `aggregateType`, `aggregateId`, `aggregateVersion`, `changeKind`, `correlationId`,
`traceParent`, and the propagation path used for loop prevention.

If the peer mandates delta payloads, compaction SHALL be disabled, per-aggregate ordered delivery
SHALL become mandatory, a per-aggregate snapshot endpoint SHALL be added for bootstrap, and the
envelope SHALL additionally carry the previous version.

> Defect **D6**. ADR-0004 §3's envelope has no field able to carry inbound provenance: its `source`
> is always `acme`. Echo suppression referenced in `TECHNICAL_PROPOSAL.md` §6.2 has no
> envelope field to read.

#### Scenario: Snapshot message emitted

- **WHEN** a `Foo` change is materialised under the default shape
- **THEN** the message carries the full mapped state and every envelope field listed above

#### Scenario: Message lost in transit

- **WHEN** a message is never received and a later message for the same aggregate is
- **THEN** the consumer converges on correct state from the later message alone

### Requirement: Losslessness is enforced by the build and the test suite

The system SHALL make "every domain field is mapped or explicitly excluded with a reason" a
build-time and test-time guarantee, not a review-time assertion.

| Guarantee | Enforced by | Fails at |
|---|---|---|
| Unmapped source or target member | mapper diagnostics promoted to errors | compile |
| Domain field neither mapped nor excluded | reflection coverage test naming the field | test |
| Silent value corruption | round-trip property tests over generated values | test |
| Enum drift | exhaustive switch with no value-returning default | compile / first message |
| Peer schema drift | vendored schema plus scheduled diff job | CI |
| Invalid payload | schema validation at materialisation | before delivery |

#### Scenario: Field added to a snapshot without a mapping

- **WHEN** a property is added to a snapshot type and not mapped
- **THEN** the build fails with the unmapped-member diagnostic

#### Scenario: Field added to a domain aggregate

- **WHEN** a property is added to `Foo` and neither mapped nor listed as an exclusion
- **THEN** the coverage test fails, naming the field path

#### Scenario: Exclusion without a reason

- **WHEN** a field is excluded without a recorded reason in the mapping matrix and the exclusion
  registry
- **THEN** the coverage test fails

#### Scenario: Value corruption introduced

- **WHEN** a mapper change loses decimal precision or shifts a time zone
- **THEN** a round-trip property test fails on generated boundary values

### Requirement: Value translation rules are normative

The system SHALL apply the following rules, and SHALL fail rather than degrade data.

| Concern | Rule |
|---|---|
| Money | Never floating point. Decimal string or minor units plus an explicit ISO-4217 currency. `null` is not `0`. |
| Measurements | Value plus explicit unit. Conversions rounded half-away-from-zero to two decimal places, never written back into the domain. |
| Timestamps | ISO-8601 UTC with offset. Date-only fields stay date-only. |
| Enums | Table-driven and fail-closed, except where a row explicitly permits dropping. |
| Strings | Trimmed and length-checked against the peer's constraint. Truncation is an error. |
| Nulls | `null` means explicitly cleared; an absent key means unchanged. Snapshots emit all keys. |
| Collections | Sorted deterministically by a stable business key, never by database order. |
| Coordinates | Fixed six-decimal precision with the spatial reference stated. |
| Phone numbers | Normalised to E.164 with a configured default region. |
| Identity | Our id in `aggregateId`; peer ids only in the external-ids collection; never overwrite our key. |

#### Scenario: Value exceeds the peer's maximum length

- **WHEN** a title exceeds the peer's declared maximum length
- **THEN** materialisation fails for that aggregate and dead-letters it, naming the field, the
  actual length and the limit
- **AND** no truncated value is sent

#### Scenario: Unmapped enum value

- **WHEN** a new domain enum value is added and not mapped
- **THEN** the build fails, or the first message carrying it dead-letters
- **AND** no fallback value is substituted

#### Scenario: Unmapped optional feature

- **WHEN** a feature tag has no external equivalent
- **THEN** it is dropped, the unmapped-value counter increases for that tag, and the daily
  threshold alert can fire

#### Scenario: Many-to-one status mapping

- **WHEN** two domain statuses map to one external status
- **THEN** the asymmetry is registered and the round-trip test asserts convergence, not equality

#### Scenario: Area unit conversion

- **WHEN** the peer expects square feet and we store square metres
- **THEN** the converted value carries an explicit unit and matches the documented rounding rule

### Requirement: Deletions have their own message type and schema

The system SHALL define how a deletion is represented in the peer's contract, and SHALL validate a
tombstone against that representation's schema rather than against the full-snapshot schema.

> Defect **D11**. `TECHNICAL_PROPOSAL.md` §5.2 validates every payload against the peer's schema and
> dead-letters on failure, ADR-0004 §3 defines `data` as a fully populated Foo, and ADR-0002
> emits a tombstone of `{id, deletedAt}`. Against a schema with required fields — and
> `MAPPING_MATRIX.md` §2 marks ten fields required — every hard delete and every retraction
> tombstone dead-letters. Deletes are one of the design's stated strengths and as specified cannot
> be delivered at all.

#### Scenario: Tombstone validated

- **WHEN** a tombstone is materialised
- **THEN** it is validated against the deletion schema and passes
- **AND** it is not dead-lettered for absent snapshot fields

#### Scenario: Peer has no deletion message

- **WHEN** the peer's contract offers no way to express a deletion
- **THEN** this is recorded as an unresolved contract gap and the retraction rules are blocked
  rather than emitting an unvalidated payload

### Requirement: Versioning is explicit in three places

The system SHALL express the contract version in the URL path, in an envelope field and in the media
type; SHALL treat additive optional fields as non-breaking; SHALL treat removals, renames and
semantic changes as a new major version; and SHALL support emitting two versions concurrently for
the same change during a migration.

#### Scenario: Additive change

- **WHEN** an optional field is added to the contract
- **THEN** no version is bumped and existing consumers are unaffected

#### Scenario: Breaking change with two subscribers

- **WHEN** one subscriber is on version 1 and another on version 2
- **THEN** the same domain change produces one message per version, each at the subscriber's
  version

#### Scenario: Unknown field received inbound

- **WHEN** a peer adds a field we do not know
- **THEN** deserialisation ignores it, asserted by a test

#### Scenario: Deprecation window

- **WHEN** a version is deprecated
- **THEN** the window is tracked per subscriber and is at least the documented minimum

### Requirement: Serialisation is deterministic and canonical

The system SHALL use one shared, source-generated serialiser configuration with a naming policy
matching the peer's schema, explicit nulls for snapshots, and strict number handling; and SHALL
provide a canonical form — stable key order, normalised numbers and dates — used for both payload
hashing and reconciliation checksums.

The canonical form SHALL be version-pinned, and a change to it SHALL be treated as a contract
change.

#### Scenario: Same state hashed twice

- **WHEN** the same aggregate state is materialised twice
- **THEN** the canonical payload bytes and the hash are identical

#### Scenario: Canonicaliser changed

- **WHEN** the canonical form is modified
- **THEN** the change is versioned and treated as a contract change, because every stored hash and
  every reconciliation comparison shifts with it
