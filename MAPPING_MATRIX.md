# Field Mapping Matrix — Acme ⇄ External Contract

> **Status:** Template + worked example.
> The peer's schema is not yet vendored, so field names on the external side are placeholders
> marked `‹…›`. **Fill this file in from `contracts/external/{system}/openapi.v1.yaml` before
> writing any mapper code** — this document is the specification the mappers implement and the
> test suite enforces.
>
> Related: ADR-0004 (ACL, losslessness enforcement), ADR-0005 (PII handling).

---

> **The domain names here are placeholders; the value rules are not.** `Foo`, `Bar`, `Baz`,
> `FooKind`, `FooStatus`, `FooTagKind` stand in for your aggregates and enums — substitute them.
> The rules in §4 and §6, and every entry in the Risk column, are cross-domain: money precision,
> unit ambiguity, date-only versus timestamp, fail-closed enums, forbidden truncation, deterministic
> collection order, personal-data gating. Those are the reason this document exists, and they apply
> whatever `Foo` turns out to be. Placeholder table:
> [CLAUDE.md](CLAUDE.md#naming--all-names-are-placeholders).

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
| Domain path | Dotted path from the aggregate root, e.g. `Foo.Address.PostalCode` |
| Type / Null | CLR type and nullability |
| Ext. field | JSON pointer in the peer's contract |
| Ext. type | Peer's declared type + constraints (`maxLength`, `enum`, `format`) |
| Transform | Exact rule: identity, unit conversion, enum lookup, format, computed |
| Req. | Peer requires the field (`R`) / optional (`O`) |
| PII | `Y` if personal data — gated per ADR-0005 §3 |
| Risk | Loss/corruption risk to test explicitly |

---

## 2. Foo (worked example — replace with real fields)

| Domain path | Type / Null | Ext. field | Ext. type | Transform | Req. | PII | Risk |
|---|---|---|---|---|---|---|---|
| `Foo.Id` | `FooId` (Guid) | `/id` | `string` uuid | `.Value.ToString("D")` | R | N | — |
| `Foo.ExternalIds[]` | `IReadOnlyList<ExternalRef>` | `/externalIds` | `array` | project `{system, id}`; sorted by `system` | O | N | ordering must be deterministic |
| `Foo.Title` | `string` | `/title` | `string` maxLen 200 | identity + **length guard** | R | N | **truncation forbidden** → fail |
| `Foo.Description` | `string?` | `/description` | `string` maxLen 4000 | HTML-sanitise, normalise newlines to `\n` | O | N | markup stripping may lose meaning |
| `Foo.Kind` | `FooKind` enum | `/kind` | `enum` | **lookup table §4.1**, fail-closed | R | N | new enum value → dead letter |
| `Foo.Status` | `FooStatus` enum | `/status` | `enum` | **lookup table §4.2**, fail-closed | R | N | semantic mismatch (see §4.2 note) |
| `Foo.Price.Amount` | `decimal` | `/price/amount` | `string` decimal | `ToString("F2", Invariant)` — **never double** | R | N | precision loss if serialised as number |
| `Foo.Price.Currency` | `string` (ISO-4217) | `/price/currency` | `string` len 3 | identity, uppercase | R | N | never infer from locale |
| `Foo.Price` (null) | `Money?` | `/price` | nullable | `null` when unpriced — **not `0`** | O | N | `0` means "free", not "unknown" |
| `Foo.Quantity` | `decimal?` | `/measure/value` + `/unit` | `number` + `enum` | if peer wants ft²: `× 10.7639104167`, round half-away-from-zero 2 dp; emit `unit` explicitly | R | N | **unit ambiguity = silent 10× error** |
| `Foo.Count` | `int?` | `/count` | `integer` | identity | O | N | — |
| `Foo.Rank` | `int?` | `/rank` | `integer` | identity; **ours is 0-based** — confirm the peer's convention | O | N | off-by-one against a 1-based peer: a silent, plausible-looking error |
| `Foo.BarId` | `BarId?` | `/barRef` | `string` uuid | id only — **no nested Bar object** | O | N | — |
| `Foo.BazId` | `BazId` | `/baz/ref` | `string` uuid | id + denormalised fields below | R | N | — |
| `Foo.Address.Line1` | `string` | `/address/street` | `string` maxLen 150 | identity + length guard | R | Y | PII gate |
| `Foo.Address.PostalCode` | `string?` | `/address/postalCode` | `string` | identity, trimmed, uppercase | O | Y | PII gate |
| `Foo.Address.CountryCode` | `string` (ISO-3166-1 α2) | `/address/country` | `string` len 2 | identity, uppercase | R | N | — |
| `Foo.Coordinates.Lat` | `double?` | `/geo/lat` | `number` | round 6 dp; SRID 4326 stated in contract | O | N | precision + datum |
| `Foo.Coordinates.Lng` | `double?` | `/geo/lon` | `number` | round 6 dp | O | N | — |
| `Foo.EffectiveOn` | `DateOnly` | `/effectiveOn` | `string` date | `ToString("yyyy-MM-dd")` — **date, not timestamp** | R | N | **midnight+TZ shifts the date** |
| `Foo.CreatedAtUtc` | `DateTimeOffset` | `/createdAt` | `string` date-time | ISO-8601, UTC, `"O"` | R | N | — |
| `Foo.UpdatedAtUtc` | `DateTimeOffset` | `/updatedAt` | `string` date-time | ISO-8601, UTC | R | N | — |
| `Foo.Tags[]` | `IReadOnlyList<FooTag>` | `/tags` | `array<enum>` | lookup §4.3; **unmapped tag → drop + counter**, not fail | O | N | silent tag loss — metric required |
| `Foo.Attachments[]` | `IReadOnlyList<Attachment>` | `/media` | `array<object>` | `{url, sortOrder, isPrimary}`, sorted by `sortOrder` then `Id` | O | N | URL must be publicly resolvable by peer |
| `Foo.Contact.Name` | `string` | `/contact/name` | `string` | identity | O | **Y** | PII gate |
| `Foo.Contact.Phone` | `string?` | `/contact/phone` | `string` E.164 | **normalise to E.164** with default region | O | **Y** | format mismatch → peer 422 |
| `Foo.Contact.Email` | `string?` | `/contact/email` | `string` email | lowercase, trim | O | **Y** | PII gate |
| `Foo.OwnerId` | `OwnerId` | — | — | **EXCLUDE** — internal identity, never leaves the boundary | — | Y | — |
| `Foo.InternalNotes` | `string?` | — | — | **EXCLUDE** — internal commercial assessment | — | N | — |
| `Foo.InternalCost` | `Money?` | — | — | **EXCLUDE** — commercially sensitive | — | N | — |
| `Foo.RowVersion` | `uint` (xmin) | `/aggregateVersion` *(envelope)* | `integer` | envelope field, not `data` | R | N | must be monotonic |
| — | — | `‹/externalCode›` | `string` | **GAP-IN** — peer field we cannot produce. Decision needed §7.1 | R? | N | blocker if required |
| `Foo.Rating` | `Rating?` | — | — | **GAP-OUT** — no home in peer schema. Decision needed §7.2 | — | N | data loss if dropped |

## 3. Bar / Baz (to be completed)

Same structure. Key decisions to record when filling in:

- **Denormalisation depth.** Does the peer want `Foo` to embed `Bar`/`Baz` details,
  or only references? Embedding means a `Bar` change must re-emit every `Foo` in it —
  a fan-out that has to be handled in the materializer (`Bar` changed → enqueue change-log
  rows for every `Foo` that references it). Cheap to implement, expensive to discover later.
  **Confirm with the peer before stage 1, step 2.**
- **Baz hierarchy.** If `Baz` is a tree (country → region → city → district) and the
  peer expects a flat set of names, record the flattening rule here, including behaviour when a
  level is missing.
- **Shared value objects.** `Address` and `Money` appear on several aggregates — map them once
  in a shared mapper, referenced from each aggregate's mapper.

---

## 4. Enum translation tables

**Global rule (ADR-0004 §6): fail closed.** An unmapped value is a materialisation error and a
dead letter, never a silent default — except where a row explicitly says otherwise.

### 4.1 `FooKind` → `‹kind›`

| Domain | External | Note |
|---|---|---|
| `Alpha` | `‹ALPHA›` | |
| `Beta` | `‹BETA›` | |
| `Gamma` | `‹GAMMA›` | |
| `Delta` | `‹DELTA›` | |
| `Epsilon` | `‹EPSILON›` | |
| `Zeta` | `‹?›` | **unresolved** — peer may lack this value → §7.3 |
| *(new value added later)* | — | **build fails**: the mapper's switch is exhaustive |

Implementation note: use an exhaustive `switch` expression over the enum with **no default arm**
that returns a value. C# will warn on a missing case, and the fallback arm throws
`UnmappedEnumValueException`. This is what makes "we added a value and forgot the mapping" a
compile-time or first-message failure rather than corrupt data at the peer.

### 4.2 `FooStatus` → `‹status›`

| Domain | External | Note |
|---|---|---|
| `Draft` | *(not emitted)* | **Filter rule:** drafts are never published — see §5 |
| `Active` | `‹ACTIVE›` | |
| `Held` | `‹IN_PROGRESS›` | ⚠️ **semantic mismatch** — confirm the peer's definition matches ours. (Named `Held`, not `Pending`, so it cannot be confused with the outbox row status of the same name) |
| `Closed` | `‹CLOSED›` | |
| `Retired` | `‹RETIRED›` | |
| `Archived` | `‹RETIRED›` | ⚠️ **many-to-one** — not round-trippable; document as accepted asymmetry |

Many-to-one mappings are the ones that break round-trip tests. Each must be listed in
`AsymmetricMappings` so the round-trip test asserts *convergence* rather than *equality*.

### 4.3 `FooTagKind` → `‹tags[]›`

Unlike the above, unmapped tags are **dropped, not fatal** — a missing tag is not
worth blocking a record update. But the drop is counted:
`integration_tag_unmapped_total{tag="…"}` with an alert at > 100/day, so a growing
vocabulary gap becomes visible.

---

## 5. Filter rules (which aggregates are published at all)

Not every domain record belongs on the wire. Filters are applied at **materialisation** time:

| Rule | Behaviour |
|---|---|
| `Status == Draft` | Not published. If it was previously published and moves to `Draft`, emit a **tombstone** (`changeKind = "Deleted"`) — otherwise the peer keeps a record we retracted. |
| Soft-deleted (`IsDeleted`) | Tombstone. |
| `Foo` with no `Baz` | Blocked — dead letter with a data-quality error, not a partial payload. |
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
| 7.1 | GAP-IN | `‹externalCode›` | Is it required? Can it be optional for our source system, or do we mint a surrogate? | | ⬜ open |
| 7.2 | GAP-OUT | `Foo.Rating` | Any extension mechanism (`additionalAttributes`), or is this data dropped? | | ⬜ open |
| 7.3 | Enum | `FooKind.Zeta` | Which external value, or is `OTHER` acceptable? | | ⬜ open |
| 7.4 | Semantics | `Held` vs `‹IN_PROGRESS›` | Do the two definitions describe the same state, or only a similar one? | | ⬜ open |
| 7.5 | Shape | Snapshot vs delta | ADR-0004 §3 — **highest-impact open question** | | ⬜ open |
| 7.6 | Shape | Denormalisation depth | Embed `Bar`/`Baz`, or references only? Drives fan-out design | | ⬜ open |
| 7.7 | PII | contact details | Is the peer authorised to receive them? Legal basis + retention? | | ⬜ open |
| 7.8 | Media | Attachment URLs | Are our URLs reachable from the peer, or must we push binaries? | | ⬜ open |

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
