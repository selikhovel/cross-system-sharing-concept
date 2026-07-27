# Backlog — Acme external integration

- **Scope:** the whole integration, all seven delivery stages of
  [ADR-0007](adr/ADR-0007-staged-delivery-and-the-first-increment.md)
- **Detail level:** Epic 1 is decomposed to tasks because it starts now. Epics 2–7 stop at features
  and stories on purpose — decomposing a stage before its predecessor has taught you anything
  produces confident tasks that get rewritten
- **Related:** [STAGE_1_READ_API.md](STAGE_1_READ_API.md) (the *how* for Epic 1) ·
  `openspec/changes/` (the verifiable requirements) · [MAPPING_MATRIX.md](MAPPING_MATRIX.md) ·
  [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md)

---

## 1. How this backlog is structured

### 1.0 Domain independence

**Nothing in this backlog depends on what the data means.** `Acme` is the service, `Foo` the
aggregate being published, `Bar` and `Baz` aggregates it references, `peer-a` an external consumer —
all placeholders ([CLAUDE.md](CLAUDE.md#naming--all-names-are-placeholders)). Substitute your own
and every epic, story and estimate stands.

That is also why the actors in §3 are roles rather than departments: *consumer team*, *peer contract
owner*, *data protection officer*, *on-call engineer*, *architect*. Those five exist in any
organisation doing this integration, whatever it is integrating.

### 1.1 The three-way split

Four documents describe the same work from four angles. Each owns exactly one thing, so a change
lands in one place instead of three:

| Document | Owns | Answers |
|---|---|---|
| `adr/ADR-000N` | **Decisions** | *Why this way, and what it costs* |
| **This document** | **Hierarchy, value, sizing, sequencing** | *What we are building, for whom, in what order, and when it is done* |
| `STAGE_1_READ_API.md` | **Engineering detail** | *How — file by file, with the traps* |
| `openspec/changes/<id>/specs/` | **Verifiable requirements** | *What must be true, as `SHALL` plus scenarios* |

**Tasks in this backlog carry a pointer, not a copy.** A task's row names the guide section that
explains it and the OpenSpec scenario that accepts it. If you find yourself writing *how* here,
it belongs in the guide.

### 1.2 Levels

| Level | Meaning here | Granularity |
|---|---|---|
| **Epic** | One delivery stage of ADR-0007. Has a consumer-visible outcome and a release gate | 1–3 weeks |
| **Feature** | A coherent capability inside a stage. Maps to one OpenSpec capability where possible | 2–5 days |
| **User story** | Value delivered to a named actor, with acceptance criteria | ½–2 days |
| **Enabler** | Necessary work with no direct actor value — project setup, a seam, a registry | ½–2 days |
| **Task** | One commit. Compiles, leaves tests green, reviewable alone | 1–6 hours |

**Enablers are labelled as enablers, not dressed as stories.** "As a developer I want a project
structure so that I can write code" is theatre: it has no actor outside the team and no acceptance
criterion a stakeholder can judge. Naming enablers honestly keeps the story set meaningful, and it
makes the ratio of enabler to story work visible — which is the number that tells you whether a
stage is mostly plumbing.

### 1.3 Identifiers

```
E1              epic         — stage number
F1.2            feature      — E1's second feature
US1.2.1         user story   — F1.2's first story
EN1.2.1         enabler      — F1.2's first enabler
T1.2.1-a        task         — first task of story US1.2.1
TEN1.2.1-a      task         — first task of enabler EN1.2.1
C7              commit       — cross-reference into STAGE_1_READ_API.md §8
```

Enabler tasks carry the `TEN` prefix rather than a suffix, so that `T1.2.1-a` and `TEN1.2.1-a` cannot
be confused in a commit message by one character.

Ids are permanent. A cancelled item keeps its id and gains a status, so that a commit message or a
review comment referencing `US1.3.2` never resolves to something else.

### 1.4 Estimates

Points are **relative sizes inside this backlog**, Fibonacci, not hours. Day figures appear only at
epic level and come from `STAGE_1_READ_API.md` §8 for Epic 1, assuming one engineer already familiar
with the codebase.

Epics 2–7 carry a T-shirt size only. A point estimate for stage 5 written today would be a guess
with a decimal point on it.

**Effort and calendar are reported separately, always.** Epic 1 is five engineering days of which two
tasks wait on another team. Conflating those is finding S7 in the
[design review](openspec/changes/add-cross-system-integration-layer/design.md).

### 1.5 Definition of ready

An item is ready when: its acceptance criteria are written, its OpenSpec scenario exists, its blocking
open questions (§6) are answered or have an agreed default, and its defect gates (§2, from ADR-0007)
are resolved.

### 1.6 Definition of done

| Level | Done when |
|---|---|
| Task | Compiles, tests green, reviewed, merged. The commit does not touch `Domain/` and adds no migration unless its epic says it may |
| Story | Every acceptance criterion demonstrable, its OpenSpec scenarios passing as tests |
| Feature | Its stories done and its OpenSpec requirement satisfied end to end |
| Epic | The stage's exit criterion in ADR-0007 met, and the consumer-facing documents updated in the same change |

---

## 2. Portfolio view

| Epic | Stage | Outcome for someone outside the team | Release gate | Size | Defect gates | Blocked by |
|---|---|---|---|---|---|---|
| **E1** | 1 | A peer reads our `Foo` catalogue in **their** contract shape | The consumer reconstructs the full catalogue; mapping matrix complete against a named schema revision; personal data present only under a recorded grant | **M** — 5 eng. days, 66 pts | none | Q2, Q3, Q6 |
| **E2** | 2 | `aggregateVersion` appears, so a consumer can ignore stale updates | Rollback leaves no capture row; no measurable write-latency change | **M** | D1, D6 | Q9 |
| **E3** | 3 | Consumers stop re-reading the catalogue; **deletions become visible** | The watermark test green as a gate; pilot syncs a week with no gap | **L** | D2, D5, D9, D11, D14 | Q10 |
| **E4** | 4 | Latency drops from a poll interval to seconds | p95 delivery < 10 s; zero dead letters; a 500 k backfill does not delay one live change | **M** | D3, D4, S5 | Q1 |
| **E5** | 5 | Peer data lands in our domain through existing command handlers | Peer messages applied; echo suppression verified against the peer | **L** | D6, S1 | Q7, Q11 |
| **E6** | 6 | Divergence becomes observable daily instead of by a business user | Checksums converge; divergence zero for a week | **S** | — | Q8 |
| **E7** | 7 | It stops needing heroics | On-call trained; alerts verified in a game day | **M** | D10 | — |

`D*`/`S*` are the defects in
[design.md](openspec/changes/add-cross-system-integration-layer/design.md); `Q*` are the open
questions in [TECHNICAL_PROPOSAL.md §2.3](TECHNICAL_PROPOSAL.md) and
[MAPPING_MATRIX.md §7](MAPPING_MATRIX.md). **A defect gate is a hard entry condition, not a risk
note:** E4 cannot start before D3 is resolved, because its central promise is the thing D3 breaks.

### 2.1 Cross-epic work

| Id | Item | Runs across | Note |
|---|---|---|---|
| **X1** | `CONSUMER_CONTRACT.md` — every obligation we place on consumers | E1 creates it, every epic extends it | Defect **D13**. A guarantee that changes without this document changing in the same commit is a guarantee nobody outside the team knows about |
| **X2** | Resolve the fourteen review defects | Milestone 0 of `openspec/changes/add-cross-system-integration-layer/tasks.md` | Sequenced by the gate column above — resolve each one before its epic, not all up front |
| **X3** | `MAPPING_MATRIX.md` completion per aggregate | E1 does `Foo`; `Bar` and `Baz` follow | Every `GAP-*` row is a blocker, not a note |
| **X4** | Security floor | E1 establishes it; E4 adds outbound credentials; E5 adds the client allowlist | No epic ships an unauthenticated endpoint |

---

## 3. Epic E1 — Stage 1: contract-shaped read API

> **Outcome:** a peer service can `GET` our `Foo` catalogue in their own contract format, page
> by page, built on demand from current domain state.
>
> **Hard constraints:** no new table, no migration, no change to the write path, no worker. Verified
> by `git diff --stat` at the end.
>
> **Not in this epic:** domain events, outbox, change capture, push, deletions, inbound,
> `aggregateVersion`. See `STAGE_1_READ_API.md` §0 — the mapper is the *only* one of the three
> mechanisms people expect here.

**Actors**

| Actor | Cares about |
|---|---|
| **Consumer team** — the peer service's engineers | Getting the catalogue, in their shape, without surprises |
| **Peer contract owner** — owns the wire format | That we implement their schema faithfully and fail loudly when we cannot |
| **Data protection officer** | That personal data does not leave without a legal basis |
| **Acme on-call engineer** | Turning it off, seeing who is calling, not being paged by it |
| **Acme architect** | That the domain model is unchanged and stays unchanged |

**Composition:** 6 features (five plus discovery) · 10 user stories · 5 enablers · 41 tasks ·
**66 pts** — 56 in stories and enablers, 10 in the discovery feature, which has no story because
nothing in it is visible to any actor.

Roughly a third of the points sit in enablers. That is expected for a first increment that also lays
down the anti-corruption layer for every later stage, and it is worth stating rather than hiding
inside stories: the ratio is what tells a stakeholder why five days do not produce five days' worth
of visible features.

---

### F1.0 — Discovery (enabler feature)

*Establish what is actually in the codebase before writing code against assumptions.*

No stories: nothing here is visible to any actor. It is the **definition of ready for the entire
epic**, and four of its findings change the code that gets written.

| Task | Title | Guide | Pts | Depends |
|---|---|---|---|---|
| **T1.0-a** | Inventory every public member of `Foo`, recursively; record it as the `MAPPING_MATRIX.md` §2 field list | §1.2.1, C0 | 3 | — |
| **T1.0-b** | Determine EF mapping shape per value object: owned type versus single-column converter | §1.2.3 | 1 | — |
| **T1.0-c** | Check every collection property for a computed body — the filter-disappears trap | §1.2.4, §7.1 | 1 | — |
| **T1.0-d** | Check for global query filters (soft delete) | §1.2.5 | 1 | — |
| **T1.0-e** | Verify the keyset comparison translates to SQL — run it and **read the generated SQL** | §1.4.1, §7.3 | 1 | — |
| **T1.0-f** | Verify column types: `numeric` versus `double`, `timestamptz` versus `timestamp`, `date` for date-only | §1.4.4–1.4.6, §1.5.2 | 1 | — |
| **T1.0-g** | Confirm which personal-data fields the peer marks `required` — if any, Q6 becomes a blocker | §1.5.3 | 1 | — |
| **T1.0-h** | Request the peer schema revision and its freeze date (Q3); confirm the auth mechanism (Q2); get the legal basis for the peer's personal-data entitlement (Q6) | §1.3 | 1 | — |

**Feature done when** every row of guide §1 has a written answer and T1.0-e's SQL has been read by a
human.

---

### F1.1 — Catalogue read access

*A consumer can obtain every `Foo`, and any single `Foo`, over HTTP.*

**OpenSpec:** `foo-read-api` — "Foos are readable in the peer's contract shape",
"Reads project directly into a flat snapshot", "Pagination is keyset-based and capped".

#### US1.1.1 — Fetch one Foo

> **As** a consumer team, **I want** to fetch a single `Foo` by its id, **so that** I can
> reconcile or repair one record without walking the whole catalogue.

**Acceptance criteria**
- An existing id returns one item in the contract shape *(scenario: Single Foo read)*
- An unknown id returns `404` with no partial body *(scenario: Unknown id)*
- A soft-deleted `Foo` is not served
- Serving the request writes nothing *(scenario: No write side effects)*

**Pts:** 3 · **Depends:** EN1.1.1, EN1.1.2, EN1.1.3

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.1.1-a** | `GET /integration/v1/foos/{id}` endpoint | §4.8, C8 | 2 |
| **T1.1.1-b** | Soft-delete and not-found handling with a test each | §4.8, C8 | 1 |

#### US1.1.2 — Walk the whole catalogue

> **As** a consumer team, **I want** to page through the entire catalogue with a cursor, **so that**
> I can build my local copy and resume after a crash without duplicating or missing records.

**Acceptance criteria**
- A full walk returns every `Foo` exactly once, with no duplicate and no skipped pre-existing
  row, **including while rows are inserted and deleted mid-walk** *(scenario: Rows inserted mid-walk)*
- The cursor is opaque and versioned; a malformed cursor returns `400`, never a silent restart
- A page size above the maximum is capped, and the cap is documented *(scenario: Oversized page requested)*
- No `COUNT(*)` and no `OFFSET` appear in the generated SQL

**Pts:** 5 · **Depends:** EN1.1.3, EN1.1.4

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.1.2-a** | Keyset page query with `Take(limit + 1)` as the has-more probe | §4.3, C5 | 2 |
| **T1.1.2-b** | `GET /integration/v1/foos?cursor=&limit=` with the page cap | §4.8, C8 | 2 |
| **T1.1.2-c** | Testcontainers walk over 10 000 rows with concurrent writes | §6.3, C5 | 1 |

#### EN1.1.1 — Integration projects and reference direction

> Enabler. Three new projects, and `Integration → Application → Domain` plus
> `Integration → Infrastructure` per guide §2.2.

**Pts:** 2 · **Depends:** F1.0

| Task | Title | Guide | Pts |
|---|---|---|---|
| **TEN1.1.1-a** | Create the three projects and wire references | §2.1, C1 | 2 |

#### EN1.1.2 — Snapshot read model and the canonical projection

> Enabler. `FooSnapshot` plus `FooSnapshotProjection.ToSnapshot` as a **static
> expression**, because stage 3's materialiser reuses the same definition and a projection buried in
> a method body gets copied instead.

**Pts:** 5 · **Depends:** F1.0

| Task | Title | Guide | Pts |
|---|---|---|---|
| **TEN1.1.2-a** | `FooSnapshot` and its nested records | §4.2, C3 | 2 |
| **TEN1.1.2-b** | The projection expression, with collections ordered **in SQL** by a stable business key | §4.2, C3 | 3 |

#### EN1.1.3 — Snapshot store

> Enabler. `IFooSnapshotStore` with `AsNoTracking()` and `AsSplitQuery()` — the second one
> because two collection navigations in one projection otherwise produce a cartesian product.

**Pts:** 3 · **Depends:** EN1.1.2

| Task | Title | Guide | Pts |
|---|---|---|---|
| **TEN1.1.3-a** | Store interface and EF implementation, single read | §4.3, C4 | 2 |
| **TEN1.1.3-b** | `AsSplitQuery` and a query-count assertion | §7.2, C5 | 1 |

#### EN1.1.4 — Opaque page cursor

**Pts:** 2 · **Depends:** EN1.1.1

| Task | Title | Guide | Pts |
|---|---|---|---|
| **TEN1.1.4-a** | Versioned base64url cursor codec, failing closed on malformed input | §4.5, C6 | 2 |

---

### F1.2 — Contract fidelity (the anti-corruption layer)

*The payload is in the peer's vocabulary, and nothing is lost, guessed or truncated on the way
there.* **This feature is the substance of the epic** — everything else is delivery.

**OpenSpec:** `foo-read-api` — "The mapping is total, and its totality is enforced by the
build", "Values are translated without silent loss", "Payloads validate against the vendored peer
schema".

#### US1.2.1 — Receive data in our own schema

> **As** the peer contract owner, **I want** field names, types, units and enum values to match my
> published schema exactly, **so that** my service can consume the payload without a translation
> layer of its own.

**Acceptance criteria**
- Generated payloads validate against the vendored peer schema *(scenario: Vendored schema present)*
- Money carries an explicit currency and is never a floating-point number
- Areas carry an explicit unit; the conversion rounding rule is the documented one
- Date-only fields carry no time component *(scenario: Date-only field)*
- Collections are ordered deterministically by a stable business key
- Coordinates use the documented precision and stated reference system

**Pts:** 8 · **Depends:** EN1.2.1, T1.0-a

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.2.1-a** | Money, decimal and instant converters | §4.4, C12 | 2 |
| **T1.2.1-b** | Area conversion with an explicit unit, half-away-from-zero at 2 dp | §4.4, C12 | 2 |
| **T1.2.1-c** | Date-only, coordinate, phone E.164 and HTML normalisation converters | §4.4, C12 | 2 |
| **T1.2.1-d** | The Mapperly mapper wiring them together, `RMG012`/`RMG020` as build errors | §4.4, C14 | 2 |

#### US1.2.2 — Fail loudly rather than send a wrong value

> **As** the peer contract owner, **I want** a value that cannot be represented in my schema to fail
> visibly on your side, **so that** I never receive a truncated address or a guessed enum value that
> looks valid and is not.

**Acceptance criteria**
- A value over the peer's `maxLength` raises, naming the field, the actual length and the limit
  *(scenario: Value exceeds the peer's maximum length)*
- **No code path truncates**, asserted by test
- An unmapped enum value raises; no fallback value is substituted *(scenario: Unmapped enum value)*
- Adding a member to a mapped enum **breaks the build**

**Pts:** 3 · **Depends:** US1.2.1

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.2.2-a** | Fail-closed enum tables with exhaustive switches and throw arms | §4.4, C11 | 2 |
| **T1.2.2-b** | Length guards plus the no-truncation test | §4.4, C13 | 1 |

#### US1.2.3 — Be sure nothing was silently dropped

> **As** the peer contract owner, **I want** proof that every field in your domain is either mapped
> or deliberately excluded with a written reason, **so that** "we forgot that one" cannot be the
> explanation for missing data six months from now.

**Acceptance criteria**
- Adding a field to `Foo` fails a test that **names that field** *(scenario: Domain field left unaccounted)*
- Every exclusion has a non-empty reason and a `MAPPING_MATRIX.md` row *(scenario: Exclusion without a reason)*
- Round-trip property tests pass for symmetric fields; registered asymmetric mappings are asserted
  for convergence *(scenario: Round-trip over generated values)*
- `MAPPING_MATRIX.md` §2 is complete for `Foo` with **zero** unresolved `GAP-*` rows

**Pts:** 5 · **Depends:** US1.2.1, US1.2.2

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.2.3-a** | `MappingRegistry` and `IntegrationFieldExclusions` | §6.1, C17 | 1 |
| **T1.2.3-b** | Reflection field-coverage test and the reason assertion | §6.1, C17 | 2 |
| **T1.2.3-c** | Round-trip and boundary property tests | §6.2, C12 | 2 |

> Run **T1.2.3-b** before declaring US1.2.1 done: it will name fields the mapper missed, and that
> list is the last input to `MAPPING_MATRIX.md`. An hour here; a peer conversation in stage 3.

#### EN1.2.1 — Vendored schema and generated contract types

> Enabler. Vendor the peer's schema into the repository so an unannounced partner change arrives as a
> pull-request diff rather than a `422` at 3 a.m., and generate the DTOs so a removed field is a
> compile error. Includes deleting the hand-written stub.

**Pts:** 5 · **Depends:** Q3 · **Status:** ⛔ externally blocked

| Task | Title | Guide | Pts |
|---|---|---|---|
| **TEN1.2.1-a** | Hand-written stub contract so the pipe is end-to-end before the schema lands | §4.1, C7 | 1 |
| **TEN1.2.1-b** | Vendor the schema, generate DTOs, delete the stub | §4.1, C16 | 3 |
| **TEN1.2.1-c** | Schema validation in the test suite | §6.4, C16 | 1 |

---

### F1.3 — Data protection

*Personal data does not leave without a legal basis.*

**OpenSpec:** `foo-read-api` — "Personal data is served only under a recorded grant".

#### US1.3.1 — Personal data leaves only under a recorded grant

> **As** the data protection officer, **I want** every personal field that leaves the service to be
> covered by a written grant naming the legal basis and the recipient's retention, **so that** the
> transfer is defensible rather than merely permitted by whoever configured it.

**Acceptance criteria**
- A field marked `pii` appears **only** where the recorded grant covers it *(scenario: Grant covers the personal data)*
- With no grant recorded, every `pii` field is omitted — a refusal, not a default *(scenario: No grant recorded)*
- The grant records **who authorised it, the legal basis, and the permitted retention**, not just "allowed"
- The personal-data registry and the mapping matrix agree, asserted by test
- Adding a new personal field fails the coverage test until it carries a grant decision *(scenario: New personal field added)*

> **Q6 is answered in two parts and both matter.** The peer *is* entitled, **and** it marks some
> personal fields `required` — so blanket redaction would produce payloads that fail its own
> validation. The story is therefore about the grant, not about stripping. **Still blocking:** the
> legal basis and the peer's retention are not yet written down, and this story cannot be accepted
> without them.

**Pts:** 3 · **Depends:** T1.0-g, US1.2.1

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.3.1-a** | `PiiFields` registry as the single source of truth, reused by stage 4's variant | §4.6, C15 | 1 |
| **T1.3.1-b** | `ContractRedactor` and the response-level redaction test | §4.6, §6.4, C15 | 2 |

---

### F1.4 — Access control and operability

*Only authorised callers, at a bounded rate, and we can turn it off.*

**OpenSpec:** `foo-read-api` — "The endpoints ship authenticated, authorised and bounded",
"The endpoints are reversible without a deploy".

#### US1.4.1 — Authenticated, scoped, bounded access

> **As** the on-call engineer, **I want** every integration request authenticated, scope-checked and
> rate-limited, **so that** a misconfigured network policy is not the only thing standing between our
> catalogue and anything that can reach the pod.

**Acceptance criteria**
- Unauthenticated ⇒ `401`, no query executed *(scenario: Unauthenticated request)*
- Missing scope ⇒ `403` *(scenario: Wrong scope)*
- Over the per-caller rate ⇒ `429`, other callers unaffected *(scenario: Caller exceeds its rate)*
- The mechanism sits behind a seam, changeable without touching the endpoints *(scenario: Mechanism changes)*
- No credential and no payload appears in any log line *(scenario: Credentials never logged)*
- TLS only; no certificate-validation callback anywhere, in any environment

**Pts:** 5 · **Depends:** EN1.1.1, Q2

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.4.1-a** | `IIntegrationAuthenticator` seam — **not** an OAuth-specific claim lookup, because stage 5 resolves `source_system` through it (defect S1) | §4.7, C9 | 2 |
| **T1.4.1-b** | Scope policy and per-caller rate limit | §4.7, C9 | 1 |
| **T1.4.1-c** | Log redaction policy and the token-never-logged test | §4.7, §6.4, C10 | 2 |

#### US1.4.2 — Turn it off without a deploy

> **As** the on-call engineer, **I want** to disable the integration endpoints at runtime, **so that**
> an incident is a configuration change and not a release.

**Acceptance criteria**
- Flag off ⇒ both routes `404`, no query executed *(scenario: Flag disabled)*
- Default is off in production
- Toggling affects no other route *(scenario: Single subscriber isolated — stage 1 analogue)*

**Pts:** 2 · **Depends:** US1.1.1

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.4.2-a** | Feature-flag endpoint filter returning `404`, not `403` | §4.10, C8 | 1 |
| **T1.4.2-b** | Request count and duration metrics by route and caller | §8, C18 | 1 |

---

### F1.5 — Boundary integrity and the consumer contract

*The domain is unchanged and stays unchanged; the consumer knows exactly what they are getting.*

**OpenSpec:** `foo-read-api` — "The domain stays unaware of the integration", "Stage 1's
guarantees and non-guarantees are written down for the consumer".

#### US1.5.1 — The domain stays clean

> **As** the Acme architect, **I want** an automated assertion that the domain and application
> layers reference nothing new, **so that** the constraint survives the next twelve pull requests
> rather than the next one.

**Acceptance criteria**
- No domain or application type references the integration assemblies or `System.Net.Http`
  *(scenario: Architecture tests run)*
- No external contract type appears in a domain signature, an application command, an EF entity, or
  any response outside the integration routes
- `git diff --stat` for the epic shows **no migration** and **no change under `Domain/`**

**Pts:** 2 · **Depends:** EN1.1.1

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.5.1-a** | Architecture tests, written on day 1 while trivially green | §6.5, C2 | 2 |

#### US1.5.2 — Know what stage 1 does and does not guarantee

> **As** a consumer team, **I want** the guarantees and the non-guarantees written down, **so that** I
> do not build an incremental sync on an endpoint that cannot support one, and do not discover the
> deletion gap in production.

**Acceptance criteria**
- `CONSUMER_CONTRACT.md` exists and states: **replace-all semantics only**, no incremental sync, **no
  deletion signal**, the page contract, the page cap, the rate limit, and that stage 1 is time-boxed
  *(scenarios: Record deleted on our side · Consumer asks for incremental sync)*
- It says how `aggregateVersion` arrives in stage 2 and that the addition is additive
  *(scenario: Version field added in stage 2)*
- The pilot consumer has read it and reconstructed the full catalogue

**Pts:** 3 · **Depends:** US1.1.2, US1.2.3

| Task | Title | Guide | Pts |
|---|---|---|---|
| **T1.5.2-a** | Write `CONSUMER_CONTRACT.md` and add its README row (defect **D13**, item **X1**) | §9, C19 | 2 |
| **T1.5.2-b** | Walk the pilot consumer through a full catalogue read | §8, group E | 1 |

---

### E1 sequencing

Commit order, dependencies and the blocked/unblocked split are in `STAGE_1_READ_API.md` §8. In
backlog terms:

| Slice | Items | Ends with |
|---|---|---|
| 1 | F1.0, EN1.1.1, US1.5.1, EN1.1.2 | The projection asserted **in memory** against a built aggregate; architecture tests green. No database, no mapper, no endpoint yet — guide §6.0 |
| 2 | EN1.1.3, EN1.1.4, TEN1.2.1-a, US1.1.1, US1.1.2 | The same assertions re-run **through PostgreSQL** (the pair that detects the translation traps), then **a peer-shaped page over HTTP** — the demo |
| 3 | US1.4.1, US1.4.2, US1.3.1 | Authenticated, bounded, redacted, reversible |
| 4 | US1.2.2, US1.2.1, US1.2.3 | The ACL, proven total |
| 5 | EN1.2.1 (b, c), US1.5.2 | Real contract types, contract handover ⛔ needs Q3 |

Slices 1–4 are unblocked. Slice 5 is the exit criterion and depends on another team.

---

## 4. Epics E2–E7 — features and stories

Deliberately not decomposed to tasks. Each epic's tasks get written when its predecessor closes,
because that is when its defect gates are resolved and its unknowns are known.

### E2 — Change capture (stage 2) · gates D1, D6

| Feature | Stories |
|---|---|
| **F2.1** Capture integrity | **US2.1.1** As the on-call engineer, I want it to be impossible to commit a business change without its integration record, so that "the update never reached them" stops being a possible incident. *AC: rollback leaves no capture row; commit leaves exactly one; five edits in one transaction leave one* |
| **F2.2** Version for idempotency | **US2.2.1** As a consumer team, I want a monotonic version per `Foo`, so that I can ignore an update older than what I already hold. *AC: appears in stage 1 responses additively; monotonic under concurrent writes; defined for every `Foo` including never-changed ones* |
| **F2.3** Capture completeness | **US2.3.1** As the architect, I want a new entity type to fail startup rather than silently escape capture, so that "we forgot to publish changes to the new table" cannot happen |
| **F2.4** Provenance | **EN2.4.1** Enabler: record causing source and propagation path on every capture row — stage 5's loop prevention has no field to read without it (defect **D6**) |

### E3 — Incremental feed (stage 3) · gates D2, D5, D9, D11, D14

| Feature | Stories |
|---|---|
| **F3.1** Incremental synchronisation | **US3.1.1** As a consumer team, I want to fetch only what changed since my cursor, so that I stop re-reading a 500 000-row catalogue to find twelve changes |
| **F3.2** Gap-free cursor | **US3.2.1** As a consumer team, I want a guarantee that the feed cannot skip a record, so that my copy is not quietly missing rows. *AC: the watermark test — slow transaction N, committed N+1, feed returns neither then both — is a release gate* |
| **F3.3** Deletion visibility | **US3.3.1** As a consumer team, I want deletions and retractions as explicit tombstones, so that I do not keep records you withdrew. *Needs the peer's deletion representation — defect **D11**, Q10* |
| **F3.4** Expired-cursor honesty | **US3.4.1** As a consumer team, I want an explicit, actionable error when my cursor is older than your retention, so that I do not receive a silently incomplete page |
| **F3.5** Materialisation robustness | **US3.5.1** As the on-call engineer, I want one bad record to be quarantined instead of stalling the pipeline, so that a single data-quality problem is not an outage. *Defects **D9**, **D14*** |
| **F3.6** Pipeline observability | **US3.6.1** As the on-call engineer, I want one number that says whether the pipeline is healthy, so that triage starts from a fact |

### E4 — Push delivery (stage 4) · gates D3, D4, D7, D12, S5

| Feature | Stories |
|---|---|
| **F4.1** Low-latency delivery | **US4.1.1** As a consumer team, I want changes pushed to me within seconds, so that I do not have to build and operate a poller |
| **F4.2** Live traffic priority | **US4.2.1** As a consumer team, I want a multi-day backfill never to delay a live price change, so that bootstrap and steady state do not compete. *Defect **D3** — the promise the current claim SQL breaks* |
| **F4.3** Failure isolation | **US4.3.1** As the on-call engineer, I want a permanently failing message quarantined on its first attempt and a transient one retried across restarts, so that one malformed record cannot block a queue for days |
| **F4.4** Subscriber onboarding as data | **US4.4.1** As the on-call engineer, I want to onboard a consumer with a row and an identity-provider client, so that a new consumer is not a deployment |
| **F4.5** Per-subscriber entitlement | **US4.5.1** As the data protection officer, I want personal data emitted only to subscribers holding an explicit grant, with the redacted variant for everyone else. *Defect **D7*** |
| **F4.6** Operator control | **US4.6.1** As the on-call engineer, I want to pause a subscriber, inspect a dead letter and replay it, all audit-logged, so that recovery does not need a database session |

### E5 — Inbound (stage 5) · gates D6 (full), S1

| Feature | Stories |
|---|---|
| **F5.1** Accept peer data | **US5.1.1** As a peer service, I want my message accepted durably and deduplicated on my own identifier, so that retrying is safe and a bug on your side is not reported to me as a delivery failure |
| **F5.2** Domain-safe application | **US5.2.1** As the architect, I want inbound data to pass every domain invariant by going through the existing command handlers, so that external input is not a privileged write path |
| **F5.3** Loop prevention | **US5.3.1** As the on-call engineer, I want a change absorbed from a peer never to bounce back to it — including across three systems, where suppressing the immediate sender is not enough. *Defect **D6*** |
| **F5.4** Identity correlation | **US5.4.1** As the architect, I want peer identifiers correlated alongside ours, never substituted for our key |

### E6 — Backfill and reconciliation (stage 6) · gate D8

| Feature | Stories |
|---|---|
| **F6.1** Bootstrap without downtime | **US6.1.1** As a consumer team, I want to load the full catalogue and switch to the incremental feed with no freeze window, so that go-live is not a coordinated outage |
| **F6.2** Divergence detection | **US6.2.1** As the on-call engineer, I want daily proof that the consumer's copy matches ours, cheap enough to run every night, so that divergence is found by a job and not by a business user. *Must be subscriber-scoped, or a filtered consumer diverges on every bucket every night — defect **D8*** |
| **F6.3** Targeted repair | **US6.3.1** As the on-call engineer, I want to re-emit a named set of records for one subscriber, so that repair does not mean a full re-export |

### E7 — Hardening and scale (stage 7) · gate D10

| Feature | Stories |
|---|---|
| **F7.1** Bounded growth | **US7.1.1** As the on-call engineer, I want each integration table pruned by a mechanism its schema actually supports, so that retention is not a plan that fails on contact. *Defect **D10*** |
| **F7.2** Capture-gap prevention | **US7.2.1** As the architect, I want bulk SQL against integrated types to fail the build unless explicitly annotated, so that the sharpest edge in the capture design is guarded |
| **F7.3** Operational readiness | **US7.3.1** As the on-call engineer, I want a runbook and a game day, so that the first real incident is not the first time anyone has seen these alerts |
| **F7.4** Scale levers | **US7.4.1** As the on-call engineer, I want the documented order in which capacity is added, so that the response to a backlog is configuration before architecture |

---

## 5. Traceability

### 5.1 Epic → decision → requirements

| Epic | Governing ADRs | OpenSpec capability |
|---|---|---|
| E1 | 0004, 0005, 0006 §2, 0007 | `foo-read-api` |
| E2 | 0002, 0007 | `change-capture` |
| E3 | 0002 §2, 0003 §1/§5, 0004 | `outbox-materialization`, `change-feed` |
| E4 | 0003 §2–§4/§6, 0005 | `outbox-delivery`, `integration-security` |
| E5 | 0001 §2, 0003 §7 | `inbound-ingestion` |
| E6 | 0006 | `backfill-reconciliation` |
| E7 | 0003 §8 | `integration-operations` |

### 5.2 E1 story → requirement

| Story | OpenSpec requirement in `foo-read-api` |
|---|---|
| US1.1.1 | Foos are readable in the peer's contract shape · Reads project directly into a flat snapshot |
| US1.1.2 | Pagination is keyset-based and capped |
| US1.2.1 | Values are translated without silent loss · Payloads validate against the vendored peer schema |
| US1.2.2 | Values are translated without silent loss |
| US1.2.3 | The mapping is total, and its totality is enforced by the build |
| US1.3.1 | Personal data is served only under a recorded grant |
| US1.4.1 | The endpoints ship authenticated, authorised and bounded |
| US1.4.2 | The endpoints are reversible without a deploy |
| US1.5.1 | The domain stays unaware of the integration |
| US1.5.2 | Stage 1's guarantees and non-guarantees are written down for the consumer |

Eleven requirements, ten stories: the eleventh — "Reads project directly into a flat snapshot" — is
shared, which is why EN1.1.2 exists as an enabler rather than being folded into one story.

---

## 6. Open questions as blockers

Authoritative list: [TECHNICAL_PROPOSAL.md §2.3](TECHNICAL_PROPOSAL.md). This table maps it onto
backlog items.

| Q | Question | Blocks | Default if unanswered |
|---|---|---|---|
| **Q2** | Which auth mechanism? Workload identity, an IdP, or a gateway — see ADR-0005 §1 | **US1.4.1** | OAuth 2.0 client credentials, HMAC fallback |
| **Q3** | Which revision of the peer's schema, and when is it frozen? Their project is still in development | **EN1.2.1**, US1.5.2, and E1's exit | Build against a stub; the epic cannot exit until a revision is named |
| **Q6** | The peer is entitled and marks some personal fields `required`. On what legal basis, with what retention on their side? | **US1.3.1**, which now asserts *presence under a recorded grant* rather than absence | **No safe default.** Record the basis before E1 ships |
| **Q1** | Snapshot or delta payloads? | E4 ordering machinery | Snapshots |
| **Q4** | Embed `Bar`/`Baz`, or references only? | E3 fan-out | References only |
| **Q7** | Does the peer push, do we poll, or both? | E5 shape | Ingress first, poller second |
| **Q8** | Catalogue size? | E6 sizing | ~500 k `Foo` records |
| **Q9** | `aggregateVersion` source: per-aggregate counter or emission sequence? | E2 | Per-aggregate counter (defects **D1**, **D2**) |
| **Q10** | How does the peer represent a deletion? | **US3.3.1** | A separate deletion message type (defect **D11**) |
| **Q11** | Propagation path in the envelope, or is direct-source suppression enough at two systems? | **US5.3.1** | Direct source plus content hash (defect **D6**) |
| **Q14** | Does the consumer need transition messages? | E4 stream locking (finding **S2**) | No — defer entirely |
| **Q16** | Who owns `CONSUMER_CONTRACT.md` and signs it off? | E2 onward | Blocks E2 (defect **D13**) |

**Only Q2, Q3 and Q6 touch Epic 1.** Everything else can be answered while E1 is being built.

### 6.1 Answered, and what they removed

| Q | Answer | Backlog effect |
|---|---|---|
| **Q5** | PostgreSQL **17** | Risk R4 closed; E3 has no version prerequisite left |
| **Q12** | One consumer, entitled to the personal data | **D7 deferred.** US4.5.1 shrinks to "one payload, one grant" — but the variant dimension stays in the schema |
| **Q13** | Checksum scope is moot at one subscriber | **D8 deferred.** E6 drops from **M** to **S** |
| **Q15** | No subscriber filters in phase 1 | **D12 deferred.** US4.5.1 loses the projection-state work; a per-aggregate last-hash row is still wanted for D6 |
| **Q17** | Exactly one consumer, one-to-one | E1 has a real reader. US5.3.1's acceptance drops the three-system loop and asserts against the peer directly |

Deferred is not resolved: each returns the day a second consumer appears, which is why the
subscriber table still ships in E4 holding one row.

---

## 7. Importing this into a tracker

| This document | Jira | Azure DevOps |
|---|---|---|
| Epic `E1` | Epic | Epic |
| Feature `F1.2` | — (use a label or a component) | Feature |
| Story `US1.2.1` | Story | User Story |
| Enabler `EN1.2.1` | Story, label `enabler` | User Story, tag `enabler` |
| Task `T1.2.1-a` | Sub-task | Task |
| Pts | Story Points | Effort |
| Guide reference | link to `STAGE_1_READ_API.md#…` | same |
| Defect gate `D3` | label `gate:D3`, blocked-by link | tag `gate-D3` |
| Open question `Q3` | label `blocked:Q3` | tag `blocked-Q3` |

Two conventions worth enforcing on import:

- **Carry the acceptance criteria verbatim.** They reference OpenSpec scenarios by name, and the
  scenario is what the test asserts. Paraphrasing them in the tracker breaks that chain.
- **Keep the ids.** A tracker key is not a substitute: commit messages, review comments and the
  documents in this repository all reference `US1.2.1`, and that reference must keep resolving.

Jira has no native Feature level between Epic and Story; use a component or a label rather than
inventing an extra Epic tier, which would put the stage boundary and the capability boundary at the
same level and lose the distinction ADR-0007 depends on.
