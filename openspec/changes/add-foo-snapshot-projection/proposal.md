# Change: add-foo-snapshot-projection

**Slice 1 of stage 1.** Self-contained: everything needed to implement it is in this folder, so it
can be handed to someone — or something — with access to the service codebase and nothing else.

## Substitute the names first

This repository uses placeholders. Before implementing, map them onto the real codebase:

| Placeholder | Substitute |
|---|---|
| `Acme` | the service / solution root namespace |
| `Catalog` | the module that owns the aggregate being published |
| `Foo` | the aggregate root to publish |
| `Bar`, `Baz` | aggregate roots that `Foo` references |
| `FooAttachment`, `FooTag` | child entities inside `Foo` |
| `FooKind`, `FooStatus`, `FooTagKind` | enums reachable from `Foo` |
| `peer-a` | the external consumer |

Nothing in this change depends on what those things mean.

## Why

The integration's first increment is a read API that serves `Foo` in the peer's contract shape.
Before any endpoint exists, one thing has to be right: **the single, canonical way to get from the
domain aggregate to a flat, serialisable snapshot.**

That projection is the load-bearing artefact. Stage 3's background materializer calls the same
definition, so if it is written as a method body rather than a reusable expression, it gets copied
and the copies drift. Getting it wrong is cheap to fix now and expensive to fix after two callers
depend on it.

Slice 1 stops at the projection asserted **in memory**. No database round trip, no endpoint, no
mapper into the peer's format. That boundary is deliberate: it is the largest amount of work that can
be verified without Docker, without the peer's schema, and without deciding the authentication
mechanism — all three of which are still open.

## What Changes

- **Adds** `Acme.Catalog.Integration` and `Acme.Catalog.Integration.Contracts` projects, plus tests.
  No NuGet package in the production project.
- **Adds** `FooSnapshot`: a flat record carrying only the fields that leave the service.
- **Adds** `FooSnapshotProjection.ToSnapshot`, a **static `Expression<Func<Foo, FooSnapshot>>**` —
  the one place domain shape is read.
- **Adds** architecture tests asserting the dependency direction, while they are trivially green.
- **Adds** a hand-written test builder that constructs `Foo` through its real factory methods, and
  three tests over the projection.
- **Records** the discovery findings that change the code, as a written artefact rather than
  knowledge in someone's head.

## Impact

| Area | Effect |
|---|---|
| Database | **None.** No migration, no new table, no index |
| `Catalog.Domain` | **None.** Verified by `git diff --stat` |
| `Catalog.Application` | None |
| `Catalog.Infrastructure` | None, beyond being referenced |
| New projects | `Acme.Catalog.Integration`, `Acme.Catalog.Integration.Contracts` (empty in this slice), test project |
| Runtime behaviour | **None.** Nothing is wired into the host yet |

## Explicitly out of scope

Adding any of these means the slice boundary moved, and the reviewer should push back:

- The snapshot store, any `DbContext` query, any Testcontainers test — **slice 2**
- Keyset pagination and the opaque cursor — **slice 2**
- HTTP endpoints, the page envelope, JSON configuration — **slice 2**
- Authentication, scopes, rate limiting, log redaction — **slice 3**
- The mapper into the peer's contract, enum tables, value converters, length guards — **slice 4**
- Vendoring the peer's schema and generating contract types — **slice 5**
- Change capture, outbox, workers, push delivery, domain events — **stages 2 onward, and domain
  events never**
