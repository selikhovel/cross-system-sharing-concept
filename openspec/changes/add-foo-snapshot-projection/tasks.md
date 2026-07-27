# Tasks — slice 1

Four commits. Each compiles and leaves the test suite green. Roughly a day and a half: half a day of
discovery, one day of code.

## 1. Discovery — no code

Half a day. Its output is a written artefact, and four of its five answers change what gets written.

- [ ] 1.1 Inventory every public member of `Foo`, recursing through value objects and child
      collections. Record it in the repository
- [ ] 1.2 For each value object, record whether it is an owned type or behind a single-column
      converter
- [ ] 1.3 Read the body of every collection property. Record any that is computed rather than a plain
      navigation — **this one cannot be caught by the tests in this slice**
- [ ] 1.4 Record whether a global query filter excludes soft-deleted rows
- [ ] 1.5 Write a keyset comparison on the aggregate id, enable SQL logging, run it, and **read the
      generated SQL**. Record whether it translates or evaluates client-side
- [ ] 1.6 Record column types: money as `numeric` or `double precision`, timestamps as `timestamptz`
      or `timestamp`, whether date-only fields are `date`
- [ ] 1.7 Confirm which fields are personal data, and whether the peer marks any of them required

**Done when** every question has a written answer and 1.5's SQL has been read by a person.

## 2. Projects and boundaries

- [ ] 2.1 Create `Catalog.Integration` and `Catalog.Integration.Contracts`; add a test project or a
      folder in the module's existing one
- [ ] 2.2 Reference `Catalog.Application` and `Catalog.Infrastructure` from `Catalog.Integration`.
      **No NuGet package in the production project**
- [ ] 2.3 Add test packages: xUnit (match the solution), `Shouldly`, `NetArchTest.Rules`, `Bogus`.
      Pin versions centrally
- [ ] 2.4 Create the folder skeleton — `Snapshots/`, `Mapping/`, `Paging/`, `Contracts/` — even where
      empty, so later slices land in the right place and the mechanism/policy seam is visible
- [ ] 2.5 Architecture tests: domain and application reference neither integration assembly nor
      `System.Net.Http`; no external contract type in a domain signature, application command, EF
      configuration or public API response

> Write 2.5 **before** there is anything to violate it. It is the only test whose cost rises the
> longer it is delayed. If it fails immediately, that is a finding about the existing codebase —
> report it, do not suppress it.

**Done when** `dotnet build` and `dotnet test` are green.

## 3. Snapshot and projection

- [ ] 3.1 `FooSnapshot` and its nested records — flat, no domain types, members `required` unless
      genuinely optional
- [ ] 3.2 `FooSnapshotProjection.ToSnapshot` as a **static
      `Expression<Func<Foo, FooSnapshot>>`**, not a method body
- [ ] 3.3 Order every projected collection by a stable business key, inside the expression
- [ ] 3.4 Apply explicitly any predicate found in a computed collection property (task 1.3)
- [ ] 3.5 Adjust for value objects behind a single-column converter (task 1.2): project the value
      object whole
- [ ] 3.6 Verify nothing normalises — no case change, rounding, unit conversion or formatting

**Done when** it compiles. A `required` member left unassigned is already a compile error.

## 4. Tests

- [ ] 4.1 `FooBuilder` calling the aggregate's **real factory methods**, with `Bogus` supplying
      values. Not AutoFixture
- [ ] 4.2 A shared assertion helper — slice 2 calls the same one against PostgreSQL
- [ ] 4.3 Fully populated aggregate: every member matches, collections in the documented order
- [ ] 4.4 Minimally populated aggregate: optional members null rather than zero or empty; collections
      empty rather than null
- [ ] 4.5 Reflection assertion: no snapshot member left at its type default when the source is fully
      populated
- [ ] 4.6 If task 1.3 found a computed collection: a test asserting an excluded row does not appear

**Done when** all tests are green.

## 5. Exit

- [ ] 5.1 `dotnet test` green, including the architecture tests
- [ ] 5.2 Discovery findings committed
- [ ] 5.3 **`git diff --stat` shows no migration and no change under the domain project**
- [ ] 5.4 Application behaviour unchanged — nothing wired into the host

> 5.3 is worth checking personally. The slice's whole claim is that it costs nothing to reverse, and
> a stray migration retires that claim quietly.

## Handover to slice 2

Slice 2 adds the store, keyset paging, the cursor, a hand-written stub contract and the two
endpoints, and **re-runs task 4's assertions through PostgreSQL**. That second run is what detects a
computed collection property, which the in-memory tests here cannot see.
