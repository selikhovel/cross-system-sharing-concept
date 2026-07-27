# Design — slice 1

Everything an implementer needs, without reference to documents outside this folder.

The service is ASP.NET Core on Clean Architecture with CQRS and DDD, EF Core and PostgreSQL 17,
organised as a modular monolith with one module in use.

---

## 1. Discovery first — five findings that change the code

**Do this before writing the projection.** Four of the five have been observed to force a rewrite
when discovered late. Record the answers in the repository; they are inputs to later slices too.

### 1.1 Inventory every public member of `Foo`

Recurse through value objects and child collections. This list becomes the snapshot and, later, the
field-coverage test. A field nobody wrote down is a field nobody decided about.

### 1.2 How are value objects mapped?

Look at the EF configuration. Two possibilities, and they lead to different projections:

- **Owned type** (`OwnsOne`) → sub-properties are separate columns and project individually:
  `Price = p.Price.Amount` translates.
- **Single-column converter** (`HasConversion`, JSON) → sub-properties **cannot** be projected.
  Project the value object whole and flatten later. The read is wider; correctness is unchanged.

### 1.3 Are collection properties computed?

This one silently returns wrong data. Read the property body:

```csharp
// DANGEROUS
public IEnumerable<FooAttachment> Attachments => _attachments.Where(a => a.IsVisible);
```

In a LINQ projection, EF resolves `f.Attachments` to the **navigation of that name** and never
evaluates the body — so the filter disappears and hidden rows are published. Encapsulation that looks
correct produces incorrect output.

If any collection is computed, apply its predicate **explicitly** in the projection, and add a test
asserting a hidden row does not appear.

### 1.4 Is there a global query filter?

`HasQueryFilter` for soft deletes. If one exists, deleted rows are already excluded; if not, filter
explicitly or retracted records get published.

### 1.5 Does a keyset comparison translate to SQL?

Not needed until slice 2, but the answer shapes the projection's id handling, so ask now. Write a
query comparing on the id, enable `LogTo(Console.WriteLine)`, run it and **read the generated SQL**.
Expected: `WHERE f.id > $1 ORDER BY f.id LIMIT $2`. If the id is a strongly-typed value with a
converter this may fail to translate or evaluate client-side; record which.

Also record column types: `numeric` versus `double precision` for money, `timestamptz` versus
`timestamp`, whether date-only fields are `date`. They drive slice 4 and are free to collect now.

---

## 2. Where the code goes

```
Modules/Catalog/
├── Catalog.Domain            unchanged
├── Catalog.Application       unchanged
├── Catalog.Infrastructure    unchanged (referenced for the DbContext)
├── Catalog.Integration            ← new
│   ├── Snapshots/                 policy — knows about Foo
│   ├── Mapping/                   (empty until slice 4)
│   ├── Paging/                    (empty until slice 2)
│   └── Contracts/                 (empty until slice 2)
└── Catalog.Integration.Contracts  ← new, empty until slice 5
```

`Catalog.Integration` references `Catalog.Application` and `Catalog.Infrastructure`. Both live in the
infrastructure ring, so this is a peer reference inside one ring, not an inversion. Use a port in
`Integration` implemented in `Infrastructure` only if the `DbContext` is `internal` to that assembly.

**Folder discipline matters even though there is one module.** Later stages add `Outbox/`,
`Delivery/` and `Inbox/`, which are mechanism and know nothing about `Foo`. Keeping mechanism and
policy in separate folders means extracting a shared building block — when a second module actually
needs one — is a move rather than a rewrite. Do not extract it now: an abstraction with no second
consumer gets redesigned the moment a real one appears.

### Why this is not a CQRS query handler

The reflex in a CQRS codebase is `GetFooSnapshotQuery : IQuery<FooSnapshot>` in `Application`.
Do not.

- The snapshot's shape is dictated by the **peer's contract**, not by a use case. Putting it in
  `Application` puts a foreign schema's shadow into the layer that owns our use cases.
- The read has no business logic, no authorisation decision beyond the endpoint scope, and no
  invariant to uphold. The handler would be a pass-through.
- Stage 3's materializer calls the same projection from a `BackgroundService` where there is no
  request and no pipeline to route through.

Route through `Application` only if the read needs something `Application` owns — a permission check,
a tenant filter, a computed value from a domain service. Then the query returns a domain-shaped
result and `Integration` maps it, keeping the contract type out of `Application`'s signature.

---

## 3. The projection

Two rules that are not style preferences.

### 3.1 A static `Expression`, not a method body

```csharp
public static class FooSnapshotProjection
{
    public static readonly Expression<Func<Foo, FooSnapshot>> ToSnapshot =
        f => new FooSnapshot { /* ... */ };
}
```

EF translates an `Expression` to SQL, selecting only the needed columns without materialising the
aggregate. And stage 3's materializer calls **this same definition**. Buried in a method body it gets
copied, and two copies of the rule "what leaves the service" drift apart.

A `required` member left unassigned in the object initialiser is a **compile error** — that is the
cheapest correctness guard available, so make every member of `FooSnapshot` `required` unless it is
genuinely optional.

### 3.2 The projection does not normalise

No `ToLowerInvariant`, no rounding, no unit conversion, no formatting. Those belong to the mapper in
slice 4. Keeping the projection a dumb column copy is what allows it to be tested before the mapper
exists, and what keeps "did we read the right column" separable from "did we format it right".

### 3.3 Collections are ordered in SQL

```csharp
Attachments = f.Attachments
    .OrderBy(a => a.SortOrder).ThenBy(a => a.Id)
    .Select(a => new AttachmentSnapshot(a.Id, a.Url, a.SortOrder, a.IsPrimary))
    .ToList(),
```

Order by a stable business key, not database order, and do it here rather than in memory. Stage 3
hashes the payload for reconciliation; a non-deterministic order makes every hash comparison a false
divergence.

---

## 4. The test pair, and the trap between its halves

Slice 1 asserts the projection **in memory**:

```csharp
private static readonly Func<Foo, FooSnapshot> Project =
    FooSnapshotProjection.ToSnapshot.Compile();
```

Fast, no Docker, and it catches the ordinary mistake — a member assigned from the wrong source.

**But `Compile()` runs the expression as LINQ-to-Objects, which executes the real property bodies.**
A computed collection (§1.3) keeps its filter here and **passes**, while EF ignores the body and
returns unfiltered rows in SQL.

The in-memory test is therefore **necessary and not sufficient**. Slice 2 re-runs the same assertions
through PostgreSQL, and the divergence between the two halves is what detects the trap. Write the
assertions once, in a shared helper called from both, so they cannot drift into asserting different
things. If the two disagree, **the in-memory result is the one that is wrong about production**.

### Test data: hand-written builders, not AutoFixture

A DDD aggregate has a private constructor and a factory that enforces invariants. `AutoFixture` will
either fail on it or bypass the factory by reflection — and a snapshot projected from an aggregate
that could never exist proves nothing. Write a builder that calls the real factory methods; use
`Bogus` for values inside it. The builder is reused by every later slice.

---

## 5. Dependencies

**No production dependency is added in this slice.** Four test packages, all permissive:

| Package | Licence | Why |
|---|---|---|
| `xunit` or `xunit.v3` | Apache-2.0 | Whatever the solution already uses. v3 changes the test project shape — check first |
| `Shouldly` | MIT | Assertions. **`FluentAssertions` v8+ is commercial**; use 7.x pinned only if the codebase already depends on it |
| `NetArchTest.Rules` | MIT | Dependency-direction assertions in three lines |
| `Bogus` | MIT | Values inside the hand-written builder |

Deliberately not added: `MediatR` (commercial from v13, and no dispatcher is needed here),
`AutoMapper` (commercial from v14), `Riok.Mapperly` (arrives with the first mapper in slice 4,
together with its diagnostics-as-errors setting), `Testcontainers` (slice 2).

Pin versions centrally.
