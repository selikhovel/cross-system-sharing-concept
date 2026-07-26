# Change feed (pull)

Derived from ADR-0003 §5, `TECHNICAL_PROPOSAL.md` §5.5 and ADR-0001 §1.

## ADDED Requirements

### Requirement: The feed cannot skip a row

The change feed SHALL expose only rows whose inserting transaction is guaranteed complete, by
filtering on the transaction-id visibility watermark
(`insert_xid < pg_snapshot_xmin(pg_current_snapshot())`), so that a consumer advancing an opaque
monotonic cursor can never step over a row that becomes visible later.

This filter SHALL be treated as a release gate, verified by an explicit test.

> This is the design's single most important line of SQL: `sequence` is assigned at insert time
> but rows become visible at commit time, so a slow transaction holding a lower sequence is
> invisible to a consumer that has already advanced past it — permanent, silent loss that ordinary
> testing does not reproduce.

#### Scenario: Slow transaction holds a lower sequence

- **WHEN** transaction A inserts sequence N and stays open
- **AND** transaction B inserts sequence N+1 and commits
- **AND** a consumer reads the feed from a cursor below N
- **THEN** the feed returns neither N nor N+1

#### Scenario: Slow transaction commits

- **WHEN** transaction A then commits
- **THEN** the same request returns both N and N+1, in sequence order

#### Scenario: Aborted transaction

- **WHEN** transaction A rolls back instead
- **THEN** the feed returns N+1 and never returns N, and the cursor advances without a gap claim

### Requirement: Cursor semantics are owned by the consumer

The feed SHALL take an opaque cursor and return the next cursor plus a more-available flag, and
SHALL store no per-consumer state.

The page size SHALL be capped, reads SHALL be non-tracking, and the feed MAY be served from a read
replica.

#### Scenario: Pagination across a large backlog

- **WHEN** a consumer walks 10 000 messages at the default page size
- **THEN** every message is returned exactly once, in sequence order, with no duplicate and no gap

#### Scenario: Consumer resumes after a restart

- **WHEN** a consumer resumes from its stored cursor
- **THEN** it receives only messages after that cursor

#### Scenario: Oversized page requested

- **WHEN** a consumer requests a limit above the cap
- **THEN** the response is capped at the maximum and the cap is discoverable in the contract

### Requirement: A replica-served feed preserves the watermark guarantee

If the feed is served from a read replica, the watermark guarantee SHALL be verified against that
replica, not only against the primary.

> Untested assumption. `TECHNICAL_PROPOSAL.md` §5.5 permits replica serving and
> ADR-0006 permits it for backfill, but the watermark argument is stated only in terms of the
> writing node's snapshot. Replica visibility follows WAL replay of commit records, so the
> guarantee is expected to hold — expected is not verified, and this is the one guarantee whose
> failure is silent.

#### Scenario: Watermark test against a replica

- **WHEN** the watermark test runs with the feed served from a streaming replica under replication
  lag
- **THEN** it produces the same result as against the primary

### Requirement: Deletions appear as tombstones, never as absences

The feed SHALL represent a deleted or retracted aggregate as a message with
`changeKind = 'Deleted'`, retained for the full retention window.

#### Scenario: Deleted aggregate in the feed

- **WHEN** an aggregate was hard-deleted within the retention window
- **THEN** a consumer walking the feed receives a tombstone message for it

#### Scenario: Consumer bootstrapped from the feed alone

- **WHEN** a consumer walks the whole retained feed
- **THEN** it can distinguish "never existed", "exists" and "deleted" for every aggregate the feed
  mentions

### Requirement: An expired cursor fails loudly

When a cursor predates the retained window, the feed SHALL answer `410 Gone` with a machine-readable
reason and the required remediation, and SHALL NOT return a silently incomplete page.

#### Scenario: Consumer offline longer than retention

- **WHEN** a consumer presents a cursor older than the oldest retained message
- **THEN** the response is `410` with a cursor-expired code and a backfill action
- **AND** no partial data is returned

#### Scenario: Consumer just inside the window

- **WHEN** a consumer presents a cursor at the boundary of the retained window
- **THEN** the feed serves it normally

### Requirement: The feed is one read path over one log

The feed SHALL read the same `outbox_message` rows the dispatcher delivers, with no separate
projection, no second serialisation path and no independent source of truth.

#### Scenario: Payload equality between paths

- **WHEN** the same message is delivered by push and read from the feed
- **THEN** the bytes are identical for the same subscriber's contract version and payload variant

#### Scenario: Subscriber-scoped content

- **WHEN** a subscriber without a PII grant reads the feed
- **THEN** it receives its redacted payload variant, not the full one
