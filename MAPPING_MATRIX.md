# Field Mapping Matrix — ProReel Estate ⇄ External Contract

> **Status:** Template + worked example.
> The peer's schema is not yet vendored, so field names on the external side are placeholders
> marked `‹…›`. **Fill this file in from `contracts/external/{system}/openapi.v1.yaml` before
> writing any mapper code** — this document is the specification the mappers implement and the
> test suite enforces.
>
> Related: ADR-0004 (ACL, losslessness enforcement), ADR-0005 (PII handling).

---

## 1. How to use this document

1. Vendor the peer's OpenAPI schema.
2. For **every** public property of every integrated aggregate, add a row. No field may be
   absent from this table — a field that exists in the domain and not here fails the
   coverage test (ADR-0004 §5b).
3. Every row is one of four dispositions:

| Disposition | Meaning |
|---|---|
| `MAP` | Direct or transformed mapping to an external field. |
| `EXCLUDE` | Deliberately not sent. **Requires a written reason** and an entry in `IntegrationFieldExclusions`. |
| `GAP-OUT` | We have data the peer's schema has no home for. **Requires a decision** (extension field / drop / request schema change). |
| `GAP-IN` | The peer requires a field we cannot produce. **Requires a decision** (default value / derive / negotiate optional). |

4. Unresolved `GAP-*` rows are **blockers**, not notes. Track them in §7.

### Column definitions

| Column | Meaning |
|---|---|
| Domain path | Dotted path from the aggregate root, e.g. `Property.Address.PostalCode` |
| Type / Null | CLR type and nullability |
| Ext. field | JSON pointer in the peer's contract |
| Ext. type | Peer's declared type + constraints (`maxLength`, `enum`, `format`) |
| Transform | Exact rule: identity, unit conversion, enum lookup, format, computed |
| Req. | Peer requires the field (`R`) / optional (`O`) |
| PII | `Y` if personal data — gated per ADR-0005 §3 |
| Risk | Loss/corruption risk to test explicitly |

---

## 2. Property (worked example — replace with real fields)

| Domain path | Type / Null | Ext. field | Ext. type | Transform | Req. | PII | Risk |
|---|---|---|---|---|---|---|---|
| `Property.Id` | `PropertyId` (Guid) | `/id` | `string` uuid | `.Value.ToString("D")` | R | N | — |
| `Property.ExternalIds[]` | `IReadOnlyList<ExternalRef>` | `/externalIds` | `array` | project `{system, id}`; sorted by `system` | O | N | ordering must be deterministic |
| `Property.Title` | `string` | `/title` | `string` maxLen 200 | identity + **length guard** | R | N | **truncation forbidden** → fail |
| `Property.Description` | `string?` | `/description` | `string` maxLen 4000 | HTML-sanitise, normalise newlines to `\n` | O | N | markup stripping may lose meaning |
| `Property.Kind` | `PropertyKind` enum | `/propertyType` | `enum` | **lookup table §4.1**, fail-closed | R | N | new enum value → dead letter |
| `Property.Status` | `ListingStatus` enum | `/status` | `enum` | **lookup table §4.2**, fail-closed | R | N | semantic mismatch (see §4.2 note) |
| `Property.Price.Amount` | `decimal` | `/price/amount` | `string` decimal | `ToString("F2", Invariant)` — **never double** | R | N | precision loss if serialised as number |
| `Property.Price.Currency` | `string` (ISO-4217) | `/price/currency` | `string` len 3 | identity, uppercase | R | N | never infer from locale |
| `Property.Price` (null) | `Money?` | `/price` | nullable | `null` when unpriced — **not `0`** | O | N | `0` means "free", not "unknown" |
| `Property.AreaSqm` | `decimal?` | `/livingArea/value` + `/unit` | `number` + `enum` | if peer wants ft²: `× 10.7639104167`, round half-away-from-zero 2 dp; emit `unit` explicitly | R | N | **unit ambiguity = silent 10× error** |
| `Property.Rooms` | `int?` | `/rooms` | `integer` | identity | O | N | — |
| `Property.Floor` | `int?` | `/floor` | `integer` | identity; `0` = ground — confirm peer convention | O | N | off-by-one vs peer's 1-based floors |
| `Property.BuildingId` | `BuildingId?` | `/buildingRef` | `string` uuid | id only — **no nested building object** | O | N | — |
| `Property.LocationId` | `LocationId` | `/location/ref` | `string` uuid | id + denormalised fields below | R | N | — |
| `Property.Address.Line1` | `string` | `/address/street` | `string` maxLen 150 | identity + length guard | R | Y | PII gate |
| `Property.Address.PostalCode` | `string?` | `/address/postalCode` | `string` | identity, trimmed, uppercase | O | Y | PII gate |
| `Property.Address.CountryCode` | `string` (ISO-3166-1 α2) | `/address/country` | `string` len 2 | identity, uppercase | R | N | — |
| `Property.Coordinates.Lat` | `double?` | `/geo/lat` | `number` | round 6 dp; SRID 4326 stated in contract | O | N | precision + datum |
| `Property.Coordinates.Lng` | `double?` | `/geo/lon` | `number` | round 6 dp | O | N | — |
| `Property.ListedOn` | `DateOnly` | `/listedOn` | `string` date | `ToString("yyyy-MM-dd")` — **date, not timestamp** | R | N | **midnight+TZ shifts the date** |
| `Property.CreatedAtUtc` | `DateTimeOffset` | `/createdAt` | `string` date-time | ISO-8601, UTC, `"O"` | R | N | — |
| `Property.UpdatedAtUtc` | `DateTimeOffset` | `/updatedAt` | `string` date-time | ISO-8601, UTC | R | N | — |
| `Property.Features[]` | `IReadOnlyList<Feature>` | `/features` | `array<enum>` | lookup §4.3; **unmapped feature → drop + counter**, not fail | O | N | silent feature loss — metric required |
| `Property.Photos[]` | `IReadOnlyList<Photo>` | `/media` | `array<object>` | `{url, sortOrder, isPrimary}`, sorted by `sortOrder` then `Id` | O | N | URL must be publicly resolvable by peer |
| `Property.Agent.Name` | `string` | `/contact/name` | `string` | identity | O | **Y** | PII gate |
| `Property.Agent.Phone` | `string?` | `/contact/phone` | `string` E.164 | **normalise to E.164** with default region | O | **Y** | format mismatch → peer 422 |
| `Property.Agent.Email` | `string?` | `/contact/email` | `string` email | lowercase, trim | O | **Y** | PII gate |
| `Property.OwnerId` | `OwnerId` | — | — | **EXCLUDE** — internal identity, never leaves the boundary | — | Y | — |
| `Property.InternalScoringNotes` | `string?` | — | — | **EXCLUDE** — internal commercial assessment | — | N | — |
| `Property.AcquisitionCost` | `Money?` | — | — | **EXCLUDE** — commercially sensitive | — | N | — |
| `Property.RowVersion` | `uint` (xmin) | `/aggregateVersion` *(envelope)* | `integer` | envelope field, not `data` | R | N | must be monotonic |
| — | — | `‹/mlsNumber›` | `string` | **GAP-IN** — peer field we cannot produce. Decision needed §7.1 | R? | N | blocker if required |
| `Property.EnergyRating` | `EnergyRating?` | — | — | **GAP-OUT** — no home in peer schema. Decision needed §7.2 | — | N | data loss if dropped |

## 3. Building / Location (to be completed)

Same structure. Key decisions to record when filling in:

- **Denormalisation depth.** Does the peer want `Property` to embed building/location details,
  or only references? Embedding means a `Building` change must re-emit every `Property` in it —
  a fan-out that has to be handled in the materializer (`Building` changed → enqueue change-log
  rows for all child properties). Cheap to implement, expensive to discover later.
  **Confirm with the peer before implementation day 2.**
- **Location hierarchy.** If `Location` is a tree (country → region → city → district) and the
  peer expects a flat set of names, record the flattening rule here, including behaviour when a
  level is missing.
- **Shared value objects.** `Address` and `Money` appear on several aggregates — map them once
  in a shared mapper, referenced from each aggregate's mapper.

---

## 4. Enum translation tables

**Global rule (ADR-0004 §6): fail closed.** An unmapped value is a materialisation error and a
dead letter, never a silent default — except where a row explicitly says otherwise.

### 4.1 `PropertyKind` → `‹propertyType›`

| Domain | External | Note |
|---|---|---|
| `Apartment` | `‹APARTMENT›` | |
| `House` | `‹HOUSE›` | |
| `Townhouse` | `‹TOWNHOUSE›` | |
| `Land` | `‹LAND›` | |
| `Commercial` | `‹COMMERCIAL›` | |
| `Garage` | `‹?›` | **unresolved** — peer may lack this value → §7.3 |
| *(new value added later)* | — | **build fails**: the mapper's switch is exhaustive |

Implementation note: use an exhaustive `switch` expression over the enum with **no default arm**
that returns a value. C# will warn on a missing case, and the fallback arm throws
`UnmappedEnumValueException`. This is what makes "we added a value and forgot the mapping" a
compile-time or first-message failure rather than corrupt data at the peer.

### 4.2 `ListingStatus` → `‹status›`

| Domain | External | Note |
|---|---|---|
| `Draft` | *(not emitted)* | **Filter rule:** drafts are never published — see §5 |
| `Active` | `‹ACTIVE›` | |
| `Reserved` | `‹UNDER_OFFER›` | ⚠️ **semantic mismatch** — confirm the peer's definition matches ours |
| `Sold` | `‹SOLD›` | |
| `Withdrawn` | `‹WITHDRAWN›` | |
| `Archived` | `‹WITHDRAWN›` | ⚠️ **many-to-one** — not round-trippable; document as accepted asymmetry |

Many-to-one mappings are the ones that break round-trip tests. Each must be listed in
`AsymmetricMappings` so the round-trip test asserts *convergence* rather than *equality*.

### 4.3 `Feature` → `‹features[]›`

Unlike the above, unmapped features are **dropped, not fatal** — a missing amenity tag is not
worth blocking a listing update. But the drop is counted:
`integration_feature_unmapped_total{feature="…"}` with an alert at > 100/day, so a growing
vocabulary gap becomes visible.

---

## 5. Filter rules (which aggregates are published at all)

Not every domain record belongs on the wire. Filters are applied at **materialisation** time:

| Rule | Behaviour |
|---|---|
| `Status == Draft` | Not published. If it was previously published and moves to `Draft`, emit a **tombstone** (`changeKind = "Deleted"`) — otherwise the peer keeps a listing we retracted. |
| Soft-deleted (`IsDeleted`) | Tombstone. |
| `Property` with no `Location` | Blocked — dead letter with a data-quality error, not a partial payload. |
| Subscriber-scoped filters (e.g. region) | Evaluated per subscriber in the fan-out step; a record leaving a subscriber's scope emits a tombstone **to that subscriber only**. |

The "leaves scope → tombstone" rule is easy to miss and produces permanently stale records at
the peer. It is tested explicitly.

---

## 6. Global value rules (normative)

Restated from ADR-0004 §6 so implementers have one place to look:

1. **Money:** decimal string or minor units; currency always explicit; `null` ≠ `0`.
2. **Areas:** value + explicit unit; conversion rounded half-away-from-zero to 2 dp; converted
   values are never written back to the domain.
3. **Timestamps:** ISO-8601 UTC with offset. Date-only stays date-only.
4. **Enums:** table-driven, fail-closed (except §4.3).
5. **Strings:** trimmed; length-checked against the peer's `maxLength`; **truncation is an error**.
6. **Nulls:** `null` = explicitly cleared; absent = unchanged. Snapshots always emit all keys.
7. **Collections:** deterministic sort by a stable business key.
8. **Coordinates:** 6 dp, EPSG:4326.
9. **Phone numbers:** E.164 with a configured default region.
10. **Ids:** ours in `aggregateId`; peer ids only in `externalIds[]`; never overwrite our key.

---

## 7. Open gaps (blockers — must be resolved before implementation)

| # | Type | Field | Question for the peer team | Owner | Status |
|---|---|---|---|---|---|
| 7.1 | GAP-IN | `‹mlsNumber›` | Is it required? Can it be optional for our source system, or do we mint a surrogate? | | ⬜ open |
| 7.2 | GAP-OUT | `Property.EnergyRating` | Any extension mechanism (`additionalAttributes`), or is this data dropped? | | ⬜ open |
| 7.3 | Enum | `PropertyKind.Garage` | Which external value, or is `OTHER` acceptable? | | ⬜ open |
| 7.4 | Semantics | `Reserved` vs `‹UNDER_OFFER›` | Do the definitions match (offer accepted vs contract signed)? | | ⬜ open |
| 7.5 | Shape | Snapshot vs delta | ADR-0004 §3 — **highest-impact open question** | | ⬜ open |
| 7.6 | Shape | Denormalisation depth | Embed building/location, or references only? Drives fan-out design | | ⬜ open |
| 7.7 | PII | Agent contact details | Is the peer authorised to receive them? Legal basis + retention? | | ⬜ open |
| 7.8 | Media | Photo URLs | Are our URLs reachable from the peer, or must we push binaries? | | ⬜ open |

---

## 8. Verification checklist (per aggregate)

Nothing is "mapped" until all of these are green:

- [ ] Every domain property appears in this document with a disposition
- [ ] Every `EXCLUDE` has a written reason and an `IntegrationFieldExclusions` entry
- [ ] No unresolved `GAP-*` rows
- [ ] Mapperly compiles with `RMG012`/`RMG020` as errors
- [ ] Coverage test passes (reflection walk over the aggregate)
- [ ] Round-trip property test passes for all symmetric fields
- [ ] Asymmetric/many-to-one mappings listed and convergence-tested
- [ ] Enum exhaustiveness test passes for every enum
- [ ] Boundary tests: max-length strings, min/max decimals, null vs empty collection, DST
      boundary dates, negative coordinates, non-ASCII text
- [ ] Generated payload validates against the peer's vendored JSON Schema
- [ ] PII fields gated and asserted in a test
- [ ] Filter rules (§5) covered by tests, **including the tombstone paths**
