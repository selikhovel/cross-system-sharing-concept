# ADR-0010 — Global identifier issuance and custody

- **Status:** Proposed — **the highest-priority open topic in the design**
- **Date:** 2026-07-25
- **Related:** ADR-0001 (no synchronous calls on the write path), ADR-0002 (change capture),
  ADR-0006 (backfill), ADR-0008 (merge), ADR-0009 (regions)
- **Open questions:** Q20 (idempotent creation — **blocking**), Q21 (identifier supersession),
  Q22 (user-visible identifier)

> **Why this is called out separately.** Every other decision in this repository can be revised
> later at the cost of a release. This one cannot. An identifier that has been issued, attached
> and observed by another system is permanent, and the failure modes here — a duplicate entity in
> the global aggregator, or two identifiers for one record — are silent on our side and expensive
> to unwind on theirs. **Q20 must be answered before stage 5 is designed in detail, and preferably
> before stage 2 ships.**

## Context

The aggregator issues a **globally unique identifier (GUID) per entity**. It is issued once,
attached permanently, and is the correlation key across the federation. Two origins exist:

- **Created in the aggregator** — the entity reaches us already carrying its identifier. Nothing
  to decide; it is recorded on ingest.
- **Created in `Acme`** — the entity exists locally first and must acquire an identifier from the
  aggregator.

The second case is the whole of this ADR, and it collides with a rule established in ADR-0001:
**no synchronous call to an external service on the write path.** Obtaining an identifier is a
network call. If creating a `Foo` requires one, then creating a `Foo` fails whenever the
aggregator is unavailable, and a commit-then-call sequence reintroduces the dual write the outbox
exists to eliminate.

So the entity necessarily exists, locally, before it has a global identifier. The design has to
make that window explicit and safe rather than pretend it away.

## Decision

### 1. Custody: the integration layer, not the domain

The identifier lives in `integration.external_reference` — the table already specified for
correlating foreign identities:

```sql
-- target shape (extends TECHNICAL_PROPOSAL §4.6)
CREATE TABLE integration.external_reference (
    aggregate_type text        NOT NULL,
    aggregate_id   uuid        NOT NULL,
    system         text        NOT NULL,
    external_id    text        NOT NULL,
    region_code    text        NOT NULL,
    status         text        NOT NULL DEFAULT 'Active'
                   CHECK (status IN ('Active','Superseded')),
    superseded_by  text,
    issued_at      timestamptz,
    issued_by      text,                    -- 'delivery-response' | 'inbound' | 'pool'
    PRIMARY KEY (aggregate_type, aggregate_id, system)
);
CREATE UNIQUE INDEX ux_external_reference_reverse
    ON integration.external_reference (system, external_id);
```

The two constraints are the point: one entity has at most one identifier from a given system, and
one identifier belongs to at most one entity.

**It is not a domain field.** An identifier belongs in the domain only if the domain *reasons*
with it — an invariant depends on it, another bounded context keys on it, or regulation requires
it on the record. Being displayed does not meet that test. Four reasons against:

- It is a foreign identity issued by a foreign system for federation, not a business concept. Our
  aggregate already has an identity.
- Under §2 it does not exist for a window after creation. A domain field that is null for an
  arbitrary period and then becomes immutable is an invariant the domain did not ask for.
- If identifiers can be superseded (§7), a domain field would have to be rewritten from outside —
  mutating an aggregate's identity externally.
- Disable the integration and the domain keeps a meaningless nullable column.

**Display is served by the read side.** A view over `external_reference`, joined into query DTOs:

```sql
CREATE VIEW integration.registration AS
SELECT aggregate_type, aggregate_id, external_id AS global_id, status, issued_at
  FROM integration.external_reference
 WHERE system = 'peer-a';
```

Read models join the *view*, never the table, so the coupling stays in one named place. The reverse
lookup is on the primary key, so the join is cheap. Exports consumed by another system are built
from the **contract mapper**, not from a separate query — a second hand-written mapping of the same
fields drifts from the first, and the divergence is discovered by whoever consumes the export.

### 2. Issuance rides on the first outbound delivery

The identifier is **not** requested through a dedicated call. It arrives in the response to the
first successful delivery of the entity:

```
Acme  ──POST create, sourceRef = {system: "acme", id: <local guid>}──►  aggregator
Acme  ◄──201 { globalId: "…" }──────────────────────────────────────────
      write external_reference + acknowledge the delivery — one transaction
```

Subsequent messages for that entity carry both identifiers. This is ordinary REST creation
semantics, so it is likely already how the aggregator's API behaves.

What this removes, compared with maintaining a pre-issued reserve: the reserve table, the refill
worker, the low-watermark alarm, the exhaustion policy, a dedicated issuance endpoint, and a
separate bulk-issuance run for the existing catalogue. Existing entities acquire identifiers as
backfill delivers them (ADR-0006), which is a substantial saving at catalogue scale.

### 3. The condition this rests on — Q20

We send a creation, the aggregator creates the entity and issues an identifier, and **the response
is lost to a timeout**. The outbox retries, because that is what it is for.

- If the aggregator deduplicates creation by our `sourceRef`, it returns the same identifier.
  Correct, and free.
- If it does not, it creates a **second entity with a second identifier**. A duplicate in the
  global aggregator, invisible from our side, because from our side everything succeeded.

> **Q20 — does the aggregator deduplicate creation by our `sourceRef` / `Idempotency-Key`?**

This is the same requirement either mechanism needs, but the cost of its absence differs sharply.
A pre-issued reserve puts the requirement on a low-traffic issuance endpoint, where a lost response
is harmless. Riding along puts it on **the path every creation takes**, where a lost response
creates a duplicate. If the answer is no, §8's fallback applies.

### 4. Recording the identifier is atomic with acknowledging delivery

Write `external_reference` and mark the delivery `Delivered` in **one transaction**. The ordering
is not symmetric:

- Acknowledge first, crash before recording → the identifier is lost permanently, because we will
  never ask again.
- Record first, crash before acknowledging → the retry returns the same identifier (given Q20) and
  rewrites the same row. Harmless.

This changes the dispatcher's contract: it no longer only classifies a status code, it parses a
response body and writes durable state from it.

### 5. Registration state, and `Defer` rather than `Skip`

```
Unregistered  →  (first successful delivery)  →  Registered  →  (aggregator merge)  →  Superseded
```

A message keyed on the global identifier cannot be emitted while the entity is `Unregistered`.
The publication policy therefore needs a fourth outcome — **`Defer`** (retry later) alongside
`Publish | Tombstone | Skip`. `Skip` would silently drop the entity and it would never be
published at all.

The presence of a row in `external_reference` doubles as proof of registration: *the aggregator
knows about this entity*. That fact is reused in §6 and in reconciliation.

### 6. Two consequences of the window

**References must be published in dependency order.** If `Foo` references `Bar` by global
identifier and `Bar` is still `Unregistered`, `Foo` cannot be emitted. Either the materialiser
publishes parents before children, or the contract accepts `sourceRef` in reference positions
until the referenced entity registers. The second is simpler but needs the aggregator's agreement.
A pre-issued reserve does not have this problem at all, which is its main advantage.

**An entity created and deleted before its first sync should produce nothing.** The aggregator
never knew it, so there is no tombstone to send. The absence of an `external_reference` row is
exactly the test: never registered plus now deleted ⇒ publish nothing.

### 7. Invariants, enforced in the database

1. **Immutable once assigned.** Never rewritten, never reassigned.
2. **Never reused.** An identifier freed by a deletion does not return to circulation. Reusing a
   GUID another system may have observed is worse than losing one; GUIDs are not scarce.
3. **Excluded from the merge.** The global identifier never participates in per-field
   last-writer-wins (ADR-0008). It is identity, not a value: aggregator-owned, immutable, never
   merged. `MAPPING_MATRIX.md` records it as such.

> **Q21 — can the aggregator merge or retire identifiers?** Aggregators routinely deduplicate,
> which retires one identifier in favour of another. If it does, `status` / `superseded_by` above
> are load-bearing and a reference-rewriting path is required, along with a decision about what
> happens to the local entity whose identity was absorbed. If it does not, obtain that in writing:
> retrofitting supersession after identifiers are in circulation is expensive.

### 8. Fallback if Q20 is answered "no": a pre-issued reserve

Request blocks of identifiers ahead of time and hold them locally:

```sql
CREATE TABLE integration.global_id_pool (
    global_id   uuid PRIMARY KEY,
    region_code text NOT NULL,
    batch_id    uuid NOT NULL,
    received_at timestamptz NOT NULL DEFAULT now(),
    claimed_at  timestamptz,
    claimed_for text
);
CREATE INDEX ix_pool_free ON integration.global_id_pool (region_code, global_id)
    WHERE claimed_at IS NULL;
```

Claiming is an `UPDATE` in our own database, inside the creating transaction — no network call, the
identifier is available immediately, and a rolled-back transaction returns it to the reserve rather
than burning it. The reserve is refilled by a worker on a low-watermark trigger, with an alarm on
exhaustion and a documented degradation path.

Assignment needs no change to command handlers: the capture interceptor of ADR-0002 already sees
`Added` aggregates in the right transaction, and can claim and record there.

The reserve is also the answer if **Q22** — is the identifier user-visible immediately after
creation? — turns out to require it. Under §2 the identifier appears only after the first
successful sync; if an operator must read it aloud the moment the record is saved, that is a UX
requirement the reserve satisfies and ride-along does not.

## Alternatives considered

**Synchronous issuance at creation.** One call, identifier available immediately, no window.
*Rejected*: it makes creating a `Foo` depend on the aggregator being up, and commit-then-call is
the dual write ADR-0001 exists to prevent.

**A dedicated asynchronous issuance call.** A worker requests an identifier for each unregistered
entity, separately from publication. *Rejected*: it needs an extra endpoint and an extra round trip
to reach exactly the state that §2 reaches as a side effect of a message we are already sending,
and it has the identical Q20 dependency.

**A pre-issued reserve as the primary mechanism.** Strictly better on three points — no window,
no dependency ordering, immediate visibility. *Not chosen as primary* because it costs a subsystem
(reserve, refill, exhaustion policy, bulk issuance for the existing catalogue) to buy properties we
do not currently need. Retained as the fallback in §8, where it earns its cost either by answering
Q20 negatively or Q22 positively.

**`Acme` generates the GUID and the aggregator accepts it.** Registration rather than issuance.
This is by far the cheapest option — no reserve, no window, no dependency ordering, no Q20 —
and it is worth asking for explicitly before accepting either mechanism above. *Not assumed*: the
requirement as stated is that the aggregator issues the identifier, and a global aggregator usually
issues its own keys precisely so it can guarantee uniqueness across sources it does not trust.

**Store the identifier on the domain aggregate.** Removes a join and makes it trivially available
everywhere. *Rejected* for the four reasons in §1. The intermediate form — a column on the domain
table written by the integration layer without a corresponding property on the aggregate — is worse
than either end: it creates the coupling without the constraints, `status` and `superseded_by` have
nowhere to live, and code will read it directly.

## Consequences

### Positive

- No network call on the write path; creating an entity never depends on the aggregator.
- No reserve subsystem, and no separate mass-issuance run for the existing catalogue — backfill
  covers it.
- The domain is untouched, and identifiers reach the UI through the read side, where display
  concerns belong.
- `external_reference` doubles as a registration ledger, which the publication gate, the
  tombstone rule and reconciliation all reuse.
- Both origins converge on one representation: issued to us, or received with the entity.

### Negative / costs accepted

- **Correctness depends on Q20, which is not ours to answer.** Without idempotent creation, a lost
  response produces a duplicate entity in the global aggregator — the most expensive failure in
  this design, and one with no local symptom.
- **A window exists in which the entity has no global identifier**, and both dependency ordering
  and the `Defer` outcome exist only to survive it.
- **Publication must respect reference dependencies**, which the materialiser did not previously
  have to consider.
- **The dispatcher gains responsibility for durable state**, widening the failure surface of a
  component that was previously send-and-classify.
- **Supersession may be required later** (Q21) and is expensive to retrofit once identifiers are in
  circulation.
- **Immediate visibility is not available.** If Q22 requires it, the fallback subsystem in §8 has to
  be built after all.
