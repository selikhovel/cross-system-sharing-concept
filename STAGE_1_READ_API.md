# Stage 1 — Read API implementation guide

- **Stage:** 1 of 7 ([ADR-0007](adr/ADR-0007-staged-delivery-and-the-first-increment.md))
- **Goal:** a peer can `GET` our `Foo` catalogue in **their** contract shape, page by page,
  built on demand from current domain state.
- **Constraints:** no new table, no migration, no change to the write path, no worker, no
  `DbContext.SaveChanges` anywhere in this stage.
- **Related:** [ADR-0004](adr/ADR-0004-contract-mapping-payload-and-versioning.md) (the ACL and how
  losslessness is enforced) · [ADR-0005](adr/ADR-0005-authentication-and-transport-security.md)
  (the security floor) · [ADR-0006 §2](adr/ADR-0006-initial-backfill-and-reconciliation.md) (the
  page contract this reuses) · [MAPPING_MATRIX.md](MAPPING_MATRIX.md) (the field spec) ·
  [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) (the day checklist this document expands)
- **Spec:** `openspec/changes/add-foo-read-api/` — 11 requirements, 32 scenarios

This document is the *how*. It assumes an existing ASP.NET Core service on Clean Architecture +
CQRS + DDD with EF Core and PostgreSQL, and it does not assume anything about that service's
internals — §1 is the list of things to check, with the fallback for each answer.

**It also assumes nothing about your domain.** `Acme` is your service, `Foo` is the aggregate you
want to publish, `Bar` and `Baz` are aggregates it references, `FooAttachment` and `FooTag` are its
child collections. Substitute your own names and every section below applies as written — the code
shape is driven by EF Core, DDD encapsulation and the peer's contract, not by what `Foo` is. The
placeholder table is in [CLAUDE.md](CLAUDE.md#naming--all-names-are-placeholders).

Value-level examples stay concrete (money, a quantity with a unit, a date-only field, coordinates,
contact details) because that is where the traps live — §4.4 and §7 are worth nothing without them.

---

## 0. The principle — and what stage 1 is *not*

Stage 1 is **a read model plus a translator, served synchronously**. It is not a messaging
pipeline, and reading it as one is the most likely way to build the wrong thing.

```
HTTP GET  →  keyset SQL  →  FooSnapshot  →  mapper  →  redact PII  →  envelope  →  JSON
```

No write. No queue. No worker. No stored payload.

| Mechanism | In stage 1? | First appears | Why |
|---|---|---|---|
| **Domain events raised by aggregates** (`AggregateRoot.RaiseEvent`, an event collection on `Foo`, a marker interface) | **No** | **In no stage at all** | ADR-0002's constraint is that the domain model is never modified for integration. Change detection is an EF `SaveChangesInterceptor` reading the `ChangeTracker` from the infrastructure ring. The whole design is deliberately free of domain events |
| Application-level integration events, opt-in | No | Stage 3, opt-in | Only for the few cases where the *transition* is the payload (`PriceChanged`). Enqueued by the **Application** layer, never by the domain — ADR-0002 §3 |
| **Transactional outbox** | **No** | Stage 2 (`outbox_change_log`) → stage 3 (`outbox_message` + feed) | Stage 1 delivers nothing, so there is nothing to make durable |
| Change capture interceptor | No | Stage 2 | — |
| Background workers | No | Stage 3 | — |
| Push delivery, retry, dead letters, subscribers | No | Stage 4 | — |
| **Anti-corruption mapper** (domain → flat snapshot → the peer's contract) | **Yes** | **Stage 1 — and it is the entire substance of the stage** | ADR-0004. It is also the one artefact every later stage reuses unchanged |

**Why no capture mechanism is needed.** Capture and the outbox exist to answer one question:
*"what changed since X?"*. Stage 1 never asks it — it answers *"what is it now?"*. The direction of
pull is the consumer's, so nothing inside our system needs to know that a change happened.

**What that costs, stated plainly**, because it is the trade this stage makes: the consumer cannot
synchronise incrementally — it re-reads and replaces — and it receives **no signal that a record was
deleted**. Both belong in `CONSUMER_CONTRACT.md`, in those words. Stage 2 adds `aggregateVersion` so
the version rule becomes usable; stage 3 adds the incremental feed and tombstones, and only then does
"no lost changes" become true.

**Where the CQRS instinct leads you wrong.** The reflex in a CQRS codebase is a query handler in
`Application` returning the contract shape. §2.4 argues against it: the shape is dictated by a
foreign schema, so it does not belong in the layer that owns our use cases.

---

## 1. Discovery — what must be true in your system

Do this **before** writing code. Every row is a real fork in the implementation, and finding the
answer on day 4 costs a rewrite. Fill the "Answer" column in a scratch copy.

### 1.1 Solution shape

| # | Check | How | Why it matters |
|---|---|---|---|
| 1.1.1 | Project graph and which ring owns `DbContext` | `dotnet list <sln> reference`, or read the `.csproj` files | Decides where the projection lives (§2.2) |
| 1.1.2 | Does `Domain` reference anything outward? | The NetArchTest in §6.5 — write it first, it is trivially green or it finds a pre-existing violation | If it is already violated, stage 1's boundary claim is not new work but cleanup |
| 1.1.3 | Minimal APIs or controllers? Existing route grouping and versioning convention | Read `Program.cs` / `Startup` | §4.8 is written for `RouteGroupBuilder`; controllers need the same code in an `ApiController` |
| 1.1.4 | Existing CQRS dispatch abstraction and its licence | Look for `IQueryHandler<,>`, `ISender`, `IRequestHandler<,>` | **If `MediatR` ≥ v13, it is commercial.** Stage 1 needs no dispatcher at all (§2.4) — do not add one |
| 1.1.5 | Is there a read-side/query project already (`Application.Queries`, a read DbContext, Dapper)? | Grep for `AsNoTracking` | If a read-side convention exists, follow it instead of inventing one |

### 1.2 The `Foo` aggregate

This is the bulk of the work. Produce a written inventory — it becomes `MAPPING_MATRIX.md` §2.

| # | Check | How | Why it matters |
|---|---|---|---|
| 1.2.1 | Every public member of `Foo`, recursively through value objects and child entities | Read the type; the coverage test (§6.1) will enforce that you missed nothing | A field you do not know about is a field you cannot map or exclude |
| 1.2.2 | Is `Id` a strongly-typed id with a value converter? | `builder.Property(x => x.Id).HasConversion(...)` in the EF configuration | **Decides whether keyset pagination translates to SQL** (§1.4.1) |
| 1.2.3 | Are `Money`, `Address`, `Coordinates` mapped as **owned types** (`OwnsOne`) or via a **single-column converter** (`HasConversion`, JSON)? | The EF configuration | An owned type projects sub-properties in SQL. A single-column converter **cannot** — you must read the whole column and map in memory (§1.4.2) |
| 1.2.4 | Are collections (`Attachments`, `Tags`) exposed as **plain mapped navigations** or as **computed properties**? | Read the property body | See the trap in §7.1 — this one silently returns wrong data |
| 1.2.5 | Is there a soft-delete flag, and is there a **global query filter** for it? | `HasQueryFilter` in the configuration | If a filter exists, the projection already excludes deleted rows. If not, filter explicitly, or you publish deleted records |
| 1.2.6 | Enum members of every enum reachable from `Foo` | Read the enums | Each needs a row in `MAPPING_MATRIX.md` §4 and an arm in a fail-closed switch |
| 1.2.7 | Nullability truth versus declaration | `SELECT count(*) FROM foos WHERE <col> IS NULL` for anything the peer marks required | A field declared non-nullable in C# but null in 4 000 rows of production data fails at materialisation, not at compile time |
| 1.2.8 | Which fields are personal data, and what the recorded grant permits | The privacy/legal owner, not the code | Drives §4.6. Q6 is answered — the peer **is** entitled — but the legal basis and its retention are not, and §1.5.3 is why that matters |

### 1.3 The peer's contract

| # | Check | Why it matters |
|---|---|---|
| 1.3.1 | Is the OpenAPI/JSON-Schema document available? (Q3) | Stage 1 can be **built** against a stub but cannot **exit** without it |
| 1.3.2 | Which fields does the peer mark `required`, and with what `maxLength`, `format`, `enum` values? | Drives the length guards, the enum tables and §1.5.3 |
| 1.3.3 | Does the peer accept unknown fields? | Determines whether extra fields are a safe additive path later |
| 1.3.4 | Is a pilot consumer named? (Q17) | Without one, stage 1's value drops to mapper validation — re-read ADR-0007 alternative (A) |

### 1.4 PostgreSQL and EF specifics

| # | Check | How | Fallback if the answer is bad |
|---|---|---|---|
| 1.4.1 | Does a keyset comparison on the id translate to SQL? | Write the query from §4.3, enable `LogTo(Console.WriteLine)`, run it, **read the generated SQL**. You want `WHERE p.id > $1 ORDER BY p.id LIMIT $2` | If it throws or evaluates client-side: use the `FromSql` variant in §4.3.1 |
| 1.4.2 | Do value-object sub-properties appear as separate columns? | `\d+ foos` in `psql` | If `money` or `address` is one `jsonb`/`text` column, project the whole value object and convert in memory — correct, just a wider read |
| 1.4.3 | Is `created_at` (or your chosen order key) indexed? | `\di foos*` | **Page by the primary key instead** (§4.3). The PK index always exists, so this keeps "no migration" true |
| 1.4.4 | `decimal` precision on money and area columns | `\d+ foos` | `numeric` without precision is fine; `double precision` is a data-quality finding to raise, not to paper over |
| 1.4.5 | `timestamptz` or `timestamp`? | `\d+ foos` | Plain `timestamp` means the offset is a convention, not data. Document the assumed zone and state it in the mapping matrix |
| 1.4.6 | Npgsql version and whether `DateOnly` is mapped to `date` | `dotnet list package` | Npgsql 6+ maps `DateOnly` ↔ `date`. Older: the field is a `DateTime` and §1.5.2 applies |
| 1.4.7 | Row count of `foos` | `SELECT count(*) FROM foos` | Sizes the page cap and the load test |
| 1.4.8 | Is a read replica available? | Ask the platform team | Not required in stage 1, but it is the cheapest scaling lever and it is a connection string, not code |

### 1.5 Three checks that are easy to skip and expensive to skip

**1.5.1 — Does anything already write to these tables outside EF?** `psql` scripts, a reporting job,
an ETL. Stage 1 only reads, so this is not yet a correctness issue — but it is the input to stage 2's
capture gap (ADR-0002's sharpest edge), and asking now is free.

**1.5.2 — Are date-only fields actually date-only in the database?** If `effective_on` is
`timestamp` holding midnight local time, converting it to UTC on the wire shifts the date backwards
for half the world. The matrix already forbids this; verify which one you have before writing the
converter.

**1.5.3 — Does the peer mark any personal-data field `required`?** Stage 1 redacts personal data
unconditionally (§2.3 Q6 default: "redact until confirmed"). If the peer's schema requires
`/contact/name`, a redacted payload **fails their validation** — the same shape of defect as
[D11](openspec/changes/add-cross-system-integration-layer/design.md). In that case Q6 stops being a
default and becomes a stage 1 blocker: either the peer makes the field optional, or the PII grant is
confirmed before stage 1 ships. In `MAPPING_MATRIX.md`'s worked example all three contact fields are
optional. **In this integration they are not.** Q6 is answered in two parts: the peer *is* entitled
to the data, **and** it marks some personal fields `required`. Blanket redaction is therefore off the
table — it would produce payloads that fail the peer's own validation — and what governs instead is
the recorded grant. §4.6 is written accordingly: the redactor is a no-op in stage 1 and exists so
that stage 4's per-subscriber variant has something to generalise. Still missing, and needed before
stage 1 ships: the legal basis and the peer's retention period.

---

## 2. Where the code goes

### 2.1 Reference graph

```
        ┌──────────────────────────── unchanged ────────────────────────────┐
        │                                                                   │
        │   Acme.Domain          Acme.Application          │
        │        (aggregates)                (use cases, CQRS)               │
        │             ▲                            ▲                        │
        └─────────────┼────────────────────────────┼────────────────────────┘
                      │                            │
        ┌─────────────┴────────────────────────────┴────────────────────────┐
        │  Acme.Infrastructure        (AppDbContext, EF configs)   │
        └─────────────▲─────────────────────────────────────────────────────┘
                      │  reads only: AsNoTracking projections
        ┌─────────────┴─────────────────────────────────────────────────────┐
        │  Acme.Integration                              ← NEW     │
        │    Snapshots/   flat read models + the projection expression      │
        │    Mapping/     Mapperly mappers, enum tables, converters   = ACL │
        │    Contracts/   envelope + page response                          │
        │    Paging/      opaque cursor codec                               │
        │    Endpoints/   the two routes                                    │
        │    Security/    IIntegrationAuthenticator seam, redaction         │
        └─────────────▲─────────────────────────────────────────────────────┘
                      │
        ┌─────────────┴─────────────────────────────────────────────────────┐
        │  Acme.Api          + one line: app.MapIntegrationV1()    │
        └───────────────────────────────────────────────────────────────────┘

        Acme.Integration.Contracts   ← NEW, generated from the peer's OpenAPI
```

Two rules, both architecture-tested (§6.5):

- **`Domain` and `Application` reference nothing new.** Not `Integration`, not
  `Integration.Contracts`, not `System.Net.Http`.
- **External contract types never leave `Integration`.** They do not appear in an application
  command, an EF entity, or any API response outside the integration routes.

### 2.2 Why `Integration` reads `Infrastructure` directly

`Integration` lives in the **infrastructure ring** (TECHNICAL_PROPOSAL §1), so referencing
`Infrastructure` for the `DbContext` is a peer-to-peer reference inside one ring, not an inversion.
The alternative — a port in `Integration` implemented in `Infrastructure` — adds an interface and a
registration to buy nothing here, because both assemblies are replaced together anyway.

**Use the port-and-adapter form only if** your `AppDbContext` is `internal` to `Infrastructure`, or
your build enforces that nothing but `Infrastructure` may reference EF Core. Then:
`IFooSnapshotStore` is declared in `Integration`, implemented in
`Infrastructure/Integration/FooSnapshotStore.cs`, and `Infrastructure` gains a reference to
`Integration`. The projection expression (§4.2) stays in `Integration` either way — it is the shape
of the contract, not of the database.

### 2.3 If the service is a modular monolith

The reference graph above assumes one Clean Architecture ring set. In a modular monolith the design
splits along a seam it already has:

| Part | Knows about `Foo`? | Belongs to |
|---|---|---|
| **Mechanism** — outbox tables, `SKIP LOCKED` claiming, dispatcher, retries, the cursor feed and its watermark, inbox, backfill | No. Entirely domain-agnostic | A shared building block, one per monolith |
| **Policy** — the snapshot, the projection, the ACL mapper, enum tables, filter rules, which fields are personal data | Yes, completely | **Inside the module that owns `Foo`** |

Stage 1 is **all policy and no mechanism**, so it needs no building block at all. Add
`<Module>.Integration` alongside the module's existing projects, plus `<Module>.Integration.Contracts`
for the generated peer types.

**The contracts project is the one boundary worth a project reference even if everything else is
folders.** ADR-0004 §1's rule — an external contract type never appears in a domain signature, an
application command, an EF entity or a public API response — is the one that gets broken by
inattention, and a project reference catches it at compile time where a namespace convention does
not.

**Do not extract the building block before a second module needs it.** With one module it is an
abstraction with no second consumer, and it will be redesigned the moment a real second module
arrives with its own requirements. Instead keep the seam visible as folders, so that extraction is a
move rather than a rewrite:

```
<Module>.Integration/
├── Snapshots/     policy      snapshot, projection, store
├── Mapping/       policy      mapper, enum tables, converters, guards, exclusions
├── Security/      mixed       authenticator seam (mechanism) · PII registry, redactor (policy)
├── Paging/        mechanism   cursor
├── Contracts/     mechanism   envelope, page response, JSON context
├── Endpoints/     mixed       routes (policy) · feature-flag filter (mechanism)
└── Errors/        mechanism
```

Later stages add `Outbox/`, `Delivery/`, `Inbox/` and `Backfill/`, all mechanism. By stage 4 the
mechanism is the bulk of the project and none of it names `Foo` — which is when extraction pays.
What keeps it that way is ADR-0002 §1's **aggregate registration map**: the mechanism receives its
aggregates from a map rather than referring to them by type.

Two consequences for discovery (§1) when modules are in play:

- **Where do `Bar` and `Baz` live?** If they belong to another module, the projection may not read
  them directly — that is a module-boundary breach — and must go through that module's published
  contract instead. This changes the projection and the stage 3 fan-out story, so answer it before
  writing the projection.
- **One `DbContext` per module?** Then the shared-outbox decision in ADR-0003 §1 applies, and the
  `integration` schema wants its own migrations history table from the first migration in stage 2.

### 2.4 Why this does *not* go through a CQRS query handler

Your service has CQRS, so the reflex is `GetFooSnapshotQuery : IQuery<FooSnapshot>` in
`Application`. Do not do that in stage 1:

- `FooSnapshot`'s shape is dictated by **the peer's contract**, not by a use case. Putting it in
  `Application` puts a foreign schema's shadow into the layer that owns our use cases — which is
  precisely what ADR-0004 §1 builds three model layers to prevent.
- The read has no business logic, no authorisation decision beyond the endpoint scope, and no
  invariant to uphold. A handler would be a pass-through with a licence cost if your mediator is
  `MediatR` ≥ v13.
- Stage 3's materialiser calls the same projection from a `BackgroundService`, where there is no
  request, no ambient context and no reason to route through the application pipeline.

**Do** route through `Application` when the read needs something `Application` owns — a permission
check, a tenant filter, a computed value produced by a domain service. Then the query returns a
domain-shaped result and `Integration` maps it, keeping the contract type out of `Application`'s
signature.

The inbound direction of stage 5 is the mirror image and goes **through** the application command
handlers, always — see ADR-0001 §2. Stage 1 is read-only, which is the whole reason it is allowed a
shortcut.

---

## 3. The data path

```
  GET /integration/v1/foos?cursor=eyJ2IjoxLC…&limit=100
        │
        │  1. authenticate → scope check → rate limit          Security/  (§4.7)
        ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ 2. decode the opaque cursor  →  (lastId)          Paging/CursorCodec  │  §4.5
  └───────────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ 3. keyset query, AsNoTracking, AsSplitQuery, Take(limit + 1)          │  §4.3
  │    projected straight into FooSnapshot — no aggregate loaded     │
  │      SELECT … FROM foos p WHERE p.id > $1 ORDER BY p.id LIMIT $2│
  └───────────────────────────────────────────────────────────────────────┘
        │  IReadOnlyList<FooSnapshot>
        ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ 4. Snapshot → ExternalFoo        Mapping/FooContractMapper  │  §4.4
  │      fail-closed enums · length guards · unit + money converters      │  = the ACL
  └───────────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ 5. redact personal data (unconditional in stage 1)   Security/Redactor│  §4.6
  └───────────────────────────────────────────────────────────────────────┘
        │
        ▼
  ┌───────────────────────────────────────────────────────────────────────┐
  │ 6. wrap: SnapshotEnvelope per item, PageResponse around the page      │  §4.1
  │ 7. serialise with the integration-only JsonSerializerContext          │  §4.9
  └───────────────────────────────────────────────────────────────────────┘
        │
        ▼
  200 OK { "items": [ { …envelope…, "data": { …peer's schema… } } ],
           "nextCursor": "eyJ2IjoxLC…", "hasMore": true }
```

Nothing in this path writes. Nothing in this path loads an aggregate. Both properties are asserted by
tests, because both stop being true the first time someone "just adds an `Include`".

---

## 4. The code

Namespaces are illustrative. Every file below lands in `Acme.Integration` unless stated.

### 4.1 Contracts — the envelope and the page

```csharp
// Contracts/SnapshotEnvelope.cs
namespace Acme.Integration.Contracts;

/// <summary>
/// Stage 1 subset of the message envelope (ADR-0004 §3). Absent by design:
/// messageId / changeKind (nothing happened — this is a read), correlationId / traceParent
/// (no originating business transaction), and aggregateVersion, which arrives in stage 2
/// as an additive change (ADR-0007 §Stage 1).
/// </summary>
public sealed record SnapshotEnvelope<TData>
{
    public required string          ContractVersion { get; init; }   // "1"
    public required string          Source          { get; init; }   // "acme"
    public required string          AggregateType   { get; init; }   // "Foo"
    public required string          AggregateId     { get; init; }
    public required DateTimeOffset  ProducedAt      { get; init; }
    public bool                     Partial         => false;
    public required TData           Data            { get; init; }
}

// Contracts/PageResponse.cs
/// <summary>
/// The page shape of ADR-0006 §2. Stage 3 adds a pinned <c>watermark</c> — additive,
/// so a consumer's parser does not change when this becomes the snapshot endpoint.
/// </summary>
public sealed record PageResponse<TItem>
{
    public required IReadOnlyList<TItem> Items      { get; init; }
    public          string?              NextCursor { get; init; }
    public required bool                 HasMore    { get; init; }
}
```

### 4.2 Snapshots — the flat read model and the projection

The projection is a **static expression**, not a method body, so the same definition serves the
single read, the page read, and stage 3's materialiser. That reuse is the point: if the projection
lives inside a method, stage 3 copies it and the two drift.

```csharp
// Snapshots/FooSnapshot.cs
namespace Acme.Integration.Snapshots;

public sealed record FooSnapshot
{
    public required Guid                             Id            { get; init; }
    public required string                           Title         { get; init; }
    public          string?                          Description   { get; init; }
    public required FooKind                     Kind          { get; init; }
    public required FooStatus                    Status        { get; init; }
    public          decimal?                         PriceAmount   { get; init; }
    public          string?                          PriceCurrency { get; init; }
    public          decimal?                         Quantity       { get; init; }
    public          int?                             Count         { get; init; }
    public          int?                             Rank         { get; init; }
    public          Guid?                            BarId    { get; init; }
    public required Guid                             BazId    { get; init; }
    public required AddressSnapshot                  Address       { get; init; }
    public          double?                          Latitude      { get; init; }
    public          double?                          Longitude     { get; init; }
    public required DateOnly                         EffectiveOn      { get; init; }
    public required DateTimeOffset                   CreatedAtUtc  { get; init; }
    public required DateTimeOffset                   UpdatedAtUtc  { get; init; }
    public required IReadOnlyList<FooTagSnapshot>   Tags      { get; init; }
    public required IReadOnlyList<AttachmentSnapshot>     Attachments        { get; init; }
    public          ContactSnapshot?                   Contact         { get; init; }
    public required bool                             IsDeleted     { get; init; }
}

public sealed record AddressSnapshot(string Line1, string? PostalCode, string CountryCode);
public sealed record FooTagSnapshot(Guid Id, FooTagKind Kind);
public sealed record AttachmentSnapshot(Guid Id, string Url, int SortOrder, bool IsPrimary);
public sealed record ContactSnapshot(string Name, string? Phone, string? Email);
```

```csharp
// Snapshots/FooSnapshotProjection.cs
using System.Linq.Expressions;

namespace Acme.Integration.Snapshots;

public static class FooSnapshotProjection
{
    /// <summary>
    /// The ONLY place domain shape is read. Reused by the single read, the page read and
    /// (stage 3) the materialiser. Every member of FooSnapshot must be assigned here —
    /// a `required` member left out is a compile error, which is the cheapest guard available.
    /// </summary>
    public static readonly Expression<Func<Foo, FooSnapshot>> ToSnapshot =
        p => new FooSnapshot
        {
            Id            = p.Id.Value,                 // see §1.4.1 if this does not translate
            Title         = p.Title,
            Description   = p.Description,
            Kind          = p.Kind,
            Status        = p.Status,
            PriceAmount   = p.Price == null ? null : p.Price.Amount,
            PriceCurrency = p.Price == null ? null : p.Price.Currency,
            Quantity       = p.Quantity,
            Count         = p.Count,
            Rank         = p.Rank,
            BarId    = p.BarId == null ? null : p.BarId.Value.Value,
            BazId    = p.BazId.Value,
            Address       = new AddressSnapshot(
                                p.Address.Line1, p.Address.PostalCode, p.Address.CountryCode),
            Latitude      = p.Coordinates == null ? null : p.Coordinates.Lat,
            Longitude     = p.Coordinates == null ? null : p.Coordinates.Lng,
            EffectiveOn      = p.EffectiveOn,
            CreatedAtUtc  = p.CreatedAtUtc,
            UpdatedAtUtc  = p.UpdatedAtUtc,

            // Ordered in SQL, not in memory — the payload hash in stage 3 depends on
            // deterministic order (MAPPING_MATRIX §6.7), so establish it here.
            Tags          = p.Tags
                             .OrderBy(f => f.Id)
                             .Select(f => new FooTagSnapshot(f.Id, f.Kind))
                             .ToList(),
            Attachments        = p.Attachments
                             .OrderBy(ph => ph.SortOrder).ThenBy(ph => ph.Id)
                             .Select(ph => new AttachmentSnapshot(
                                 ph.Id, ph.Url, ph.SortOrder, ph.IsPrimary))
                             .ToList(),

            Contact         = p.Contact == null
                                ? null
                                : new ContactSnapshot(p.Contact.Name, p.Contact.Phone, p.Contact.Email),
            IsDeleted     = p.IsDeleted
        };
}
```

> If §1.2.3 found `Money` or `Address` behind a single-column converter, drop the sub-property
> access and project the value object whole (`Price = p.Price`), then flatten in the mapper. The
> read is wider; correctness is unchanged.

### 4.3 The store — keyset paging

```csharp
// Snapshots/IFooSnapshotStore.cs
public interface IFooSnapshotStore
{
    Task<FooSnapshot?> GetAsync(Guid id, CancellationToken ct);

    /// <param name="afterId">Exclusive keyset cursor; null starts at the beginning.</param>
    /// <returns>Up to <paramref name="limit"/> + 1 rows; the caller uses the extra row as
    /// the has-more signal, so no COUNT query is ever issued.</returns>
    Task<IReadOnlyList<FooSnapshot>> GetPageAsync(
        Guid? afterId, int limit, CancellationToken ct);
}
```

```csharp
// Snapshots/FooSnapshotStore.cs
internal sealed class FooSnapshotStore(AppDbContext db) : IFooSnapshotStore
{
    public Task<FooSnapshot?> GetAsync(Guid id, CancellationToken ct) =>
        Query().Where(s => s.Id == id).FirstOrDefaultAsync(ct);

    public async Task<IReadOnlyList<FooSnapshot>> GetPageAsync(
        Guid? afterId, int limit, CancellationToken ct)
    {
        var q = Query();

        if (afterId is { } after)
            q = q.Where(s => s.Id > after);

        return await q.OrderBy(s => s.Id)
                      .Take(limit + 1)      // +1 = the has-more probe
                      .ToListAsync(ct);
    }

    private IQueryable<FooSnapshot> Query() =>
        db.Set<Foo>()
          .AsNoTracking()
          .AsSplitQuery()                   // two collections in one projection — see §7.2
          .Select(FooSnapshotProjection.ToSnapshot);
}
```

Three deliberate choices:

- **Keyset over the primary key, not `(created_at, id)`.** ADR-0006 §2 specifies
  `(created_at, id)`; stage 1 narrows it to `id` so that the query rides the primary-key index and
  **no migration is needed** (§1.4.3). Cost: pages are not in chronological order. If ids are
  UUIDv7 the order is chronological anyway; if a consumer needs chronological order, adding the
  composite index is stage 3's business.
- **Filtering the projected snapshot, not the entity** (`s.Id > after`). Ordering and filtering after
  a `Select` still translate to SQL, and this keeps the strongly-typed-id conversion inside the
  projection where §1.4.1 already verified it. If your ordering must be on the entity, do
  `.OrderBy(p => p.Id)` before `.Select(...)`.
- **`Take(limit + 1)`.** A `COUNT(*)` on every page doubles the query count and is wrong under
  concurrency anyway.

#### 4.3.1 Fallback if the keyset comparison does not translate

If §1.4.1 showed client-side evaluation or an exception, keep the projection and replace the source:

```csharp
private IQueryable<FooSnapshot> PageSource(Guid? afterId, int limit) =>
    db.Set<Foo>()
      .FromSql($"""
          SELECT * FROM core.foos
           WHERE ({afterId}::uuid IS NULL OR id > {afterId}::uuid)
           ORDER BY id
           LIMIT {limit + 1}
          """)
      .AsNoTracking()
      .AsSplitQuery()
      .Select(FooSnapshotProjection.ToSnapshot);
```

`FromSql` (not `FromSqlRaw`) interpolates as parameters, so this is not a concatenation hazard. The
ordering must be repeated in LINQ if you compose further — `FromSql` ordering is not preserved
through `Select` in every provider version, so verify the SQL again.

### 4.4 The mapper — the anti-corruption layer

```xml
<!-- Acme.Integration.csproj -->
<PropertyGroup>
  <!-- RMG012 unmapped target member · RMG020 unmapped source member -->
  <WarningsAsErrors>$(WarningsAsErrors);RMG012;RMG020</WarningsAsErrors>
</PropertyGroup>
```

```csharp
// Mapping/FooContractMapper.cs
using Riok.Mapperly.Abstractions;

namespace Acme.Integration.Mapping;

[Mapper]
public partial class FooContractMapper
{
    // Every ignore carries a reason and a MAPPING_MATRIX.md row. No exceptions.
    [MapperIgnoreSource(nameof(FooSnapshot.IsDeleted))]   // filter input, not payload data
    [MapperIgnoreSource(nameof(FooSnapshot.BazId))]  // mapped explicitly below
    [MapProperty(nameof(FooSnapshot.Title),      nameof(ExternalFoo.Title))]
    [MapProperty(nameof(FooSnapshot.Count),      nameof(ExternalFoo.Count))]
    [MapProperty(nameof(FooSnapshot.Rank),      nameof(ExternalFoo.Rank))]
    public partial ExternalFoo ToContract(FooSnapshot source);

    // ---- explicit conversions: each one is a documented rule in MAPPING_MATRIX §6 ----

    private static string MapId(Guid id) => id.ToString("D");

    private static string MapTitle(string title) =>
        ContractString.Required(title, max: 200, field: "/title");

    private static string? MapDescription(string? html) =>
        html is null ? null
                     : ContractString.Optional(HtmlText.SanitiseAndNormalise(html),
                                               max: 4000, field: "/description");

    private static ExternalMoney? MapPrice(decimal? amount, string? currency) =>
        amount is null || currency is null
            ? null                                   // null ≠ 0: "unpriced", not "free"
            : new ExternalMoney
              {
                  Amount   = Decimals.ToContractString(amount.Value, decimals: 2),
                  Currency = currency.ToUpperInvariant()
              };

    private static ExternalQuantity? MapQuantity(decimal? sqm) =>
        sqm is null ? null : Units.ToContractQuantity(sqm.Value);

    private static ExternalGeo? MapGeo(double? lat, double? lon) =>
        lat is null || lon is null
            ? null
            : new ExternalGeo { Lat = Coordinates.Round(lat.Value),
                                Lon = Coordinates.Round(lon.Value) };

    private static string MapEffectiveOn(DateOnly d) => d.ToString("yyyy-MM-dd", Invariant);
    private static string MapInstant(DateTimeOffset t) => t.ToUniversalTime().ToString("O", Invariant);

    private static CultureInfo Invariant => CultureInfo.InvariantCulture;
}
```

**Enums are hand-written and fail closed** — a Mapperly by-name mapping would be one refactor away
from silently matching a new value:

```csharp
// Mapping/FooKindMap.cs
internal static class FooKindMap
{
    public static string ToContract(FooKind kind) => kind switch
    {
        FooKind.Alpha  => "ALPHA",
        FooKind.Beta      => "BETA",
        FooKind.Gamma  => "GAMMA",
        FooKind.Delta       => "DELTA",
        FooKind.Epsilon => "EPSILON",
        // MAPPING_MATRIX §7.3 — unresolved with the peer. Fail, never guess.
        _ => throw new UnmappedEnumValueException(nameof(FooKind), kind.ToString())
    };
}
```

The missing `default` arm is the mechanism: C# warns on a non-exhaustive switch over an enum, and
the throw arm turns "we added `Zeta` and forgot" into a loud failure on the first affected record
instead of `"OTHER"` at the peer forever.

**Length guards raise, never truncate:**

```csharp
// Mapping/ContractString.cs
internal static class ContractString
{
    public static string Required(string value, int max, string field)
    {
        var trimmed = value.Trim();
        if (trimmed.Length == 0)
            throw new ContractViolationException(field, "required value is empty");
        if (trimmed.Length > max)
            throw new ContractViolationException(
                field, $"length {trimmed.Length} exceeds the peer's maxLength {max}");
        return trimmed;
    }

    public static string? Optional(string? value, int max, string field) =>
        value is null ? null : Required(value, max, field);
}
```

```csharp
// Mapping/Decimals.cs  ·  Units.cs
internal static class Decimals
{
    // MidpointRounding.AwayFromZero is the documented rule (MAPPING_MATRIX §6.2).
    // Never rely on ToString("F2") for this — its midpoint behaviour is not the rule we published.
    public static string ToContractString(decimal value, int decimals) =>
        Math.Round(value, decimals, MidpointRounding.AwayFromZero)
            .ToString("0." + new string('0', decimals), CultureInfo.InvariantCulture);
}

internal static class Units
{
    private const decimal SqmToSqft = 10.7639104167m;

    public static ExternalQuantity ToContractQuantity(decimal sqm) =>
        ContractUnits.PeerWantsSquareFeet
            ? new ExternalQuantity { Value = Math.Round(sqm * SqmToSqft, 2, MidpointRounding.AwayFromZero),
                                 Unit  = "SQFT" }
            : new ExternalQuantity { Value = Math.Round(sqm, 2, MidpointRounding.AwayFromZero),
                                 Unit  = "SQM" };
}
```

The `Unit` field is never omitted. A bare number is how a 10× area error ships.

### 4.5 The cursor

```csharp
// Paging/Cursor.cs
namespace Acme.Integration.Paging;

/// Opaque to the consumer, versioned so the encoding can change without breaking one.
public static class Cursor
{
    private const byte Version = 1;

    public static string Encode(Guid lastId)
    {
        Span<byte> buf = stackalloc byte[17];
        buf[0] = Version;
        lastId.TryWriteBytes(buf[1..]);
        return Base64Url.EncodeToString(buf);
    }

    public static bool TryDecode(string? cursor, out Guid lastId)
    {
        lastId = default;
        if (string.IsNullOrEmpty(cursor)) return false;

        Span<byte> buf = stackalloc byte[17];
        if (!Base64Url.TryDecodeFromChars(cursor, buf, out var written)
            || written != 17 || buf[0] != Version)
            return false;

        lastId = new Guid(buf[1..]);
        return true;
    }
}
```

A malformed cursor is a `400`, not an empty first page — silently restarting a walk is how a
consumer concludes it has synchronised when it has not.

### 4.6 Redaction

Personal-data fields live in **one** registry, so stage 4's per-subscriber variant reuses it instead
of re-deriving the list:

```csharp
// Security/PiiFields.cs
public static class PiiFields
{
    /// JSON pointers, matching MAPPING_MATRIX.md's PII column exactly.
    /// The coverage test (§6.1) asserts this list and the matrix agree.
    public static readonly ImmutableArray<string> Pointers =
    [
        "/address/street", "/address/postalCode",
        "/contact/name", "/contact/phone", "/contact/email"
    ];
}

// Security/ContractRedactor.cs
/// Q6 is answered: the peer IS entitled to the personal data, and marks some of it `required`.
/// Stage 1 therefore emits it under a recorded grant and this redactor is a deliberate no-op —
/// it exists so the seam is in place, and so stage 4 has something to generalise into the
/// per-subscriber payload variant (defect D7, deferred at one consumer).
internal sealed class ContractRedactor
{
    // One grant, recorded in configuration alongside its legal basis and the peer's retention.
    public ExternalFoo Apply(ExternalFoo p) => _grant.CoversPersonalData ? p : Strip(p);

    private static ExternalFoo Strip(ExternalFoo p) => p with
    {
        Address = p.Address is null ? null : p.Address with { Street = null, PostalCode = null },
        Contact = null
    };
}
```

> **Do not reach for blanket redaction here.** Because the peer marks some personal fields
> `required`, stripping them produces payloads that fail its own validation — the same shape of
> defect as D11. What makes this safe is the *grant*, not the *stripping*, and the grant is not
> complete until the legal basis and the peer's retention period are written down. `PiiFields` stays
> regardless: it drives the coverage assertion now, and stage 4's variant and stage 7's purge routine
> later.

### 4.7 Security floor

```csharp
// Security/IIntegrationAuthenticator.cs
/// The seam of ADR-0005 §2. Stage 1 needs identity only for the rate limit and the audit line;
/// stage 5 needs it to resolve `source_system`, which is why it must NOT be an OAuth-specific
/// claim lookup (defect S1).
public interface IIntegrationAuthenticator
{
    bool TryResolveCaller(HttpContext ctx, out CallerIdentity caller);
}

public sealed record CallerIdentity(string Id, string Mechanism);
```

```csharp
// Program.cs (Api)
builder.Services.AddRateLimiter(o =>
{
    o.AddPolicy("integration", ctx =>
        RateLimitPartition.GetFixedWindowLimiter(
            ctx.RequestServices.GetRequiredService<IIntegrationAuthenticator>()
               .TryResolveCaller(ctx, out var caller) ? caller.Id : "anonymous",
            _ => new FixedWindowRateLimiterOptions
                 { PermitLimit = 60, Window = TimeSpan.FromMinutes(1), QueueLimit = 0 }));
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("integration.feed.read", p => p
        .RequireAuthenticatedUser()
        .RequireClaim("scope", "acme.feed.read"));
```

Non-negotiable regardless of mechanism (ADR-0005 §3): TLS only, no
`ServerCertificateCustomValidationCallback`, `MaxRequestBodySize` set, and a Serilog destructuring
policy redacting `Authorization`, `X-Signature`, `X-Api-Key`, `*token*`, `*secret*`.

### 4.8 The endpoints

```csharp
// Endpoints/IntegrationV1Endpoints.cs
public static class IntegrationV1Endpoints
{
    private const int DefaultLimit = 100;
    private const int MaxLimit     = 500;

    public static IEndpointRouteBuilder MapIntegrationV1(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/integration/v1/foos")
                       .RequireAuthorization("integration.feed.read")
                       .RequireRateLimiting("integration")
                       .AddEndpointFilter<IntegrationFeatureFlagFilter>()   // §4.10
                       .WithTags("integration-v1");

        group.MapGet("/{id:guid}", GetOne).WithName("GetFooSnapshot");
        group.MapGet("/",          GetPage).WithName("GetFooSnapshotPage");
        return app;
    }

    private static async Task<Results<Ok<SnapshotEnvelope<ExternalFoo>>, NotFound>> GetOne(
        Guid id,
        IFooSnapshotStore store,
        FooContractMapper mapper,
        ContractRedactor redactor,
        TimeProvider clock,
        CancellationToken ct)
    {
        var snapshot = await store.GetAsync(id, ct);
        if (snapshot is null || snapshot.IsDeleted)
            return TypedResults.NotFound();

        return TypedResults.Ok(Envelope(snapshot, mapper, redactor, clock));
    }

    private static async Task<Results<Ok<PageResponse<SnapshotEnvelope<ExternalFoo>>>,
                                     BadRequest<ProblemDetails>>> GetPage(
        string? cursor,
        int? limit,
        IFooSnapshotStore store,
        FooContractMapper mapper,
        ContractRedactor redactor,
        TimeProvider clock,
        CancellationToken ct)
    {
        Guid? afterId = null;
        if (cursor is not null)
        {
            if (!Cursor.TryDecode(cursor, out var decoded))
                return TypedResults.BadRequest(Problems.InvalidCursor());
            afterId = decoded;
        }

        var take = Math.Clamp(limit ?? DefaultLimit, 1, MaxLimit);   // capped, not rejected
        var rows = await store.GetPageAsync(afterId, take, ct);

        var hasMore = rows.Count > take;
        var page    = hasMore ? rows.Take(take).ToList() : rows;

        return TypedResults.Ok(new PageResponse<SnapshotEnvelope<ExternalFoo>>
        {
            Items      = page.Select(s => Envelope(s, mapper, redactor, clock)).ToList(),
            NextCursor = hasMore ? Cursor.Encode(page[^1].Id) : null,
            HasMore    = hasMore
        });
    }

    private static SnapshotEnvelope<ExternalFoo> Envelope(
        FooSnapshot s, FooContractMapper mapper,
        ContractRedactor redactor, TimeProvider clock) =>
        new()
        {
            ContractVersion = "1",
            Source          = "acme",
            AggregateType   = "Foo",
            AggregateId     = s.Id.ToString("D"),
            ProducedAt      = clock.GetUtcNow(),
            Data            = redactor.Redact(mapper.ToContract(s))
        };
}
```

Note what is **not** here: no soft-deleted row is served (`IsDeleted` check plus the global query
filter if §1.2.5 found one), no `Include`, no `SaveChanges`, no per-request `JsonSerializerOptions`
construction.

A `ContractViolationException` from the mapper — an over-long title, an unmapped enum — should
surface as a `500` with the field name logged, not as a partial payload. It is a data-quality
incident on our side, and stage 3 converts the same exception into a dead letter.

### 4.9 Serialisation

```csharp
// Contracts/IntegrationJson.cs
[JsonSourceGenerationOptions(
    PropertyNamingPolicy   = JsonKnownNamingPolicy.CamelCase,   // match the PEER's schema exactly
    DefaultIgnoreCondition = JsonIgnoreCondition.Never,         // snapshots emit explicit nulls
    NumberHandling         = JsonNumberHandling.Strict)]
[JsonSerializable(typeof(SnapshotEnvelope<ExternalFoo>))]
[JsonSerializable(typeof(PageResponse<SnapshotEnvelope<ExternalFoo>>))]
internal partial class IntegrationJsonContext : JsonSerializerContext;
```

Register it **only** on the integration group, never on the whole app — the peer's naming policy is
theirs, and adopting it globally leaks a foreign schema into every existing endpoint:

```csharp
group.AddEndpointFilter(new JsonOptionsFilter(IntegrationJsonContext.Default));
```

### 4.10 Wiring and the flag

```csharp
// DependencyInjection.cs (Integration)
public static IServiceCollection AddIntegrationReadApi(
    this IServiceCollection services, IConfiguration config)
{
    services.Configure<IntegrationOptions>(config.GetSection("Integration"));
    services.AddScoped<IFooSnapshotStore, FooSnapshotStore>();
    services.AddSingleton<FooContractMapper>();     // Mapperly mappers are stateless
    services.AddSingleton<ContractRedactor>();
    services.AddSingleton<IIntegrationAuthenticator, ScopeClaimAuthenticator>();
    services.TryAddSingleton(TimeProvider.System);
    return services;
}
```

```jsonc
// appsettings.json
"Integration": {
  "ReadApiEnabled": false,      // default off in production — ADR-0007 reversibility
  "MaxPageSize": 500,
  "PeerAreaUnit": "SQM"
}
```

`IntegrationFeatureFlagFilter` returns `404` (not `403`) when the flag is off: an endpoint that does
not exist yet should be indistinguishable from an endpoint that does not exist.

---

## 5. Files and dependencies

### 5.1 Files to create

```
src/Acme.Integration/
├── Contracts/            SnapshotEnvelope.cs · PageResponse.cs · IntegrationJson.cs
├── Snapshots/            FooSnapshot.cs · FooSnapshotProjection.cs
│                         IFooSnapshotStore.cs · FooSnapshotStore.cs
├── Mapping/              FooContractMapper.cs · FooKindMap.cs · FooStatusMap.cs
│                         FooTagKindMap.cs · ContractString.cs · Decimals.cs · Units.cs
│                         Coordinates.cs · Phone.cs · HtmlText.cs
│                         MappingRegistry.cs · IntegrationFieldExclusions.cs
├── Paging/               Cursor.cs
├── Security/             IIntegrationAuthenticator.cs · ScopeClaimAuthenticator.cs
│                         PiiFields.cs · ContractRedactor.cs
├── Endpoints/            IntegrationV1Endpoints.cs · IntegrationFeatureFlagFilter.cs
│                         Problems.cs
├── Errors/               ContractViolationException.cs · UnmappedEnumValueException.cs
├── IntegrationOptions.cs
└── DependencyInjection.cs

src/Acme.Integration.Contracts/     generated from contracts/external/{system}/openapi.v1.yaml
contracts/external/{system}/openapi.v1.yaml  vendored, diffable in a PR

tests/Acme.Integration.Tests/
├── Mapping/              FieldCoverageTests.cs · RoundTripTests.cs · EnumTests.cs
│                         LengthGuardTests.cs · ValueRuleTests.cs
├── Paging/               CursorTests.cs · KeysetWalkTests.cs        (Testcontainers)
├── Security/             RedactionTests.cs · LogRedactionTests.cs · AuthorizationTests.cs
├── Architecture/         BoundaryTests.cs
└── Schema/               PayloadValidationTests.cs

src/Acme.Api/Program.cs             + AddIntegrationReadApi() + MapIntegrationV1()
```

One line changes in an existing project. Everything else is new and deletable.

### 5.2 Dependencies

Pin exact versions centrally (`Directory.Packages.props`) — a floating range on a source generator
turns a mapping diagnostic into a build that behaves differently on two machines.

**Slice 1 adds no production dependency at all** — four test packages and nothing else. Every
package below lands in the commit that first needs it, which keeps "what is this for?" answerable at
review time.

| Package | Licence | Slice | Why this one | Instead of |
|---|---|---|---|---|
| `Riok.Mapperly` | Apache-2.0 ✅ | 4 | Compile-time source generator. Its unmapped-member diagnostics are **load-bearing**, not a convenience — ADR-0004 §5(a). Reference it in the same commit as the first mapper (C14), together with the `WarningsAsErrors` line — not earlier, because an unused dependency in slice 1 is a dependency nobody has justified yet | `AutoMapper`: commercial from v14, and it makes unmapped members a *runtime* concern, which defeats the entire enforcement mechanism |
| `NetArchTest.Rules` | MIT ✅ | 1 | Asserts the dependency direction in three lines | `TngTech.ArchUnitNET` (Apache-2.0) — more capable, heavier. Fine if the solution already uses it |
| `xunit` or `xunit.v3` | Apache-2.0 ✅ | 1 | Test framework | Whatever the solution already uses. **v3 changes the project shape** (test projects become executables) — check which one you are on before copying test project settings |
| `Shouldly` | MIT ✅ | 1 | Assertions with readable failures | **`FluentAssertions` v8+ is commercial (Xceed).** Use FA `7.x`, pinned, *only* if the codebase already depends on it — consistency beats saving one dependency. Never let a transitive update cross v8 |
| `Microsoft.Extensions.TimeProvider.Testing` | MIT ✅ | 1 | `FakeTimeProvider`, so `producedAt` is deterministic in tests | A hand-rolled `IClock`. `TimeProvider` has been in the framework since .NET 8 |
| `Bogus` | MIT ✅ | 1 | Realistic values **inside** hand-written builders | `AutoFixture`: see §5.3 — it cannot construct a DDD aggregate honestly |
| `Testcontainers.PostgreSql` | MIT ✅ | 2 | Verifies SQL **translation** and the keyset walk against real PostgreSQL | The EF in-memory provider translates nothing, and translation is the only thing worth testing here. **Requires Docker on every dev machine and in CI** — confirm that before slice 2 |
| `Microsoft.AspNetCore.Authentication.JwtBearer` | MIT ✅ | 3 | Token validation with JWKS caching | — |
| Built-in rate limiter (`Microsoft.AspNetCore.RateLimiting`) | framework | 3 | Per-caller fixed window | `AspNetCoreRateLimit` — unnecessary since .NET 7 |
| `FsCheck.Xunit` | BSD-3-Clause ✅ | 4 | The `[Foo]` round-trip tests in §6.2 | `CsCheck` (Apache-2.0), or plain `[Theory]` with hand-picked boundaries. All three are acceptable; §6.2's syntax assumes FsCheck |
| `libphonenumber-csharp` | Apache-2.0 ✅ | 4 | E.164 normalisation with a default region, correctly | A regex. E.164 with region inference is not a regex problem, and the failure mode is a `422` from the peer on real customer data |
| `Ganss.Xss` (HtmlSanitizer) | MIT ✅ | 4 | Sanitise and normalise `description` | `AngleSharp` (MIT) if you need full DOM parsing rather than sanitisation |
| `NSwag.MSBuild` + `NJsonSchema` | MIT ✅ | 5 | Generates DTOs from the vendored OpenAPI **and** validates payloads against the same file — one dependency, both jobs | `Kiota` (MIT) generates clients but no validator, so you would add `JsonSchema.Net` and resolve OpenAPI `$ref`s yourself. Choose Kiota only if stage 5's outbound poller needs a typed client |
| `Npgsql.EntityFrameworkCore.PostgreSQL`, `OpenTelemetry.*`, `Serilog` | PostgreSQL / Apache-2.0 ✅ | — | Already in the solution | — |

**Deliberately not added in stage 1**, each for a stated reason:

| Not added | Why |
|---|---|
| `MediatR` | Commercial from v13 — and stage 1 needs no dispatcher at all (§2.4). If the solution is already on v12 or lower, do not route this read through it |
| `AutoMapper` | Commercial from v14; runtime reflection; unmapped members become a runtime concern |
| `MassTransit` | Commercial from v9, and there is no broker in any stage of this design |
| `Dapper` | EF projections already reuse the model, the value converters and the query filters. A second data-access path means two places to keep the mapping honest |
| `Polly` / `Microsoft.Extensions.Http.Resilience` | Stage 1 makes no outbound HTTP call. Arrives in stage 4 |
| `Quartz.NET`, `Hangfire` | No scheduled work until stage 6, and `BackgroundService` plus `SKIP LOCKED` covers it then |
| `JsonSchema.Net` | Only if you pick Kiota over NSwag. Two schema stacks in one project is a way to have two opinions about what valid means |

### 5.3 Two dependency decisions worth arguing

**`AutoFixture` for aggregates: no.** A DDD aggregate typically has a private constructor and a
factory method that enforces invariants. `AutoFixture` either fails on it or bypasses the factory by
reflection — and a snapshot projected from an aggregate that could never exist proves nothing. Use
**hand-written builders that call the real factory methods**, with `Bogus` supplying values inside
them. The builder is also the artefact every later stage's tests reuse.

**`Base64Url` availability.** `System.Buffers.Text.Base64Url` (used by the cursor codec in §4.5) is
**.NET 9+**. On .NET 8, use `WebEncoders.Base64UrlEncode` from
`Microsoft.AspNetCore.WebUtilities` — same output, different call. Confirm the target framework
before copying §4.5.

---

## 6. Tests

### 6.0 The slice-1 pair: in memory first, then the same assertions through PostgreSQL

Slice 1 ends with the projection **asserted in memory**, before there is a store, an endpoint or a
mapper. It is fast, it needs no Docker, and it catches the ordinary mistake — a member assigned from
the wrong source.

```csharp
public sealed class FooSnapshotProjectionTests
{
    private static readonly Func<Foo, FooSnapshot> Project =
        FooSnapshotProjection.ToSnapshot.Compile();

    [Fact]
    public void Projects_a_fully_populated_foo()
    {
        var foo = new FooBuilder()
            .WithTitle("Example title, variant 3")
            .WithPrice(1_250_000m, "EUR")
            .WithArea(84.5m)
            .WithAttachments(3)
            .WithTags(FooTagKind.TagA, FooTagKind.TagB)
            .WithAgent("A. Example", "+1 555 0100", "A.Example@example.test")
            .Build();

        var s = Project(foo);

        s.Id.ShouldBe(foo.Id.Value);
        s.PriceAmount.ShouldBe(1_250_000m);
        s.PriceCurrency.ShouldBe("EUR");
        s.Attachments.Select(p => p.SortOrder).ShouldBe([0, 1, 2]);   // deterministic order, set in SQL
        s.Tags.Count.ShouldBe(2);

        // The projection does NOT normalise: no lowercasing, no rounding, no unit conversion.
        // Those belong to the mapper (§4.4). Keeping the projection dumb is what makes the two
        // layers independently testable — and it is why this test can exist before the mapper does.
        s.Contact!.Email.ShouldBe("A.Example@example.test");
    }

    [Fact]
    public void Projects_a_minimally_populated_foo()
    {
        var s = Project(new FooBuilder().Minimal().Build());

        s.PriceAmount.ShouldBeNull();     // unpriced ≠ zero
        s.Attachments.ShouldBeEmpty();
        s.Contact.ShouldBeNull();
    }

    [Fact]
    public void No_member_is_left_at_its_default_when_the_source_is_fully_populated()
    {
        var s = Project(new FooBuilder().FullyPopulated().Build());

        foreach (var member in typeof(FooSnapshot).GetProperties())
            member.GetValue(s).ShouldNotBeNull($"{member.Name} was not projected");
    }
}
```

A `required` member left unassigned in the object initialiser is already a **compile** error, so the
compiler covers omission and the third test covers the case the compiler cannot see: assigned, but
from the wrong place.

> **The trap that makes this a pair, not a test.** `ToSnapshot.Compile()` runs the expression as
> LINQ-to-Objects, which **executes the real property bodies**. So a collection exposed as
> `IEnumerable<Attachment> Attachments => _attachments.Where(p => p.IsVisible)` applies its filter here and
> **passes** — while EF resolves `p.Attachments` to the navigation by name, ignores the body, and returns
> hidden attachments in SQL (trap §7.1).
>
> The in-memory test is therefore **necessary and not sufficient**. Slice 2's Testcontainers test
> must assert **the same field values through PostgreSQL**, and the divergence between the two is
> what detects the trap. Write them as one shared assertion helper called from both, so they cannot
> drift into asserting different things:
>
> ```csharp
> internal static void ShouldMatchTheFixture(this FooSnapshot s) { /* one set of assertions */ }
> ```
>
> If the two disagree, the in-memory result is the one that is wrong about production.

### 6.1 Field coverage — the highest-value test in the workstream

```csharp
[Fact]
public void Every_domain_field_is_either_mapped_or_explicitly_excluded()
{
    var unaccounted = DomainFieldWalker.Walk(typeof(Foo))
        .Where(path => !MappingRegistry.Covers(path))
        .Where(path => !IntegrationFieldExclusions.Contains(path))
        .ToList();

    unaccounted.Should().BeEmpty(
        "every domain field must be mapped or excluded with a reason in MAPPING_MATRIX.md");
}

[Fact]
public void Every_exclusion_carries_a_written_reason()
{
    IntegrationFieldExclusions.All
        .Should().OnlyContain(e => !string.IsNullOrWhiteSpace(e.Reason));
}

[Fact]
public void Pii_registry_and_mapping_matrix_agree()
{
    var fromMatrix = MappingMatrixParser.PiiPointers("MAPPING_MATRIX.md");
    PiiFields.Pointers.Should().BeEquivalentTo(fromMatrix);
}
```

`DomainFieldWalker` recurses public properties, stops at primitives and registered value objects,
and yields dotted paths (`Foo.Address.PostalCode`). Add a field to `Foo` and forget the
integration, and this test names the field.

### 6.2 Round-trip and boundaries

```csharp
[Foo(MaxTest = 500)]
public void Money_round_trips_without_precision_loss(decimal amount)
{
    var snapshot = Any.FooSnapshot() with { PriceAmount = amount, PriceCurrency = "EUR" };
    var contract = _mapper.ToContract(snapshot);
    decimal.Parse(contract.Price!.Amount, CultureInfo.InvariantCulture)
           .Should().Be(Math.Round(amount, 2, MidpointRounding.AwayFromZero));
}

[Theory]
[InlineData("2026-03-29")]   // DST spring-forward
[InlineData("2026-10-25")]   // DST fall-back
[InlineData("2026-01-01")]
public void Date_only_fields_never_shift(string iso)
{
    var contract = _mapper.ToContract(
        Any.FooSnapshot() with { EffectiveOn = DateOnly.Parse(iso) });
    contract.EffectiveOn.Should().Be(iso);
}

[Fact]
public void Over_long_title_raises_and_names_both_lengths()
{
    var act = () => _mapper.ToContract(
        Any.FooSnapshot() with { Title = new string('x', 201) });

    act.Should().Throw<ContractViolationException>()
       .WithMessage("*/title*201*200*");
}

[Fact]
public void Unmapped_enum_value_fails_closed()
{
    var act = () => FooKindMap.ToContract(FooKind.Zeta);
    act.Should().Throw<UnmappedEnumValueException>();     // MAPPING_MATRIX §7.3
}
```

Also cover: non-ASCII titles, null versus empty collection, negative coordinates,
`Quantity = 0` versus `null`, and a many-to-one status mapping asserted for *convergence* rather than
equality.

### 6.3 Keyset walk under concurrency (Testcontainers)

```csharp
[Fact]
public async Task Walk_returns_every_row_exactly_once_even_while_rows_are_written()
{
    await Seed(10_000);
    var seen = new HashSet<Guid>();
    string? cursor = null;

    do
    {
        var page = await Get($"/integration/v1/foos?cursor={cursor}&limit=250");
        foreach (var item in page.Items)
            seen.Add(Guid.Parse(item.AggregateId)).Should().BeTrue("no row is returned twice");

        await InsertRandomFoos(20);      // data shifts under the walk
        await DeleteRandomFoos(5);

        cursor = page.NextCursor;
    } while (cursor is not null);

    seen.Should().HaveCountGreaterThanOrEqualTo(9_995);   // deletes may legitimately drop rows
}
```

The invariant a keyset walk guarantees — and offset paging does not — is *no duplicate and no
skipped pre-existing row*. Assert exactly that, not a row count.

### 6.4 Payload validation and redaction

```csharp
[Fact]
public void Generated_payloads_validate_against_the_vendored_peer_schema()
{
    var schema = JsonSchema.FromFile("contracts/external/acme/openapi.v1.yaml#/components/schemas/Foo");

    foreach (var snapshot in Any.FooSnapshots(200))
        schema.Evaluate(Serialise(_mapper.ToContract(snapshot)))
              .IsValid.Should().BeTrue();
}

[Fact]
public async Task No_personal_data_appears_in_any_response()
{
    var json = await GetRaw("/integration/v1/foos?limit=500");

    foreach (var pointer in PiiFields.Pointers)
        JsonPointer.Parse(pointer).Evaluate(JsonNode.Parse(json)!)
                   .Should().BeNull($"{pointer} is personal data and stage 1 redacts it");
}
```

### 6.5 Boundaries

```csharp
[Fact]
public void Domain_and_application_do_not_reach_outward()
{
    Types.InAssembly(typeof(Foo).Assembly)
         .ShouldNot().HaveDependencyOnAny(
             "Acme.Integration", "Acme.Integration.Contracts", "System.Net.Http")
         .GetResult().IsSuccessful.Should().BeTrue();

    Types.InAssembly(typeof(IFooRepository).Assembly)
         .ShouldNot().HaveDependencyOn("Acme.Integration.Contracts")
         .GetResult().IsSuccessful.Should().BeTrue();
}

[Fact]
public void External_contract_types_never_escape_the_integration_assembly()
{
    // no external DTO in an EF entity, an application command, or a non-integration response
}
```

Write §6.5 on day 1, while it is trivially green. It is the only test whose cost rises the longer
you wait.

---

## 7. Traps specific to a DDD + EF Core codebase

| # | Trap | Why it bites | What to do |
|---|---|---|---|
| 7.1 | **A collection exposed as a computed property.** `public IEnumerable<Attachment> Attachments => _attachments.Where(p => p.IsVisible);` | In a LINQ projection, EF resolves the member access `p.Attachments` to the *navigation named `Attachments`* and never evaluates your body — so **the filter silently disappears** and you publish hidden attachments. Encapsulation that looks correct produces wrong data. | Project from the mapped navigation and apply the predicate explicitly in the projection. Add a test asserting a hidden attachment does not appear |
| 7.2 | **Two collection navigations in one projection.** `Tags` and `Attachments` together | Without `AsSplitQuery()` EF emits one query whose result set is the cartesian product: a 100-row page × 8 attachments × 12 features = 9 600 rows to read one page | `AsSplitQuery()` (already in §4.3). Verify the query count in a test if page latency matters |
| 7.3 | **Strongly-typed id with a value converter in a `WHERE`/`ORDER BY`** | Comparison may fail to translate or, worse, evaluate client-side after loading the table | §1.4.1 — read the generated SQL once. Fallback in §4.3.1 |
| 7.4 | **Value object behind a single-column converter** | `p.Price.Amount` is not translatable; EF either throws or pulls the whole entity | Project the value object whole and flatten in the mapper (§4.2 note) |
| 7.5 | **A global query filter you did not know about** | Soft-deleted rows already excluded — or *not*, and you publish retracted records | §1.2.5. If a filter exists, do not add a second one; if it does not, filter in the projection |
| 7.6 | **`decimal` → `double` anywhere on the path** | `0.1m + 0.2m` is exact; `0.1 + 0.2` is not. A price that differs in the seventh decimal breaks stage 6's checksum comparison, not the display | Keep `decimal` end to end; serialise money as a string (§4.4) |
| 7.7 | **`timestamp` instead of `timestamptz`** | The offset is a convention held in someone's memory, not data. Two services disagree by an hour twice a year | §1.4.5 — document the assumed zone in the mapping matrix, and raise the column type as a separate finding |
| 7.8 | **Sharing the API's `JsonSerializerOptions`** | The peer's naming policy leaks into your existing public endpoints, or theirs into the peer's payload | Integration-only source-generated context, applied per group (§4.9) |
| 7.9 | **`Include` creeping into the projection** during review | Turns a projection into an aggregate load; the "no rehydration" property in ADR-0002 §2 quietly dies | A test asserting the query plan/entity-type count, or a code review rule on `FooSnapshotProjection.cs` |
| 7.10 | **`AsNoTracking` forgotten** | The change tracker fills with thousands of entities per page; memory and latency both climb | Set `QueryTrackingBehavior.NoTracking` on a dedicated read context, or keep it explicit as in §4.3 |
| 7.11 | **Collections ordered in memory** | Stage 3 hashes the payload for reconciliation; a nondeterministic order makes every hash comparison a false divergence | Order in SQL, by a stable business key, as in §4.2 |
| 7.12 | **`Guid.NewGuid()` (v4) ids** | Keyset paging by id still works — order is stable — but pages are not chronological, so "give me recent records" is not answerable | Fine for stage 1. Note it for the consumer; the composite index is stage 3's decision |

---

## 8. Execution plan

> The planning view of the same work — epics, features, stories, acceptance criteria, sizing and
> which open question blocks which item — is [BACKLOG.md](BACKLOG.md) §3. Its tasks reference the
> `C` numbers below, so the two documents stay one-to-one. Sizing and actor value live there; how to
> build each commit lives here.

Nineteen commits in five groups. **Every commit compiles and leaves the test suite green** — the
order below is chosen for that property, which is why the stub contract appears in C7 and dies in
C16 rather than the mapping being built before there is anything to serve.

`[svc]` = the service's repository. `[doc]` = this repository. Two repositories, so keep the commit
trails separate.

### Group A — Skeleton and boundaries (day 1)

| # | What | Files | Verify | Commit |
|---|---|---|---|---|
| **C0** | Answer every row of §1 and record it. Not code — the gate. | `[doc]` §1 answers appended to `MAPPING_MATRIX.md` §2 as the field inventory | Every §1 row has an answer; §1.4.1's generated SQL read with your own eyes; §1.5.3 resolved | `docs: record Foo field inventory and EF mapping findings` |
| **C1** | Three projects; reference direction `Integration → Application → Domain`; `Integration → Infrastructure` per §2.2 | `[svc]` `.csproj` ×3, solution file | `dotnet build` | `chore: add integration projects` |
| **C2** | The architecture tests of §6.5, **before** there is anything to violate them | `[svc]` `Architecture/BoundaryTests.cs` | Green — or it found a pre-existing violation, which is a finding to raise, not to fix here | `test: assert domain and application do not reach outward` |
| **C3** | `FooSnapshot` + `FooSnapshotProjection.ToSnapshot` | `[svc]` `Snapshots/` ×2 | Compiles. A `required` member left unassigned in the expression **is** the compile error — that is the guard | `feat: add flat property snapshot and projection` |

> C1–C3 depend on nothing external. If Q2 and Q3 are still unanswered, this group still ships today.

### Group B — Reading (day 1–2)

| # | What | Files | Verify | Commit |
|---|---|---|---|---|
| **C4** | `IFooSnapshotStore` + the EF implementation, single read only | `[svc]` `Snapshots/IFooSnapshotStore.cs`, `FooSnapshotStore.cs` | Testcontainers test: seed one property, read it, every field matches | `feat: add property snapshot store` |
| **C5** | Keyset page (`Take(limit + 1)`), ordering, `AsSplitQuery` | `[svc]` same files | §6.3 walk test over 10 000 rows with concurrent inserts and deletes: **no duplicate, no skipped pre-existing row**. Log the SQL and confirm one `LIMIT`, no `OFFSET`, no `COUNT` | `feat: add keyset paging to the snapshot store` |
| **C6** | Opaque versioned cursor | `[svc]` `Paging/Cursor.cs` | Round-trip test; malformed, truncated, wrong-version and empty inputs all fail closed | `feat: add opaque page cursor` |

> If C5's SQL shows client-side evaluation, stop and switch to §4.3.1 *here*, not later — everything
> downstream depends on the store's shape.

### Group C — Surface (day 2–3)

| # | What | Files | Verify | Commit |
|---|---|---|---|---|
| **C7** | Envelope, page response, the JSON context, and a **hand-written stub `ExternalFoo`** with a naive mapper so the pipe is end-to-end before the peer's schema exists | `[svc]` `Contracts/` ×3, `Mapping/FooContractMapper.cs` (stub form) | A unit test serialises one envelope and asserts camelCase, explicit nulls, no unexpected member | `feat: add snapshot envelope and page response` |
| **C8** | Both endpoints behind `Integration:ReadApiEnabled` (default **off**) | `[svc]` `Endpoints/` ×3, `DependencyInjection.cs`, `Program.cs` (one line) | `curl` returns a page; unknown id `404`; malformed cursor `400`; `limit=99999` capped; flag off ⇒ `404` on both routes | `feat: add property read endpoints behind a flag` |
| **C9** | The authenticator seam, scope policy, per-caller rate limit, body size cap | `[svc]` `Security/IIntegrationAuthenticator.cs`, `ScopeClaimAuthenticator.cs`, `Program.cs` | Unauthenticated `401`; wrong scope `403`; over-rate `429`; TLS-only asserted in config review | `feat: authenticate and rate-limit the integration routes` |
| **C10** | Log redaction policy and the request log line | `[svc]` logging config, `Security/` | §6.4-style test: a token in a logged object graph does not appear in the output; the request line carries caller, route, page size, cursor, status, duration and **no payload** | `feat: redact credentials from integration logs` |

> **End of C8 is the demo.** A peer-shaped page comes back over HTTP. Everything after this is making
> it correct rather than making it work — and that ordering is deliberate, because a visible endpoint
> is what gets the peer to look at a payload and tell you the field names are wrong.

### Group D — The anti-corruption layer (day 3–4, and this is the real work)

| # | What | Files | Verify | Commit |
|---|---|---|---|---|
| **C11** | Enum lookup tables: exhaustive switch, no value-returning `default`, throw arm | `[svc]` `Mapping/FooKindMap.cs`, `FooStatusMap.cs`, `FooTagKindMap.cs` | Every enum member has an arm; an unmapped value throws `UnmappedEnumValueException`; adding a member to the enum **breaks the build** | `feat: add fail-closed enum translation tables` |
| **C12** | Value converters: money, area with an explicit unit, instants, date-only, coordinates, phone E.164, HTML normalisation | `[svc]` `Mapping/Decimals.cs`, `Units.cs`, `Coordinates.cs`, `Phone.cs`, `HtmlText.cs` | §6.2 boundary tests: DST dates, non-ASCII, `0` versus `null`, negative coordinates, `decimal` precision. Money serialises as a string, area always with its unit | `feat: add contract value converters` |
| **C13** | Length guards | `[svc]` `Mapping/ContractString.cs` | Over-long value raises naming the field, the actual length and the limit. **A test asserts no code path truncates** | `feat: fail on values exceeding the peer's maxLength` |
| **C14** | The real Mapperly mapper wiring C11–C13 together; `RMG012`/`RMG020` promoted to errors | `[svc]` `Mapping/FooContractMapper.cs`, `.csproj` | Remove a mapping ⇒ the build fails. Every `MapperIgnore*` carries a reason and a matrix row | `feat: map property snapshots through the anti-corruption layer` |
| **C15** | `PiiFields` registry + `ContractRedactor` | `[svc]` `Security/PiiFields.cs`, `ContractRedactor.cs` | §6.4 test: no PII pointer resolves in a 500-item response. `PiiFields` and the matrix agree | `feat: redact personal data from stage 1 responses` |
| **C16** | Vendor the peer's schema, generate DTOs, **delete the stub**, add schema validation to the suite | `[svc]` `contracts/external/{system}/openapi.v1.yaml`, generated project, `Schema/PayloadValidationTests.cs` | 200 generated payloads validate against the vendored schema. The stub type no longer exists anywhere | `feat: generate contract types from the vendored peer schema` |
| **C17** | The reflection field-coverage test and the exclusions registry | `[svc]` `Mapping/MappingRegistry.cs`, `IntegrationFieldExclusions.cs`, `Mapping/FieldCoverageTests.cs` | Add a field to `Foo` ⇒ the test fails **naming that field**. Every exclusion has a non-empty reason | `test: assert every domain field is mapped or excluded` |

> C17 is placed last in this group on purpose: run it once and it will name fields C14 missed. That
> list is the last input to `MAPPING_MATRIX.md`, and finding it here costs an hour, whereas finding it
> in stage 3 costs a peer conversation.

### Group E — Handover (day 5)

| # | What | Files | Verify | Commit |
|---|---|---|---|---|
| **C18** | Request count and duration by route and caller | `[svc]` telemetry registration | Metrics visible on the existing dashboard | `feat: add integration read API metrics` |
| **C19** | `MAPPING_MATRIX.md` filled for `Foo`, §7 gaps resolved; `CONSUMER_CONTRACT.md` written | `[doc]` `MAPPING_MATRIX.md`, `CONSUMER_CONTRACT.md`, `README.md` row | Zero unresolved `GAP-*` rows. The consumer document states replace-all semantics, no incremental sync, **no deletion signal**, the page cap, the rate limit, and the time box | `docs: complete the Foo mapping and add the consumer contract` |

Then: walk the pilot consumer through a full catalogue read, and check §9 line by line.

### Dependencies and what is actually blocked

```
C0 ──┬─► C1 ─► C2
     │        └─► C3 ─► C4 ─► C5 ─┐
     │                  C6 ───────┼─► C7 ─► C8 ─► C9 ─► C10
     │                            │
     └─► (Q2 answer needed by C9) │
                                  └─► C11 ─► C12 ─► C13 ─► C14 ─► C15 ─► C17
                                                                    │
        (Q3: peer schema) ──────────────────────────────────────────┴─► C16 ─► C19
```

- **C1–C8, C11–C15, C17 are unblocked.** They can be built today against the stub contract.
- **C9 needs Q2** (which auth mechanism). Until it is answered, build the seam and one implementation
  against whatever your service already uses, and keep the endpoints flagged off.
- **C16 and C19 need Q3** (the peer's schema) and a counterpart on the other team. These are the only
  externally blocked steps, and they are the exit criteria — which is why Q3 is requested on day 0 and
  chased daily.

### Effort versus calendar

Engineering effort is **five days**. Calendar is longer, and the gap is not padding: C19 waits on
another team answering eight open gaps in `MAPPING_MATRIX.md` §7, and C16 waits on a document we do
not own. **Report the two numbers separately** — the proposal's single "~4 weeks" figure conflates
them, which is finding S7 in the review.

---

## 9. Definition of done

- [ ] The pilot consumer reconstructs the full `Foo` catalogue through the paged endpoint
- [ ] `MAPPING_MATRIX.md` complete for `Foo`, **zero** unresolved `GAP-*` rows
- [ ] Coverage, round-trip, boundary, enum, redaction and architecture tests green
- [ ] Generated payloads validate against the vendored peer schema
- [ ] Keyset walk over the full catalogue: no duplicate, no skipped pre-existing row
- [ ] p95 page latency within the agreed budget at the default page size
- [ ] No field marked `pii` in any response; no token in any log line
- [ ] `Integration:ReadApiEnabled` off in production until the consumer is ready; turning it off
      stops serving with no deploy
- [ ] `CONSUMER_CONTRACT.md` states replace-all semantics, no incremental sync, **no deletion
      signal**, the page cap, the rate limit, and that stage 1 is time-boxed
- [ ] `git diff --stat` shows **no migration** and **no change under `Domain/`**

The last two are the ones to check personally. Stage 1's whole claim is that it costs nothing to
reverse, and a stray migration or a "small" domain tweak quietly retires that claim.

---

## 10. What stage 1 deliberately does not do

No change capture, no outbox, no worker, no push, no change notification, **no deletions**, no
inbound, no subscriber table, no retention, no partitioning, no `aggregateVersion`. If any of these
appears in the pull request, it belongs to a later stage and it takes that stage's defect gates with
it (ADR-0007 §Defect gates).

What stage 2 changes about this code, so nothing here is built to be thrown away:

| Stage 1 artefact | Stage 2+ |
|---|---|
| `FooSnapshot` + projection | Unchanged. Stage 3's materialiser calls the same expression |
| `FooContractMapper`, enum maps, converters, guards | Unchanged. This *is* the ACL |
| Coverage / round-trip / boundary tests | Unchanged, and they guard every later stage |
| `SnapshotEnvelope` | Gains `aggregateVersion` (stage 2), then `messageId`, `changeKind`, `occurredAt`, `correlationId`, `traceParent`, `propagationPath` (stage 3) — all additive |
| `PageResponse` | Gains a pinned `watermark` (stage 3) and the endpoint becomes ADR-0006 §2's snapshot endpoint |
| `ContractRedactor` | Replaced by the per-subscriber payload variant (stage 4, defect D7) |
| `IIntegrationAuthenticator` | Extended, not replaced — stage 5 resolves `source_system` through it |
| Keyset by primary key | Becomes `(created_at, id)` **if** a consumer needs chronological order |
