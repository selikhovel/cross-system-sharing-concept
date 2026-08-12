# ADR-0004 — Anti-Corruption Layer, payload shape and contract versioning

- **Status:** Proposed
- **Date:** 2026-07-25
- **Open question:** payload shape (snapshot vs delta) is **not yet confirmed by the peer
  team** — this ADR specifies both and states the default plus the switch criteria.
- **Related:** ADR-0002 (materialisation), ADR-0003 (delivery), MAPPING_MATRIX.md
- **Amended by:** [ADR-0008](ADR-0008-bidirectional-synchronisation-and-conflict-resolution.md) §2 —
  the envelope (§3 below) gains `changedFields[]`. The snapshot decision stands; the snapshot is
  no longer the message's only content, because applying one wholesale reverts fields the sender
  never touched
- **Amended by:** [ADR-0009](ADR-0009-reference-data-replication-and-region-scoping.md) §4 — the
  fail-closed enum rule (§6 below) applies only to enums this service owns. Values drawn from an
  aggregator-owned picklist are resolved at runtime against a replica, and an unknown code means a
  stale replica rather than bad data: refresh and retry once, then permanent

## Context

The external system **owns the wire format**. We do not get to publish a Published Language
and make peers conform; we must translate our domain into a schema someone else controls and
will change without asking us. Two consequences follow immediately:

1. **A strict Anti-Corruption Layer is mandatory.** If the foreign schema's vocabulary leaks
   into `Acme.Domain` — even as an enum value or a nullable field added "because the
   partner needs it" — the domain model starts drifting toward a shape that serves an external
   integration rather than the business domain. That drift is irreversible in practice.
2. **The mapping must be provably total.** "All fields mapped, nothing lost" cannot be a
   review-time assertion; it has to be a build-time and test-time guarantee (§5).

The peer has not yet confirmed whether they want full snapshots or deltas, so the payload
shape decision is made *conditionally*, with a clear default and a documented escape hatch.

## Decision

### 1. Three isolated model layers, translated only at the boundary

```
Acme.Domain                 ← rich aggregates, business language, NEVER changed for integration
        │  (read-only projection query)
        ▼
Acme.Integration.Snapshots  ← flat, internal, serialisation-friendly read models
        │  (Mapperly / explicit mappers = the Anti-Corruption Layer)
        ▼
Acme.Integration.Contracts.External.V1   ← generated from the PEER's OpenAPI schema
```

Rules, enforced by architecture tests:

- `Domain` references nothing from `Integration`.
- `Application` references nothing from `Integration.Contracts.External`.
- **External contract types never appear** in a domain signature, an application command,
  an EF entity, or a public API response. They exist only inside `Integration`.
- The middle **snapshot layer is not optional overhead** — it is the seam. When the peer
  ships schema v2, only `Snapshots → Contracts.V2` changes; the domain-to-snapshot projection,
  which is the part that knows about our business, stays put. Mapping straight from the
  aggregate to the foreign DTO couples domain refactors to partner releases in both directions.

### 2. Contract types are generated, not hand-written

The peer's OpenAPI/JSON-Schema document is **vendored into the repository** at
`contracts/external/{system}/openapi.v1.yaml` and client + DTOs are generated at build time
(Kiota or NSwag; both MIT/Apache-2.0). Hand-writing partner DTOs guarantees drift.

Consequences we want from vendoring:

- The schema file is diffable in a pull request — **an unannounced partner schema change
  shows up as a code review**, not as a 422 at 3 a.m.
- A CI job fetches the peer's live schema daily and fails if it differs from the vendored copy.

> **While the peer is still in development, the schema is a moving target — and that strengthens
> this section rather than weakening it.** The peer's project dictates the format and we can
> influence their implementation only marginally, so the premise above holds: we translate into a
> schema someone else controls. What changes is the *frequency*: the drift job will fire regularly,
> and each firing is **expected signal, not an incident** — it is the mechanism working. Route it to
> a review queue, not to an on-call page, until their schema stabilises, then tighten it.
>
> Two consequences follow. Every "the schema is complete" statement is **as of a vendored revision**,
> so record which revision. And the mapping matrix is not filled in once: it is re-checked against
> each new revision until the peer declares theirs frozen.
- The generated DTOs are compiled against, so a removed field is a **compile error**.

### 3. Payload shape — default: event-carried state transfer (snapshot)

**Default (recommended): full aggregate snapshot in every message**, wrapped in an envelope
carrying an event type.

```jsonc
{
  "messageId":        "01927f3e-8a2b-7c1d-9e4f-000000000001",  // UUIDv7, stable across retries
  "messageType":      "acme.foo.changed",
  "specVersion":      "1.0",
  "contractVersion":  "1",
  "occurredAt":       "2026-07-25T09:14:02.113Z",              // business change time
  "producedAt":       "2026-07-25T09:14:03.907Z",              // materialisation time
  "source":           "acme",
  "aggregateType":    "Foo",
  "aggregateId":      "9c1b…",
  "aggregateVersion": 42,                                       // monotonic, idempotency key
  "changeKind":       "Updated",                                // Created | Updated | Deleted
  "correlationId":    "…",
  "traceParent":      "00-4bf92f…-00f067…-01",                  // W3C trace context
  "partial":          false,
  "data":             { /* the peer's Foo schema, fully populated */ }
}
```

Rationale for snapshots as the default:

| Foo | Snapshot | Delta |
|---|---|---|
| Survives a lost/skipped message | ✅ next message re-converges | ❌ permanent divergence |
| Requires strict ordering | ❌ no (version wins) | ✅ yes — expensive (ADR-0003 §4) |
| Enables compaction | ✅ | ❌ |
| Consumer bootstrap complexity | low | high (needs separate snapshot API) |
| Payload size | larger | smaller |
| Proving "no field lost" | straightforward | requires diff-completeness proofs too |

Snapshots make ADR-0003's compaction legal and make the loss story trivial. The cost is
bandwidth, which for a property record (~2–8 KB) at this volume is irrelevant.

**If the peer mandates deltas**, the following changes are required and must be re-costed:

- Compaction is disabled; per-aggregate ordered delivery with stream locks becomes mandatory.
- A separate `GET /properties/{id}` snapshot endpoint is required for consumer bootstrap and
  for recovery after a gap.
- The envelope adds `previousAggregateVersion`, and consumers must reject a message whose
  `previousAggregateVersion` ≠ their stored version, then re-bootstrap.
- Add ~1 sprint for the ordering and gap-detection machinery.

> **Action required:** confirm the shape with the peer team before stage 3.
> Until confirmed, build for snapshots — it is the strictly simpler path and the delta variant
> can reuse the same envelope, mapper, and outbox.

### 4. Versioning

- **Contract version in three places:** URL path (`/integration/v1/...`), the `contractVersion`
  envelope field, and the media type (`application/vnd.acme.foo.v1+json`).
- **Additive changes** (new optional field) → no version bump; consumers must ignore unknown
  fields (`JsonSerializerOptions` on the inbound side does this by default and we assert it).
- **Breaking changes** (field removed, renamed, type or semantics changed) → new major version.
- **Dual publishing during migration:** the subscriber table stores each subscriber's
  `contract_version`, so the materializer can emit v1 and v2 side by side for the same change.
  This is only possible because materialisation is deferred (ADR-0002) — a direct consequence
  we are deliberately cashing in.
- **Deprecation window:** 2 releases or 90 days, whichever is longer, tracked per subscriber.
- `messageType` uses reverse-DNS-ish naming: `acme.{aggregate}.{event}`.

### 5. Losslessness is enforced by the build, not by review

Four mechanisms, in order of how early they catch a mistake:

**(a) Mapperly with unmapped members as compile errors.** Mapperly (Apache-2.0, compile-time
source generator) emits diagnostics for unmapped source and target members. We promote them
to errors:

```xml
<PropertyGroup>
  <!-- RMG012: unmapped target member; RMG020: unmapped source member -->
  <WarningsAsErrors>$(WarningsAsErrors);RMG012;RMG020</WarningsAsErrors>
</PropertyGroup>
```

Adding a field to a snapshot without mapping it **fails the build**. Deliberate omissions must
be explicit and reasoned:

```csharp
[Mapper]
public partial class FooContractMapper
{
    // Every ignore carries a documented reason — see MAPPING_MATRIX.md
    [MapperIgnoreSource(nameof(FooSnapshot.InternalNotes))]  // internal-only, ADR-0005 §PII
    [MapperIgnoreTarget(nameof(ExternalFoo.ExternalCode))]             // peer-owned, we never populate
    public partial ExternalFoo ToContract(FooSnapshot source);
}
```

**(b) Reflection-based field-coverage test.** A test walks every public property of every
domain aggregate (recursively) and asserts it is either present in the mapping registry or
listed in `IntegrationFieldExclusions` with a written reason. **Adding a field to `Foo`
and forgetting the integration fails a test with the field name in the message.** This is the
single highest-value test in the whole workstream.

```csharp
[Fact]
public void Every_domain_field_is_either_mapped_or_explicitly_excluded()
{
    var unaccounted = DomainFieldWalker.Walk(typeof(Foo))
        .Where(path => !MappingRegistry.Covers(path))
        .Where(path => !IntegrationFieldExclusions.Contains(path))
        .ToList();

    unaccounted.Should().BeEmpty(
        "every domain field must be mapped or explicitly excluded with a reason in MAPPING_MATRIX.md");
}
```

**(c) Round-trip property tests.** For every field that exists on both sides, generate random
valid domain values (Bogus/AutoFixture), map domain → contract → domain, and assert equality
on the mapped subset. This is what catches *silent value corruption* — truncation, precision
loss, timezone drift, enum fallback — which coverage tests cannot see.

**(d) Schema validation in CI and at runtime.** Every produced payload is validated against
the peer's vendored JSON Schema before it is written to the outbox. A payload that would fail
their validation is dead-lettered **at materialisation time** with a clear error, instead of
being discovered as a 422 after N retries.

### 6. Value-translation rules (the actual sources of data loss)

Full field-by-field table in **MAPPING_MATRIX.md**. The rules that govern it:

| Concern | Rule |
|---|---|
| **Money** | Never `double`. Transport as decimal string (`"1250000.00"`) or minor units + ISO-4217 currency, whichever the peer's schema requires. Currency is always explicit — never implied by locale. |
| **Area / measurements** | Store the unit alongside the value. If the peer uses ft² and we use m², convert with a single well-tested converter and **document the rounding rule** (half-away-from-zero, 2 dp). Never round-trip a converted value back into the domain. |
| **Dates & times** | ISO-8601 with explicit offset, UTC on the wire. Date-only fields (`EffectiveOn`) stay `date`, never `timestamp` — adding a midnight time zone-shifts the date across the date line. |
| **Enums** | Explicit lookup table per enum. **Fail closed**: an unmapped domain enum value is a materialisation error and a dead letter, never a silent `"Other"`. Inbound: an unknown peer value maps to a quarantine status, never a guessed one. |
| **Null vs absent** | Defined once, globally: `null` = *explicitly cleared*, key absent = *unchanged/unknown*. With snapshot payloads we always emit the full object, so absence never occurs — this rule exists for the inbound direction and for a possible delta variant. |
| **Strings** | Silent truncation is **forbidden**. If a domain value exceeds the peer's `maxLength`, materialisation fails and dead-letters with the field name and both lengths. Truncating an address is a data-quality incident, not a rounding detail. |
| **Collections** | Deterministic ordering (by a stable business key, not database order) so payload hashes are comparable for change detection and tests. |
| **Coordinates** | Fixed precision (6 dp ≈ 0.11 m); SRID stated explicitly (WGS 84 / EPSG:4326). |
| **HTML / rich text** | Sanitised and normalised on the way out; the peer's schema states whether it accepts markup. |
| **Identity** | We always send **our** id in `aggregateId`, plus any peer id we know in `externalIds[]`. We never overwrite our primary key with a foreign one — the id correlation table is `integration.external_reference`. |

### 7. Serialisation

`System.Text.Json` with a **source-generated** `JsonSerializerContext` (no reflection, AOT-safe,
measurably faster on the hot materialisation path), `JsonNamingPolicy` matching the peer's
schema exactly, `DefaultIgnoreCondition = Never` for snapshots (explicit nulls), and
`NumberHandling = Strict`. One shared options instance, never per-call.

## Alternatives considered

- **Map the aggregate directly to the peer's DTO (no snapshot layer).** Fewer types, but the
  peer's schema becomes the shape of every mapping change and domain refactors break partner
  code paths directly. Rejected.
- **AutoMapper.** Commercial from v14, runtime reflection, and — decisively here — it makes
  unmapped members a *runtime* concern. Mapperly's compile-time diagnostics are the core of §5(a).
  Rejected.
- **Hand-written mappers everywhere.** Kept for genuinely complex fields (computed, multi-source,
  conditional) as the skill guidance recommends, but not as the default: hand-written mappers
  cannot produce an "unmapped member" diagnostic.
- **Untyped `JsonNode` passthrough.** Fastest to write, zero compile-time safety, no schema
  drift detection. Rejected.
- **Protobuf/gRPC contracts.** Not offered by the peer; would still need the same ACL.

## Consequences

### Positive

- The domain is insulated from a schema we do not control.
- "Every field is mapped" is enforced by the compiler and by a test that names the missing field.
- Partner schema drift surfaces as a PR diff and a red CI job, not a production 422.
- Dual-version publishing during migrations is possible without a data migration.
- Invalid payloads are caught before delivery, so retries are never spent on doomed requests.

### Negative / costs accepted

- **Three model layers** — more types, more mapping code. This is the price of the ACL and it
  is charged deliberately.
- **Every new domain field requires a conscious decision** (map it or exclude it with a reason).
  Intentional friction; it is what makes losslessness real.
- **Vendored schema must be kept fresh**; a stale copy plus a silent partner change is still
  possible if the daily drift job is ignored. Alert routing matters.
- **The snapshot-vs-delta decision is still open** and a delta mandate costs roughly one extra
  sprint.
