# ADR-0008 — Bidirectional synchronisation: per-field last-writer-wins over a three-way merge

- **Status:** Proposed
- **Date:** 2026-07-25
- **Related:** ADR-0003 (compaction, delivery), ADR-0004 (payload shape), ADR-0006 (backfill),
  ADR-0007 (staging)
- **Open question:** Q18 — what metadata the aggregator can send and echo. The design works at
  the worst answer; each capability it does have removes one approximation.
- **Amends:** ADR-0004 §3 (the envelope gains `changedFields[]`), ADR-0003 §4 (compaction unions
  `changedFields` instead of superseding them)

## Context

ADR-0004 chose full-snapshot payloads. That decision was taken under an assumption which has
since been contradicted: it treated `Acme` as the writer and the peer as a reader. The actual
requirement is that **every field of `Foo`, `Bar` and `Baz` is editable in both systems**, with
no natural partition of ownership — the operator screens on both sides expose the same record.

Under that requirement, snapshot-apply has a failure mode it does not have one-way:

```
10:00:00  both systems:  price = 300k, contact.phone = +7 900 …
10:00:05  peer opens the record and changes ONLY the phone
10:00:07  Acme changes price = 280k and emits a snapshot
10:00:09  the peer emits its snapshot — assembled from its 10:00:05 read,
          in which price is STILL 300k
10:00:11  Acme applies the peer snapshot wholesale  →  price reverts to 300k
```

Nobody deleted anything. Both edits were correct. One is gone, with no record that it ever
existed. This is the classic full-document overwrite, and it cannot occur while only one side
writes — which is why nothing in the existing documents addresses it.

Three further gaps follow from the same assumption:

1. **`aggregateVersion` cannot express causality.** It is a scalar we own, monotonic in our own
   emission. It answers "is this message newer than our previous one". It cannot answer the only
   question that matters here — *was the peer's edit made in knowledge of our version N, or
   blind?* One counter cannot represent two writers.
2. **No resolution model exists.** Today the winner is whichever message happened to be applied
   last, which is a function of delivery timing.
3. **A lost update is invisible.** Compaction (ADR-0003 §4) discards intermediate states, which
   are exactly the evidence that a conflict occurred. Reconciliation reports divergence but
   cannot say who was right or what was lost.

The team's resolution policy is **automatic everywhere** — no human quarantine queue, no
blocking review step.

## Decision

### 1. Three-way merge against a shadow baseline

We keep, per aggregate and per peer, the **last known state of the peer**. "What changed on
their side" is then computed rather than asked for:

```
        shadow (baseline)  ←  what we last saw from the peer
       /                 \
   ours                   theirs
```

```sql
CREATE TABLE integration.peer_shadow (
    aggregate_type text        NOT NULL,
    aggregate_id   uuid        NOT NULL,
    peer_system    text        NOT NULL,
    region_code    text        NOT NULL,
    state          jsonb       NOT NULL,   -- canonical contract-shaped snapshot
    state_hash     bytea       NOT NULL,
    updated_at     timestamptz NOT NULL,
    PRIMARY KEY (aggregate_type, aggregate_id, peer_system)
);
```

The merge is evaluated **per field**:

| base vs ours | base vs theirs | Action |
|---|---|---|
| = | = | nobody changed it — nothing |
| ≠ | = | only we changed it → **keep ours**, emit |
| = | ≠ | only they changed it → **take theirs** |
| ≠ | ≠, same value | converged independently → update the shadow only |
| ≠ | ≠, different values | **genuine conflict** → last-writer-wins (§3) |

Only the last row consults a clock. The other four are deterministic and lossless, and in
practice they carry most of the traffic: the problem that motivates this ADR is solved without
comparing timestamps at all.

### 2. `changedFields[]` in the envelope

The payload stays a full snapshot — that is what makes the design tolerant of a lost message and
keeps compaction legal. Alongside it travels the list of fields the sender actually touched:

```jsonc
{
  "aggregateVersion": 42,
  "changedFields": ["contact.phone"],   // merge metadata, not payload
  "data": { /* full snapshot */ }
}
```

Only the listed fields are applied; the rest of the snapshot is used to refresh the shadow. This
single addition removes the failure in the Context section.

**When the peer sends no such list, we derive it** by diffing their snapshot against the shadow.
The design therefore works against a peer that sends bare state with no metadata at all; the
metadata only makes it cheaper and more precise.

### 3. Per-field last-writer-wins, with a hybrid logical clock

```sql
CREATE TABLE integration.field_state (
    aggregate_type text        NOT NULL,
    aggregate_id   uuid        NOT NULL,
    field_path     text        NOT NULL,   -- 'price.amount'
    updated_at     text        NOT NULL,   -- hybrid logical clock, not wall clock
    updated_by     text        NOT NULL,   -- 'acme' | 'peer-a'
    PRIMARY KEY (aggregate_type, aggregate_id, field_path)
);
```

- **Hybrid logical clock** `(wallMs, counter, nodeId)`, not wall clock. Clock skew of a few
  seconds between services produces the wrong winner, deterministically and silently.
- **Tie-break by system identifier**, lexicographically — deterministic and identical on both
  sides, so both converge on the same answer without coordinating.

Two implementation rules that decide whether this converges at all:

- **Applying a remote change must not refresh the timestamp.** Record *their* HLC and
  `updated_by = peer`. Writing `now()` makes our echo always look newer, the peer accepts it,
  answers with its own echo, and the field oscillates forever. This is the single most common way
  a last-writer-wins integration fails.
- **Collections merge per item**, keyed by a stable item id. A collection treated as one field
  reproduces the whole-document overwrite one level down: the peer adds a `FooAttachment`, we add
  ours, one disappears.

### 4. Partial application, in the application layer

The inbound translator produces a command that updates **only the fields the merge resolved** —
never a whole-entity replace. These are new application-layer commands
(`UpdateFooFieldsCommand`); the domain is unchanged and every invariant still runs.

Field-level echo suppression follows: fields we applied from the peer are not reported back as
changed by us, or `changedFields` lies and the loop returns.

### 5. `conflict_log` — the compensating control for automatic resolution

Because conflicts resolve without review, the losing value exists nowhere else:

```sql
CREATE TABLE integration.conflict_log (
    id             bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    aggregate_type text NOT NULL,
    aggregate_id   uuid NOT NULL,
    field_path     text NOT NULL,
    our_value      jsonb,
    their_value    jsonb,
    winner         text NOT NULL,
    our_ts         text,
    their_ts       text,
    resolved_at    timestamptz NOT NULL DEFAULT now()
);
```

It blocks nothing. It exists so that "who reverted the price on this record" has an answer a
month later. Without it the question is unanswerable in principle.

### 6. Flap detector

A field changing more than *N* times within a window suspends synchronisation **for that field**
and raises an alert. Convergence depends on the peer implementing a compatible tie-break; if it
does not — or if it simply always overwrites — the two systems can ping-pong indefinitely. With
last-writer-wins everywhere and a peer whose behaviour we cannot verify, this is not paranoia: it
is the difference between an alert and a week of traffic burned on one oscillating record.

### 7. Amendment to compaction

ADR-0003 §4 supersedes a pending delivery when a newer message for the same aggregate appears.
That stays, with one change: the superseding message's **`changedFields` is the union** of its own
and those of the messages it supersedes. Taking only the latest would drop the fact that an
earlier edit touched a different field, and the peer would merge against an incomplete list.

## Alternatives considered

**Field ownership — single writer per field.** The cheapest correct answer, and the first thing to
try: assign every field an owner, and the merge problem disappears. *Rejected on the facts* — the
team confirmed there is no natural partition; every field is editable in both systems. Should a
partition emerge later, it is a strict simplification of this design, not a replacement: owned
fields simply skip the merge.

**Quarantine every conflict for human resolution.** Loses nothing, and for a small set of critical
fields it is the right answer. *Rejected as the policy* — the team chose automatic resolution, and
a queue that nobody drains is worse than a rule, because divergence then waits behind a backlog
instead of converging.

**Acme always wins.** Simple and deterministic on our side, and tempting because we are the
system of record for the region. *Rejected*: it silently discards every peer edit, which makes
the peer's UI a lie; and it converges only if the peer concedes — if it assumes the same, both
sides keep overwriting each other, which is the oscillation of §6 with extra steps.

**Version vectors with explicit conflict detection.** Strictly better than last-writer-wins: it
distinguishes a genuinely concurrent edit from a stale one, instead of guessing by clock. It
requires the peer to echo the version of our record it was working from (`basedOnVersion`).
*Not rejected — deferred*, pending Q18. If the peer can echo it, this becomes an upgrade to §3
with no change to §1, §2 or §4.

**CRDTs.** Convergence by construction, no clocks, no conflicts. *Rejected*: the fields here are
scalars with business meaning — a price, a status, an address — not counters or grow-only sets.
A CRDT does not tell you which price is correct; it only guarantees both sides agree on the same
wrong one. The cost is large and the benefit does not apply.

**Delta payloads instead of snapshots.** Would carry only changed fields and remove the overwrite
directly. *Rejected*: deltas require strict ordering (ADR-0003 §4 explains what that costs), lose
the ability to re-converge after a dropped message, and need a separate snapshot endpoint for
bootstrap. `changedFields` metadata over a snapshot buys the same property without any of it.

## Consequences

### Positive

- The overwrite in the Context section cannot occur: untouched fields are never applied.
- Four of the five merge cases are deterministic and lossless; the clock is consulted only for a
  genuine concurrent edit.
- Works against a peer that sends no metadata whatsoever, because the shadow supplies the diff.
- Convergence does not require the two systems to coordinate at runtime — only to share a
  tie-break rule.
- Every automatic resolution is recorded, so a lost edit is explainable after the fact.
- Compatible with the staging in ADR-0007: none of this is needed until stage 5.

### Negative / costs accepted

- **Lost updates are now policy, not accident.** On a genuine concurrent edit one value is
  discarded automatically. Deterministically, per field, and logged — but discarded.
- **Storage roughly doubles per synchronised aggregate.** The shadow holds a second copy of the
  contract-shaped state, and `field_state` holds a row per field.
- **Correctness partly depends on behaviour we do not control.** If the peer's tie-break differs,
  convergence degrades to oscillation; the flap detector contains the damage but does not fix the
  cause.
- **The shadow must be initialised.** Until backfill (ADR-0006) populates it, the first inbound
  merge for an aggregate has no baseline and must fall back to taking the peer's state whole —
  precisely the behaviour this ADR exists to avoid. Backfill therefore becomes a prerequisite for
  inbound, not a follow-up.
- **A canonical serialiser is now load-bearing.** The shadow, the hash and the diff all depend on
  one stable JSON form; changing it invalidates every stored baseline.
- **More moving parts in the inbound path**: merge, HLC, echo suppression and the flap detector
  all sit between a received message and a domain command.
