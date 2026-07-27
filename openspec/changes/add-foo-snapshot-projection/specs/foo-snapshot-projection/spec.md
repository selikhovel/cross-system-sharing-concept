# Foo snapshot projection

Slice 1 of the contract-shaped read API: the single canonical way from the domain aggregate to a
flat, serialisable snapshot, asserted without a database.

## ADDED Requirements

### Requirement: Discovery findings are recorded before the projection is written

The team SHALL record, as a written artefact in the repository, the answers to the five questions
that change the projection's implementation: the full public member inventory of `Foo`; whether each
value object is mapped as an owned type or through a single-column converter; whether any collection
property is computed rather than a plain navigation; whether a global query filter excludes
soft-deleted rows; and whether a keyset comparison on the aggregate id translates to SQL.

Column types for money, timestamps and date-only fields SHALL be recorded at the same time.

> Four of these have been observed to force a rewrite when found late, and one — the computed
> collection — cannot be found by the tests in this slice at all.

#### Scenario: Findings are available to a reviewer

- **WHEN** the slice is reviewed
- **THEN** each of the five questions has a written answer in the repository
- **AND** the generated SQL for the keyset comparison has been read by a person, not assumed

#### Scenario: A collection property is computed

- **WHEN** discovery finds a collection exposed as a computed property with a filter in its body
- **THEN** the projection applies that predicate explicitly
- **AND** a test asserts a row excluded by the predicate does not appear in the snapshot

#### Scenario: A value object is behind a single-column converter

- **WHEN** discovery finds a value object mapped to one column
- **THEN** the projection reads the value object whole rather than its sub-properties

### Requirement: The integration projects exist with a one-way reference direction

The system SHALL place integration code in its own project inside the module that owns the
aggregate, referencing the module's application and infrastructure projects, and SHALL keep the
peer's generated contract types in a separate project.

The production integration project SHALL take **no NuGet package dependency** in this slice.

#### Scenario: Solution builds

- **WHEN** the solution is built
- **THEN** the integration project compiles and references only the module's own projects

#### Scenario: Contract types are separated

- **WHEN** the peer's generated types are added in a later slice
- **THEN** they land in their own project, so that a reference from the domain or application layer
  is a compile error rather than a convention

### Requirement: The dependency direction is asserted by test

The system SHALL assert, by automated test, that the domain and application layers reference neither
the integration assemblies nor `System.Net.Http`, and that no external contract type appears in a
domain signature, an application command, an EF entity configuration, or a public API response
outside the integration routes.

These tests SHALL be written in this slice, while they are trivially green.

#### Scenario: Architecture tests run

- **WHEN** the test suite runs
- **THEN** the domain assembly references neither integration assembly nor `System.Net.Http`
- **AND** the application assembly references no external contract type

#### Scenario: A pre-existing violation is found

- **WHEN** the tests are added and fail immediately
- **THEN** the failure is reported as a finding about the existing codebase
- **AND** it is not silently suppressed to make the slice pass

### Requirement: A flat snapshot read model carries only what leaves the service

The system SHALL define a flat record type containing exactly the fields that are published, using no
domain types, no nested aggregates and no behaviour.

Members SHALL be `required` unless genuinely optional, so that a member left unassigned in the
projection is a compile error.

#### Scenario: Snapshot contains no domain type

- **WHEN** the snapshot type is inspected
- **THEN** every member is a primitive, a string, a date type, or another flat snapshot record

#### Scenario: A member is added and not projected

- **WHEN** a `required` member is added to the snapshot and the projection is not updated
- **THEN** the build fails

### Requirement: One canonical projection expression, translatable to SQL

The system SHALL express the domain-to-snapshot projection as a **single static
`Expression<Func<Foo, FooSnapshot>>`**, not as a method body, so that the same definition is reused
by every caller.

The expression SHALL contain only constructs EF Core can translate: member access, null checks,
conditional expressions, and ordered collection projections.

> The reuse is the point. A background materializer calls this same definition in a later stage; a
> projection hidden inside a method gets copied, and the copies drift apart on the question of what
> leaves the service.

#### Scenario: Reused by a second caller

- **WHEN** a later stage materialises the same aggregate outside a web request
- **THEN** it calls this expression rather than defining its own

#### Scenario: Query is composed over the expression

- **WHEN** a query applies the expression through `Select`
- **THEN** the generated SQL selects columns rather than materialising the aggregate

### Requirement: The projection does not normalise values

The projection SHALL copy values without transformation: no case changes, no rounding, no unit
conversion, no formatting, no truncation.

> Normalisation belongs to the anti-corruption mapper in a later slice. Keeping the projection a dumb
> copy is what allows it to be tested before that mapper exists, and what keeps "did we read the
> right column" separable from "did we format it right".

#### Scenario: String values pass through

- **WHEN** a text field containing mixed case and surrounding whitespace is projected
- **THEN** the snapshot holds it byte for byte

#### Scenario: Numeric values pass through

- **WHEN** a decimal is projected
- **THEN** it is neither rounded nor converted to another unit

### Requirement: Collections are ordered deterministically inside the projection

The projection SHALL order every collection by a stable business key, expressed inside the
expression so the ordering is performed by the database.

#### Scenario: Same aggregate projected twice

- **WHEN** the same aggregate is projected twice
- **THEN** collection members appear in the same order both times

#### Scenario: Ordering is not left to the database's discretion

- **WHEN** the generated SQL is inspected
- **THEN** it carries an explicit `ORDER BY` for each projected collection

### Requirement: The projection is asserted in memory against a constructed aggregate

The system SHALL verify the projection by compiling the expression and invoking it on aggregates
built through their **real factory methods**, without a database.

Test aggregates SHALL be constructed by a hand-written builder rather than by a reflection-based
auto-fixture library, because an aggregate assembled around its own invariants proves nothing.

#### Scenario: Fully populated aggregate

- **WHEN** a fully populated aggregate is projected
- **THEN** every snapshot member equals the corresponding domain value
- **AND** collections are present in the documented order

#### Scenario: Minimally populated aggregate

- **WHEN** an aggregate with every optional value absent is projected
- **THEN** optional members are null rather than a zero or empty substitute
- **AND** collections are empty rather than null

#### Scenario: A member is projected from the wrong source

- **WHEN** a fully populated aggregate is projected
- **THEN** a reflection assertion confirms no snapshot member is left at its type default

### Requirement: The in-memory assertions are reusable by the database test that follows

The assertions in this slice SHALL live in a shared helper, so the slice that runs the projection
against PostgreSQL asserts **the same things** rather than a restatement of them.

> Compiling the expression executes the real property bodies, so a computed collection keeps its
> filter in memory and loses it under EF translation. The in-memory test is necessary and not
> sufficient; the divergence between the two runs is what detects that trap. Where they disagree, the
> in-memory result is the one that is wrong about production.

#### Scenario: The database test is added

- **WHEN** the projection is later exercised against a real database
- **THEN** it calls the same assertion helper
- **AND** any disagreement between the two runs fails the build rather than being reconciled by hand

### Requirement: The slice changes no schema and no domain code

The system SHALL introduce no migration, no table, no index and no column, and SHALL make no change
to the domain project.

> Slice 1's claim is that it costs nothing to reverse. A stray migration or a "small" domain tweak
> quietly retires that claim, which is why this is a requirement rather than an intention.

#### Scenario: Reviewing the change

- **WHEN** the diff for this slice is reviewed
- **THEN** it contains no migration file
- **AND** no file under the domain project is modified

#### Scenario: Nothing is wired into the host

- **WHEN** the application starts
- **THEN** behaviour is unchanged, because no endpoint, worker or interceptor has been registered
