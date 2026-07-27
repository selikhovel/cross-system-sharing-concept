# Tasks

Stage 1 of ADR-0007. Day-level ordering and the full prerequisite list are in
[IMPLEMENTATION_PLAN.md](../../../IMPLEMENTATION_PLAN.md); this is the checklist for the capability
itself.

## 0. Prerequisites

- [ ] 0.1 Request the peer's OpenAPI schema (Q3) — the only blocker on this stage's exit
- [ ] 0.2 Confirm the available auth mechanism (Q2) — the endpoints ship authenticated
- [x] 0.3 Pilot consumer named (Q17): exactly one, one-to-one
- [ ] 0.4 Confirm the .NET and EF Core versions in the solution, and the `Foo` row count

## 1. Projects and boundaries

- [ ] 1.1 Create `Acme.Integration`, `Acme.Integration.Contracts` (stub) and
      `Acme.Integration.Tests`
- [ ] 1.2 Architecture test: domain and application reference neither integration assembly nor
      `System.Net.Http` — add it now, while it is trivially green
- [ ] 1.3 Packages: `Riok.Mapperly`, `JsonSchema.Net`, `Testcontainers.PostgreSql`,
      `NetArchTest.Rules`, `FluentAssertions` pinned to 7.x

## 2. Snapshot and mapper

- [ ] 2.1 `FooSnapshot` flat read model, no domain types
- [ ] 2.2 Non-tracking projection query, single-id and keyset-range overloads, projected straight
      into the snapshot with no rehydration and no include chains
- [ ] 2.3 `FooContractMapper` with unmapped-member diagnostics promoted to build errors
- [ ] 2.4 Every ignore carries a written reason referencing the mapping matrix
- [ ] 2.5 Value converters: money, area with an explicit unit, dates, date-only fields, phone,
      coordinates, deterministic collection ordering
- [ ] 2.6 Length guards that raise, naming field and both lengths — never truncate
- [ ] 2.7 Enum lookup tables with exhaustive switches and no value-returning default

## 3. Endpoints

- [ ] 3.1 `GET /integration/v1/foos/{id}` returning the page envelope with one item, or `404`
- [ ] 3.2 `GET /integration/v1/foos?cursor=&limit=` returning items, next cursor and a
      more-available flag
- [ ] 3.3 Keyset pagination over a stable ordered key, opaque cursor, no offset paging
- [ ] 3.4 Page size default and maximum, capped rather than rejected
- [ ] 3.5 One shared source-generated serialiser: peer naming policy, explicit nulls, strict numbers
- [ ] 3.6 **No `aggregateVersion` field** — replace-all semantics, no improvised timestamp version
- [ ] 3.7 Feature flag `Integration:ReadApiEnabled`, default off in production

## 4. Security floor

- [ ] 4.1 `IIntegrationAuthenticator` seam
- [ ] 4.2 Authentication and one read scope on both routes
- [ ] 4.3 Per-caller rate limiting and a request body size limit
- [ ] 4.4 TLS only; no certificate-validation callbacks in any environment
- [ ] 4.5 Log redaction policy, with a test asserting a token never reaches the output
- [ ] 4.6 Unconditional omission of every field marked as personal data

## 5. Tests

- [ ] 5.1 ⭐ Reflection field-coverage test: every public `Foo` field mapped or excluded with a
      reason
- [ ] 5.2 Round-trip property tests over generated boundary values, with asymmetric mappings
      convergence-tested
- [ ] 5.3 Length-guard, enum-exhaustiveness and null-versus-empty-collection tests
- [ ] 5.4 Personal-data omission test
- [ ] 5.5 Testcontainers: keyset walk over 10 000 rows returns every row exactly once, including
      while rows are inserted and deleted mid-walk
- [ ] 5.6 Payload validation against the vendored schema
- [ ] 5.7 Authorisation tests: unauthenticated `401`, wrong scope `403`, rate limit `429`, flag off
      `404`
- [ ] 5.8 Architecture tests green

## 6. Contract and handover

- [ ] 6.1 Vendor `contracts/external/{system}/openapi.v1.yaml`, generate DTOs, **delete the stub**
- [ ] 6.2 Fill `MAPPING_MATRIX.md` for `Foo` and resolve its §7 gaps with the peer counterpart —
      zero unresolved `GAP-*` rows
- [ ] 6.3 Write `CONSUMER_CONTRACT.md`: replace-all semantics, no incremental sync, no deletion
      signal, page contract, page cap, rate limit, stage 1 is time-boxed
- [ ] 6.4 Request count and duration metrics by route and caller; one log line per request without
      payload content
- [ ] 6.5 Walk the pilot consumer through a full catalogue read

## 7. Exit criteria

- [ ] 7.1 The pilot consumer reconstructs the full `Foo` catalogue from the paged endpoint
- [ ] 7.2 `MAPPING_MATRIX.md` complete for `Foo`, no unresolved gaps
- [ ] 7.3 Coverage, round-trip and personal-data tests green
- [ ] 7.4 p95 page latency within the agreed budget
- [ ] 7.5 No field marked as personal data in any response
