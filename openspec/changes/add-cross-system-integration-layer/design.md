# Design — critical review of the integration design

Written while deriving the specs in `specs/`. Everything below is a finding against the existing
documents, with the evidence, the concrete failure it produces, and a recommended resolution.
Nothing here edits an ADR: findings that change a decision need an amendment (the ADRs are still
`Proposed`) or a superseding ADR.

---

## 1. What holds up

Stated first, because the register below is long and the design is mostly right.

| Claim | Verdict |
|---|---|
| Two-stage outbox: cheap transactional capture, deferred materialisation | **Correct and the best decision in the set.** It buys compaction, re-materialisation for contract v2, dual-version publishing, and mapping-bug recoverability. ADR-0002 §2's five reasons are all real. |
| `SaveChangesInterceptor` capture in the same `SaveChangesAsync` | **Correct.** Genuinely removes the dual write, and the rollback test is the right executable statement of it. |
| `xid8` visibility watermark on the *feed* | **Correct, and the single most valuable line in the design.** The sequence-assigned-at-insert / visible-at-commit gap is a real silent-loss bug that testing does not find. |
| `FOR UPDATE … SKIP LOCKED` over leader election | **Correct.** Right trade for this volume; scales with replicas at zero coordination cost. |
| Compaction as the answer to per-aggregate ordering | **Correct**, and it is what makes out-of-order delivery harmless rather than merely unlikely. |
| Snapshot payloads as the default | **Correct**, and ADR-0004 §3's comparison table is honest about the cost. |
| Backfill as a consumer-driven paged pull with a pinned watermark | **Correct.** No freeze window, resumable by construction, and one mechanism serving initial load, DR and onboarding. |
| Losslessness enforced by Mapperly diagnostics + a reflection coverage test | **Correct in kind.** Compile-time beats review-time. |
| Permanent-vs-transient error classification, 422 straight to dead letter | **Correct**, and it designs out the most common production outbox failure. |
| "Only the dispatcher's transport changes when Kafka arrives" | **Fair.** The outbox, contracts, mapping and inbox do survive. |

The defects below are almost all in the **seams between** these decisions, not in the decisions
themselves — which is the normal place for a design of this size to be wrong.

---

## 2. Defect register

Severity: **Blocking** = the design cannot be implemented as written, or ships a silent-loss
bug. **Major** = implementable but produces a known incident. **Minor** = internal
inconsistency that will cost an implementer time.

### D1 — `aggregateVersion` cannot be known when the envelope is built · Blocking

`TECHNICAL_PROPOSAL.md` §5.2 orders the materialiser loop: step 6 "compute `aggregateVersion`
(see §5.3), build the envelope, compute `payload_hash`", step 8 "insert `outbox_message`". §5.3
recommends option 3 — "**the outbox `sequence`**" — as the phase-1 default. But `sequence` is
`bigint GENERATED ALWAYS AS IDENTITY` (§4.2): its value exists only *after* the insert. The
envelope embeds `aggregateVersion`, the payload embeds the envelope, and `payload_hash` is
computed over the payload, so the ordering is circular.

Workarounds all carry costs the documents do not price: pre-allocating with `nextval` requires
`OVERRIDING SYSTEM VALUE` on a `GENERATED ALWAYS` column; insert-then-update rewrites a `jsonb`
column on the hot path and invalidates the hash; a second identity column duplicates the problem.

**Recommendation:** adopt §5.3 option 2 — a dedicated `IntegrationVersion` shadow property
(`builder.Property<long>("IntegrationVersion")`) bumped per aggregate — as the *primary*
mechanism, not the upgrade path. It is knowable before the insert, it is what a consumer
actually wants (a version tied to our state, not to our emission order), and it also fixes D2.
The proposal's own argument that this "is not a domain-model change because it is an EF shadow
property" is sound; the reason to defer it was simplicity, and D1 removes that reason.

### D2 — Backfill snapshot pages carry no `aggregateVersion` · Blocking

ADR-0006 §2: backfill "pages are materialised **on demand** from current domain state and are
**not written to the outbox at all**". ADR-0006 §1 then relies on "the `aggregateVersion`
idempotency rule (ADR-0004 §3)" to make double application during the handover converge.

With `aggregateVersion` = the outbox `sequence` (D1), a page item has no version to carry.
Worse, at go-live most of the 500 k-row catalogue has **never produced an outbox message** —
capture only started at phase 0 — so no sequence exists for it even in principle. The
idempotency contract is therefore undefined for exactly the messages that bootstrap the peer,
and ADR-0006 §3's guard ("the dispatcher refuses to send a backfill message whose
`aggregateVersion` is lower than an already-delivered live one") has nothing to compare.

**Recommendation:** D1's per-aggregate counter, which is defined for every row from the moment
the column exists. If option 3 is kept anyway, the fallback is to stamp every item of a run with
the run's pinned watermark `W`: it is comparable, it is below every post-handover live message,
and it converges — but it makes "version" mean two different things depending on which endpoint
served the message, which is a contract wart worth avoiding.

### D3 — The dispatcher claim SQL does not order by priority · Blocking

ADR-0006 §3 promises: "**The dispatcher always drains priority 0 first**, so a two-day backfill
never delays a live price change." `TECHNICAL_PROPOSAL.md` §5.4 step 2 says "priority 0 before
priority 9". `TECHNICAL_PROPOSAL.md` §4.3 even builds the index for it —
`ix_delivery_claim (subscriber_id, priority, next_attempt_at, id)` with the comment "column
order matches `ORDER BY` exactly".

The claim SQL in ADR-0003 §2 — the one both other documents point at — is
`ORDER BY next_attempt_at, id`. No `priority`. With 500 k backfill rows queued at
`next_attempt_at = now()` before a live change arrives, every live change queues behind the
entire backfill. The exact starvation the design says is structurally impossible is what it
implements.

**Recommendation:** amend ADR-0003 §2 to `ORDER BY priority, next_attempt_at, id`. One-line fix;
the reason it is Blocking rather than Minor is that the SQL is the artefact implementers copy,
and the failure it produces (a two-day live-traffic stall at go-live) is severe and only
observable under backfill load.

### D4 — The compaction statement references a column that does not exist, and cannot see `compactable` · Major

ADR-0003 §4:

```sql
UPDATE integration.outbox_delivery
   SET status = 'Superseded', ...
 WHERE subscriber_id = @subscriberId
   AND aggregate_key = @aggregateKey
   AND status        = 'Pending'
   AND sequence      < @newSequence;
```

`integration.outbox_delivery` (§4.3) has no `sequence` column — it has `message_sequence`. The
statement does not compile.

Second, larger problem in the same statement: ADR-0003 §4 disables compaction for
`ChangeKind = Event` messages via "`compactable = false`", but `compactable` is a column on
`outbox_message` (§4.2), not on `outbox_delivery`. The `UPDATE` as written will happily supersede
a pending `Event` delivery — losing a transition message the design guarantees is always
delivered. Fixing it needs either a join to `outbox_message` (across a partitioned table, on
`(message_id, created_at)`) or a denormalised `compactable` column on `outbox_delivery`.

Also drifting: ADR-0003 §2 names the claim index `ix_outbox_delivery_claim` over
`(subscriber_id, next_attempt_at, id)`; §4.3 names it `ix_delivery_claim` over
`(subscriber_id, priority, next_attempt_at, id)`.

**Recommendation:** denormalise `compactable` onto `outbox_delivery` at fan-out time (it is
immutable per message), fix the column name, and reconcile the index definitions in one edit.

### D5 — The `xid8` watermark is applied to stage 1, where its rationale does not hold, and it inflates the primary SLI · Major

ADR-0003 §5 closes with "The same watermark is applied to the dispatcher's materialisation scan
for the same reason", and `TECHNICAL_PROPOSAL.md` §5.2 step 1 claims change-log rows "below the
`xid8` watermark".

The reason does not transfer. The feed needs the watermark because it hands out a **monotonic
cursor** that a consumer advances past rows it could not see. Stage-1 claiming has no cursor: it
selects `WHERE status = 'Pending'` under row locks. A change-log row inserted by a
still-running transaction is simply invisible; when it commits it becomes visible and is claimed
on the next poll. No row can be skipped, because nothing has moved past it.

The cost is real and lands on the wrong metric. The watermark delays *all* materialisation by the
duration of the longest concurrent write transaction, and
`integration_outbox_pending_age_seconds` is the primary SLI with a page at 300 s (§9.1, §9.2).
One long-running domain transaction — a report, a bulk import, an idle transaction left open by
a connection-pool bug — pages on-call for a pipeline that is working correctly.

(Incidental: it is the *materializer*, not the dispatcher, that scans the change log.)

**Recommendation:** drop the watermark from stage 1 and state why it is unnecessary there — the
distinction between cursor-based and status-based claiming is exactly the kind of reasoning an
ADR exists to preserve. Keep it on the feed, where it is load-bearing.

### D6 — Echo suppression depends on a field that exists nowhere, and cannot break a loop longer than one hop · Blocking

`TECHNICAL_PROPOSAL.md` §6.2: "**Echo suppression is required**: messages produced from an
inbound message of source S carry `causationSource = S` and are not fanned out back to S."

`causationSource` does not exist. `outbox_change_log` (§4.1) has `correlation_id`,
`causation_id`, `trace_parent`, `actor_id`. `outbox_message` (§4.2) has `correlation_id` and
`trace_parent`. The envelope (ADR-0004 §3) has `correlationId`, `traceParent` and `source` —
where `source` is `"acme"`, i.e. *us*, not the inbound origin. There is no column to
write it to, no envelope field to carry it, and no fan-out predicate to read it.

Worse, direct-source suppression is not sufficient even once implemented. With three systems
that all replicate state and all suppress only their immediate sender, A → B → C → A closes a
loop no participant filters: every hop is a local write, every local write bumps the version, so
the consumer-side `aggregateVersion` check — the backstop for everything else in this design —
never stops it. This is the classic multi-master state-replication loop, and it is
indistinguishable from real traffic on a dashboard.

**Recommendation:** two mechanisms, both cheap:

1. **Content-based no-op suppression as the primary defence.** The design already computes
   `payload_hash` over the canonical payload. If the newly materialised hash equals the last
   emitted hash for that `(aggregate, subscriber variant)`, emit nothing. This terminates loops
   of *any* topology, because a loop by definition converges on a fixed point, and it also
   removes the pointless traffic from writes that change nothing externally visible (an internal
   note edited on a captured aggregate).
2. **A propagation path, not a source.** Carry the ordered list of systems a change has already
   traversed (`propagationPath: ["peer-a", "acme"]`) in the envelope and in the
   change log, and refuse fan-out to any system already on the path. This needs a column, an
   envelope field (ADR-0004 §3), and agreement from the peer — so it is a contract change, which
   is why (1) should not wait for it.

### D7 — Per-subscriber payload variants are not modelled · Blocking

ADR-0005 §3: "Fields flagged `pii` … are emitted only to subscribers whose entry carries the
`pii` grant. **The materializer produces a redacted variant otherwise.**" `subscriber` has
`pii_granted` (§4.4).

But `outbox_message` (§4.2) stores exactly one `payload` and one `payload_hash`, and
`TECHNICAL_PROPOSAL.md` §5.2 step 8 keys the fan-out on `contract_version` only:
"fan out `outbox_delivery` rows for every enabled subscriber … at each subscriber's
`contract_version`". `outbox_delivery` has no payload column. So the redacted variant has
nowhere to live, and two subscribers differing only in `pii_granted` are served the same bytes —
in whichever direction the materialiser happens to resolve it. If it resolves to the full
payload, that is a compliance incident (R7) delivered by the mechanism meant to prevent it.

The same hole applies to subscriber-scoped `filter_expression` (§4.4) if filters ever affect
payload *content* rather than only whether a message is emitted.

**Recommendation:** make the variant an explicit dimension: `payload_variant text NOT NULL`
(`'Full' | 'Redacted'`, extensible) on `outbox_message`, part of the fan-out key alongside
`contract_version`, part of `ux_outbox_message_id`'s uniqueness reasoning, and part of every
statement that compares hashes. Rendering at dispatch time is the tempting alternative and
should be rejected: it puts mapping on the delivery path, and it leaves `payload_hash`
undefined, which breaks reconciliation (D8) and the dark-launch validation of phases 0–1.

### D8 — Reconciliation checksums are global, but every subscriber's expected set is not · Blocking

ADR-0006 §4: "Each aggregate contributes `sha256(canonical_contract_json)`; bucket hash is the
XOR of member hashes… The consumer compares bucket hashes."

Two things make the consumer's comparison fail by construction:

- **Filters.** `MAPPING_MATRIX.md` §5 and `subscriber.filter_expression` mean a subscriber
  legitimately holds a *subset* of the catalogue. Every bucket containing an out-of-scope record
  mismatches. With 256 buckets over 500 k rows, one filtered subscriber diverges on
  approximately all of them.
- **Redaction (D7).** A subscriber without the PII grant holds different *content*, so its hash
  differs even for records it correctly has.

The nightly job's output for such a subscriber is 256 mismatches every night, which is not an
alert — it is noise that trains everyone to ignore
`integration_reconciliation_divergent_total`. And ADR-0006's own fallback ("we still run it
internally against our own last-delivered `payload_hash` set") inherits the problem, because
last-delivered hashes are per variant.

**Recommendation:** scope the checksum endpoint to the authenticated subscriber — its filter,
its contract version, its payload variant — and say so in the endpoint contract. Pin the
canonicaliser per contract version as ADR-0006 already requires, and add the variant to what is
pinned.

### D9 — Stage-1 failures have no retry policy, no terminal state, and no alert · Major

`outbox_change_log` (§4.1) carries `status … CHECK (status IN ('Pending','Materialized','Failed','Skipped'))`,
`attempt` and `last_error` — but no `next_attempt_at`. ADR-0003 §6's backoff schedule, jitter,
dead-lettering and alerting are specified for `outbox_delivery` and (§7) for `inbox_message`.
Nothing states what happens to a change-log row whose materialisation throws.

Consequences, all silent: `ix_change_log_claim` is `WHERE status = 'Pending'`, so a row moved to
`Failed` leaves the pipeline entirely — no delivery, no `dead_letter` row (§4.6's `dead_letter`
has no change-log linkage), no metric (§9.1 counts pending, not failed), no alert (§9.2 has no
row for it). A materialiser bug affecting one aggregate type therefore produces a permanent,
unannounced data gap that only the nightly reconciliation might notice.

`'Skipped'` compounds it: the value appears only inside the `CHECK` constraint and is defined
nowhere. Presumably it means "filtered out per `MAPPING_MATRIX.md` §5" — but a filtered-out row
and a row that failed to filter are the same state from the outside.

**Recommendation:** give stage 1 the same three things stage 2 has — `next_attempt_at`, the
ADR-0003 §6 schedule, and a terminal path that writes a `dead_letter` row with
`direction = 'Outbound'`, `failure_kind = 'Mapping' | 'Validation'` — plus
`integration_changelog_failed_total` and an alert. Define `'Skipped'` in ADR-0002 or drop it
from the constraint.

### D10 — The retention table contradicts the DDL · Minor

ADR-0003 §8 versus `TECHNICAL_PROPOSAL.md` §4:

| ADR-0003 §8 says | The DDL says |
|---|---|
| `outbox_delivery`: "30 days after terminal state — follows its parent partition" | §4.3 is an unpartitioned table with `bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY`; it cannot follow a partition it is not part of, and partitioning it by `created_at` would require changing that PK |
| `inbox_message`: "30 days after `Processed` — monthly partition" | §4.5 is unpartitioned |
| `outbox_message`: monthly `RANGE` partition, `DETACH` + drop | §4.2 defines partitioning but then recommends shipping **unpartitioned** with batched `DELETE` |

So the phase-1 reality is `DELETE` on three hot queue tables — precisely what ADR-0003 §8 argues
against ("deleting from a hot queue table produces bloat and unpredictable autovacuum behaviour
exactly when the queue is busiest").

**Recommendation:** state the phase-1 retention mechanism per table explicitly, accept the
`DELETE` with an autovacuum note, and record the partitioning trigger (§4.2's "~10 M rows") as
the condition that moves each table to the target state.

### D11 — Tombstones will fail peer-schema validation, so every delete dead-letters · Blocking

`TECHNICAL_PROPOSAL.md` §5.2 step 7 validates every payload against the peer's vendored JSON
Schema and dead-letters on failure "with the offending field named". ADR-0004 §3 defines `data`
as "the peer's Foo schema, **fully populated**". ADR-0002's delete handling emits a
tombstone `{id, deletedAt}` built from `ChangeTracker` original values — there is no snapshot and
no mapper run, because the row is gone.

A tombstone's `data` is not a fully populated Foo. Against any schema with required fields —
and `MAPPING_MATRIX.md` §2 marks `title`, `kind`, `status`, `price`, `measure`,
`address.street`, `country`, `effectiveOn`, `createdAt`, `updatedAt` as required — validation fails.
Every hard delete dead-letters, and so does every tombstone from the three retraction rules in
`MAPPING_MATRIX.md` §5 (draft retraction, soft delete, subscriber scope exit). The design's
handling of deletes is one of its stated strengths ("Deleted aggregates appear as tombstone
envelopes, not as absences", §5.5) and as specified it cannot deliver one.

**Recommendation:** treat the delete representation as a first-class contract question, not an
implementation detail. It needs: which message type/schema the peer expects for a deletion
(a separate `acme.foo.deleted` schema, a `changeKind` discriminator with a relaxed
`data`, or a documented sentinel), and a validation rule that selects the schema by
`changeKind`. Add it to `MAPPING_MATRIX.md` §7 — it is a peer gap, and it is a bigger one than
`‹externalCode›`.

### D12 — "Leaves a subscriber's scope → tombstone" cannot be implemented from the retained state · Major

`MAPPING_MATRIX.md` §5: "a record leaving a subscriber's scope emits a tombstone **to that
subscriber only**", flagged as "easy to miss and produces permanently stale records at the
peer. It is tested explicitly."

Deciding that a record *left* scope requires knowing it was *in* scope for that subscriber
before. The only state that records this is `outbox_delivery`, retained 30 days (ADR-0003 §8).
For any aggregate not changed in the last 30 days — which, in a 500 k-row catalogue at 10 k
changes/day, is most of it — the history is gone, the materialiser cannot distinguish "left
scope" from "was never in scope", and no tombstone is emitted. The peer keeps the record
forever. This is the failure mode the rule exists to prevent, reached by a different route.

Related, and worth its own decision: `subscriber.filter_expression jsonb` (§4.4) is an
expression language with no grammar, no evaluation semantics, no null/missing-field behaviour,
and no stated limits. It is data controlled through a table, evaluated against domain snapshots
inside the materialiser — an interpreter we own, on the emission path, outside the type system
and outside the mapping test suite.

**Recommendation:** add a durable per-`(subscriber, aggregate)` projection state row
(last emitted version, last variant, in-scope flag, last hash) that also carries D6's suppression
hash and D7's variant, and derive scope transitions from it rather than from delivery history.
Then either specify `filter_expression`'s grammar and evaluation rules with tests, or restrict
phase 1 to a fixed set of named, code-defined filters selected by key — which covers the region
example and keeps the interpreter out of the design.

### D13 — The consumer-facing contract document does not exist · Major

Three places treat it as an existing artefact: ADR-0003 §3 ("This requirement is stated in the
consumer-facing contract document, not left implicit"), ADR-0003 §8 ("This is stated in the
contract"), ADR-0006 §5 ("The consumer's rule, stated in the contract"), and ADR-0001's
Negative section makes it a hard requirement ("it is a hard requirement we must document in the
consumer-facing contract"). `README.md`'s document table lists four documents and six ADRs; none
of them is it. `project.md`'s consistency table already has a row pointing at "the
consumer-facing contract statements".

Everything a consumer must do to make this design correct lives only in our internal reasoning:
idempotency by `aggregateVersion` or `messageId`, tolerating at-least-once and double delivery
during handover, the 30-day catch-up window and the 410 response, ignoring unknown fields,
tombstone handling, the checksum comparison, and (per D6) whatever loop-prevention field is
agreed. A design whose correctness depends on consumer behaviour and does not state that
behaviour anywhere the consumer can read is not finished.

**Recommendation:** add `CONSUMER_CONTRACT.md` as a top-level document (with its README row),
owning exactly the obligations we place on the other side. It is also the natural home for the
push-vs-pull stream difference in S4.

### D14 — Batch failure isolation is unspecified · Major

`TECHNICAL_PROPOSAL.md` §5.2: "Steps 7–10 run in one transaction, so a crash re-materialises
rather than half-emitting." Step 7 also dead-letters a schema-invalid payload "now".

For a batch of 100 aggregates, these two sentences conflict. If the dead letter and the marking
are in the batch transaction, one poison aggregate either rolls back 99 correctly materialised
messages or is committed alongside them — and if the failure is an exception rather than a
validation result, the batch aborts, the rows return to `Pending`, and the poison row is retried
with the same 99 on every cycle. That is the poison-message retry loop ADR-0003 §6 designs out
for delivery, reintroduced at materialisation. It is also how a single bad record stalls the
primary SLI.

**Recommendation:** state the transaction boundary as **per aggregate**, batched only for the
claim and the read, with the poison row's terminal handling (D9) committed independently of its
batch peers.

---

## 3. Smaller inconsistencies

| # | Finding | Where |
|---|---|---|
| S1 | The ingress sample dereferences `ctx.User.FindFirst("client_id")!.Value`. Under ADR-0005 Tier 2 (mTLS) and Tier 3 (HMAC) there is no `client_id` claim, so a correctly authenticated request throws a `NullReferenceException` → 500, and the peer reads that as delivery failure and retries. `source_system` must come from the authenticated principal via the `IIntegrationAuthenticator` seam, not from an OAuth-specific claim. | proposal §6.1 vs ADR-0005 §2 |
| S2 | Per-aggregate stream locking for `ChangeKind = Event` messages appears in one sentence, with no table, no column, no claim SQL, and no entry in `IMPLEMENTATION_PLAN.md`. It is the only ordered-delivery machinery in the design and it is unpriced. | ADR-0003 §4 |
| S3 | "Replay" means two incompatible things: reset `outbox_delivery` to `Pending` (needs the `outbox_message` payload, retained 30 days) and re-send from `dead_letter.payload` (retained 180 days). A dead letter older than 30 days cannot be replayed by the first mechanism, and the runbook does not distinguish them. | ADR-0003 §6, proposal §9.5, §4.6 |
| S4 | "One log, two read paths. **Push and pull cannot diverge because they read the same rows.**" They read the same table but not the same rows: push delivers a *compacted subsequence*, the feed exposes every message. Converged state is identical; the message stream is not. A consumer migrating from push to pull sees messages it never received, and one migrating the other way stops seeing intermediate states. Worth stating rather than denying. | ADR-0001 Consequences vs ADR-0003 §4 |
| S5 | ADR-0006 §3's guard needs `max(aggregate_version)` among **delivered** rows per `(subscriber, aggregate)`. `ix_delivery_compaction` is partial on `status = 'Pending'`, so the guard has no index. On a 500 k-row backfill that is 500 k unindexed lookups. | proposal §4.3 vs ADR-0006 §3 |
| S6 | `outbox_change_log.status = 'Skipped'` exists only inside a `CHECK` constraint; its meaning is defined nowhere (see D9). | proposal §4.1 |
| S7 | "~4 weeks for one engineer to production-ready" covers outbox, inbox, feed, backfill, checksums, reconciliation, OAuth2 **and** an HMAC fallback, admin API, analyzer, load tests, runbook, a game day and a six-phase rollout. It also excludes the peer-coordination latency that Q3 makes the actual critical path, and `MAPPING_MATRIX.md` §7 has eight open gaps each needing a round trip with another team. The engineering estimate is defensible; the calendar is not, and the two are being reported as one number. | proposal §1, IMPLEMENTATION_PLAN |

---

## 4. New open questions

To be folded into `TECHNICAL_PROPOSAL.md` §2.3 and `MAPPING_MATRIX.md` §7 (numbering continues
from Q8 / §7.8). Each has a default so nothing blocks on silence.

| # | Question | Defect | Default if unanswered |
|---|---|---|---|
| Q9 | Confirm `aggregateVersion`'s source: per-aggregate counter (§5.3 option 2) or emission sequence (option 3)? | D1, D2 | **Option 2** — option 3 is not implementable as ordered |
| Q10 | How does the peer represent a deletion — separate message type, discriminated `data`, or not at all? | D11 | **Separate `*.deleted` message type**; validate by `changeKind` |
| Q11 | Which loop-prevention mechanism does the peer support: an envelope propagation path, or do we rely on content-hash suppression alone? | D6 | **Content-hash suppression** on our side; no contract change |
| Q12 | Do any subscribers differ in PII grant or filter scope in phase 1? If yes, `payload_variant` is required on day 2, not later. | D7, D8 | **Assume yes** — model the variant from the start |
| Q13 | Is the checksum endpoint subscriber-scoped (their filter + variant) or global? | D8 | **Subscriber-scoped** |
| Q14 | Does any consumer need `ChangeKind = Event` transition messages in phase 1? | S2 | **No** — defer stream locking entirely |
| Q15 | Which subscriber filters are actually needed in phase 1? | D12 | **A fixed set of named code-defined filters**, not a `jsonb` expression language |
| Q16 | Who owns `CONSUMER_CONTRACT.md` and which peer counterpart signs it off? | D13 | **Blocks phase 2** — a pull consumer cannot be correct without it |

---

## 5. Decision log for this change

| Decision | Rationale |
|---|---|
| Requirements state the **corrected** behaviour, with the defect id in a note | A spec that encodes a known-broken statement teaches an implementer the wrong thing; a spec that silently diverges from an ADR destroys the audit trail. Both are avoided by stating the correction and naming the defect. |
| No ADR, proposal or matrix is edited by this change | ADRs are the reasoning trail; a defect found downstream is resolved by amending the ADR (still `Proposed`) or superseding it — not by a spec rewriting it in place. |
| Nine capabilities, aligned to the six ADRs plus operations | Keeps each delta traceable to one decision record, so an amended ADR maps to a bounded set of requirements. |
| `openspec/specs/` stays empty | Nothing is deployed. Every ADR is `Proposed` and this repository contains no code. |
