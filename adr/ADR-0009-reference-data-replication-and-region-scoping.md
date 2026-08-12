# ADR-0009 — Reference data replication and region scoping

- **Status:** Proposed
- **Date:** 2026-07-25
- **Related:** ADR-0004 (enum mapping), ADR-0006 (backfill, reconciliation), ADR-0008 (merge)
- **Open question:** Q19 — the aggregator's reference-data API shape and its regional model
- **Amends:** ADR-0004 §6 (the fail-closed enum rule now applies only to enums we own)

## Context

Two facts arrived after the existing documents were written, and together they add a third class
of data the design does not currently account for.

**The aggregator owns the vocabularies.** Picklists, enumerated values and the metadata around
them belong to the target aggregator (`peer-a`), not to `Acme`. We do not create them, we do not
publish them, and we cannot decline a value it adds.

**The vocabularies are regional.** Each region has its own picklists and its own metadata, and
`Acme` holds only the data for the region it serves — today one region, several later. Our
catalogue is a **partial replica** of a global one.

Entity data and reference data have opposite properties, and treating them as one mechanism
produces a system that is wrong for both:

| | Entities (`Foo`, `Bar`, `Baz`) | Reference data (picklists, metadata) |
|---|---|---|
| Ownership | shared, every field (ADR-0008) | aggregator, wholly |
| Direction | bidirectional | **inbound only** |
| Conflicts | per-field LWW over a three-way merge | none — the target is right by definition |
| Unit of versioning | field | **the whole set** |
| Scope | region | region |

Reference data needs no outbox, no shadow, no clock and no merge. It needs a replicated cache
with a set version. That is a much cheaper mechanism — but it is a *different* one.

There is also a rule in the existing design that this makes actively harmful. ADR-0004 §6
requires enums to be mapped by a compile-time table with an exhaustive `switch`, failing closed
on an unmapped value. For enums `Acme` owns, that is right. For an aggregator picklist it means:
the aggregator adds a value for a region on Tuesday, our compiled table does not know it, and
**every entity carrying that value dead-letters until we ship a release**. Waiting on our
deployment cadence to absorb someone else's reference row is not acceptable.

## Decision

### 1. A separate, one-way replication pipeline

Reference data is pulled from the aggregator into a versioned local replica. It never touches the
outbox, because we never publish it.

```sql
CREATE TABLE integration.reference_set (
    region_code text        NOT NULL,
    set_name    text        NOT NULL,      -- 'FooKind'
    version     text        NOT NULL,      -- ETag or version token from the aggregator
    synced_at   timestamptz NOT NULL,
    PRIMARY KEY (region_code, set_name)
);

CREATE TABLE integration.reference_value (
    region_code text    NOT NULL,
    set_name    text    NOT NULL,
    code        text    NOT NULL,
    label       jsonb,                     -- localized labels
    sort_order  int,
    is_active   boolean NOT NULL DEFAULT true,
    attributes  jsonb,                     -- regional particulars of the value
    PRIMARY KEY (region_code, set_name, code)
);
```

A set is applied **atomically, in one transaction**. Applying value by value leaves a window in
which the replica is internally inconsistent, and any entity validated in that window gets an
answer that is neither the old vocabulary nor the new one.

### 2. Project into the domain's own reference table

The domain's foreign keys must not point into the `integration` schema — that would couple the
domain's schema to the integration's:

```
aggregator  →  integration.reference_value      (versioned replica)
                        │  atomic projection on set apply
                        ▼
               <domain>.foo_kind                (the domain's existing table, its FK target)
```

The domain keeps referencing its own table and does not know something external fills it. The
integration layer is the only writer. A `source` column on the domain table distinguishes
aggregator-sourced rows from historical local ones — without it the cutover is irreversible.

### 3. Deprecate, never delete

A value withdrawn by the aggregator becomes `is_active = false`. It is not removed.

Existing records reference it. Hard-deleting turns correct historical data into invalid data
retroactively, which is a far worse outcome than an inactive row. Inactive values cannot be
chosen for new records and remain readable on old ones.

### 4. An unknown code is a stale replica, not bad data

This is the amendment to ADR-0004 §6. The enum strategy splits in two:

| Class | Mechanism | Unmapped value |
|---|---|---|
| Enums `Acme` owns (`FooStatus` workflow states) | compile-time table, exhaustive `switch` | **fail closed** — build error or dead letter, as before |
| Aggregator picklists (`FooKind`, tag vocabularies) | runtime lookup against the replica | **refresh the set and retry once**, then permanent |

The error class inverts for picklist-backed fields: from permanent to transient-then-permanent.
A cooldown on the refresh is required, or a batch of messages carrying a genuinely unknown code
turns into a request storm against the aggregator.

### 5. Ordering is a hard dependency

An entity referencing a code absent from the replica cannot be applied. Bootstrap order is
therefore fixed:

```
regions  →  reference sets for the region  →  entities
```

A startup gate enforces it at runtime: inbound entity processing does not begin until every set
has synced at least once. `reference_set_age_seconds` is the SLI — a stale replica surfaces as a
rise in entity-level errors, and without this metric the cause is looked for in the wrong layer.

### 6. Region as a data dimension, not a deployment assumption

`region_code` is added **now** to every integration table — `outbox_message`, `outbox_delivery`,
`peer_shadow`, `field_state`, `reference_set`, `reference_value`, `subscriber` — populated from
instance configuration. Region also appears in the API paths from the first endpoint:

```
GET /integration/v1/{region}/foos/changes
```

The resolution logic stays single-region: one region from configuration, no per-record region
resolution. Only the shape is prepared.

The asymmetry is the reason. Adding the column now costs nothing. Adding it after months of data
costs a migration plus a hunt for every place region was implicitly assumed. Adding a path
segment later is a breaking change for every consumer.

### 7. A partial replica changes two mechanisms that already exist

**Reconciliation must be region-scoped.** ADR-0006 §4 compares our catalogue against the peer's.
We hold one region of a global catalogue, so an unscoped comparison reports the rest of the world
as divergence. This is a different axis from the subscriber scoping settled by Q13: that question
concerned multiple subscribers, of which there is one; this concerns our data being a subset of
the aggregator's, which is true with a single subscriber.

**`OutOfScope` is not `Deleted`.** When a record ceases to belong to our region we must not emit a
deletion:

```
changeKind: "Deleted"     →  the entity ceased to exist       (aggregator removes it)
changeKind: "OutOfScope"  →  the entity is no longer ours     (aggregator keeps it,
                                                               reassigns regional custody)
```

Conflating them makes a regional transfer quietly erase the record from the global aggregator —
a data-loss bug with no local symptom whatsoever.

### 8. Extensions, with a registry

Regional particulars may require fields the base contract does not carry, so an
`additionalAttributes` bag is provided. It carries a known failure mode: within a year it becomes
the place where anything nobody wanted to negotiate is dumped, and no one can say what is in it.

The guard is a registry. **Every extension key is registered in `MAPPING_MATRIX.md`** with its
type, owner and region, and an unregistered key inbound is dead-lettered. The escape hatch stays;
the dumping ground does not appear.

### 9. Vocabulary reconciliation is a one-time workstream, not a task

`Acme` already has reference tables whose codes grew independently of the aggregator's. Before any
of the above matters, the two vocabularies must be mapped to each other by hand: our `APT` against
their `ALPHA`, our three sub-kinds against their one, our seven statuses against their five.

This is manual, does not automate, and is routinely larger than the pipeline it enables. It starts
in stage 0, in parallel with requesting the schema — not when the pipeline is ready, or the
pipeline will be ready and idle.

## Alternatives considered

**Keep picklists as compiled C# enums.** What the design does today. *Rejected*: it makes our
release cadence a dependency of the aggregator's data entry. A reference row added on Tuesday
blocks every entity carrying it until our next deployment, and the failure presents as a flood of
dead letters rather than as "a new value appeared".

**Replicate reference data through the same outbox/inbox.** One mechanism instead of two.
*Rejected*: the semantics do not match at any point — one-way rather than bidirectional, no
conflicts to resolve, versioned per set rather than per field, and no ordering requirement between
sets. Forcing it through the entity pipeline means carrying merge machinery that can never fire
and a per-item version that has no meaning.

**Look values up on the aggregator on demand.** No replica to keep fresh, never stale.
*Rejected*: it puts a network call on the validation path of every inbound message, couples our
processing availability to theirs, and multiplies request volume by the number of picklist-backed
fields.

**Cache with a TTL and no version.** Simpler than versioned sets. *Rejected*: there is no atomic
apply, so a partially-refreshed cache is observable; and "empty because it expired" is
indistinguishable from "empty because the set is empty", which is exactly the ambiguity the
startup gate exists to remove.

**Treat the region as a deployment detail only.** True today — one region per instance — and it
would let us omit `region_code` entirely. *Rejected on cost asymmetry*, §6: the column is free now
and expensive later, and the API path is a breaking change later.

## Consequences

### Positive

- A new aggregator value is absorbed without a release, because the domain stores codes rather
  than compiled enums.
- Reference data gets a mechanism proportional to it — no outbox, no merge, no clock.
- The domain's schema stays independent of the integration schema; the projection is the only seam.
- Historical records stay valid when the aggregator withdraws a value.
- Multi-region becomes a configuration and query change rather than a migration.
- Two silent data-loss paths are closed by name: entities blocked on a stale replica, and regional
  transfers read as deletions.

### Negative / costs accepted

- **A second pipeline to build and operate** — worker, versioning, projection, staleness SLI,
  runbook entry.
- **A hard ordering dependency**: reference data must be current before entity processing, so a
  reference-side outage stalls the entity path even when the entity path is healthy.
- **Unbounded vocabulary growth is now possible.** Absorbing values without a release means nobody
  reviews them; the unmapped-value counter and the extension registry are the only controls.
- **`region_code` is carried everywhere from the start**, including through a period where it has
  exactly one value — visible dead weight until the second region exists.
- **Vocabulary reconciliation is manual and unbudgeted**, and it gates go-live.
- **Two enum strategies now coexist**, and choosing the wrong one for a field is a silent error:
  a picklist treated as an owned enum blocks on releases, an owned enum treated as a picklist
  loses its exhaustiveness guarantee. `MAPPING_MATRIX.md` must record which class each field is in.
