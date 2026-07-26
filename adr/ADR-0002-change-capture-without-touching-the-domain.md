# ADR-0002 — Capturing domain changes without modifying the domain model

- **Status:** Proposed
- **Date:** 2026-07-25
- **Related:** ADR-0001 (integration style), ADR-0003 (outbox storage), ADR-0004 (payload)

## Context

A hard requirement from the team: **the existing domain model must not be changed to serve
integration.** No new base classes on aggregates, no `List<IDomainEvent>` added to
`Foo`, no `IIntegrationAware` marker interfaces, no fields that exist only so an
external system can consume us. The domain expresses the business domain; the fact
that a partner service wants a JSON copy is not a business concern of `Foo`.

This is the correct instinct (it is exactly what the Anti-Corruption Layer and Published
Language patterns exist for), but it creates a concrete engineering problem: **if aggregates
do not raise events, how does the integration layer learn that something changed?**

Three sub-questions:

1. **Capture** — how do we detect that aggregate X changed, transactionally?
2. **Materialisation** — when do we build the external-format payload?
3. **Fidelity** — do we need *what happened* (business event) or *what it is now* (state)?

## Decision

### 1. Capture: an EF Core `SaveChangesInterceptor` over the `ChangeTracker`

Change detection lives entirely in the infrastructure layer. A `SaveChangesInterceptor`
inspects the `ChangeTracker` immediately before the transaction commits, resolves each
touched entity to its **owning aggregate root**, and writes one narrow row per affected
aggregate into `integration.outbox_change_log`.

```csharp
// Acme.Infrastructure.Integration.ChangeCapture
public sealed class OutboxChangeCaptureInterceptor(
    IAggregateRootResolver resolver,
    TimeProvider clock,
    IIntegrationContext context) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        var db = eventData.Context!;
        db.ChangeTracker.DetectChanges();

        var touched = db.ChangeTracker.Entries()
            .Where(e => e.State is EntityState.Added
                                or EntityState.Modified
                                or EntityState.Deleted)
            .Select(e => resolver.Resolve(e))          // entity  ->  (rootType, rootId)
            .Where(r => r is not null)
            .Select(r => r!.Value)
            .Distinct();                                // one row per aggregate, not per row

        foreach (var (rootType, rootId, changeKind) in touched)
        {
            db.Set<OutboxChangeLogEntry>().Add(new OutboxChangeLogEntry
            {
                AggregateType  = rootType,
                AggregateId    = rootId,
                ChangeKind     = changeKind,            // Created | Updated | Deleted
                OccurredAt     = clock.GetUtcNow(),
                CorrelationId  = context.CorrelationId,
                CausationId    = context.CausationId,
                TraceParent    = Activity.Current?.Id,
                ActorId        = context.ActorId,
                Status         = ChangeLogStatus.Pending
            });
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}
```

Because the interceptor adds entities to the *same* `DbContext` before the same
`SaveChangesAsync` call, capture and business state land in **one atomic commit**. No
distributed transaction, no dual write.

`IAggregateRootResolver` is a small, explicitly configured map — deliberately not magic:

```csharp
services.AddAggregateRootResolution(map =>
{
    map.Root<Foo>(p => p.Id);
    map.Owned<FooAttachment>(x => x.FooId, root: typeof(Foo));
    map.Owned<FooTag>(x => x.FooId, root: typeof(Foo));
    map.Root<Bar>(b => b.Id);
    map.Owned<BarUnit>(x => x.BarId, root: typeof(Bar));
    map.Root<Baz>(l => l.Id);
    // Anything not registered is NOT captured — and a startup check reports it.
});
```

A startup validation compares the registered map against the EF model and **fails fast** if
an entity type is neither registered as a root, registered as owned, nor explicitly
excluded. This is the mechanism that prevents "we forgot to publish changes to the new
table" — the classic silent integration bug.

### 2. Materialisation: deferred, out of band (two-stage outbox)

The interceptor writes only *"aggregate `Foo/{guid}` changed at T"* — roughly 200 bytes.
It does **not** build the external JSON. A separate `MaterializerWorker` picks up pending
change-log rows, loads the full aggregate through a dedicated read query, runs it through
the Anti-Corruption Layer mapper (ADR-0004), and writes the finished contract payload into
`integration.outbox_message`.

Why not serialise inside the interceptor:

- **No N+1 inside `SaveChanges`.** Building a `Foo` contract needs `Bar`,
  `Baz`, attachments, tags. Loading those from inside `SavingChangesAsync` triggers
  queries on a connection that is mid-transaction — slow, and a deadlock generator under load.
- **Business latency stays untouched.** Mapping cost is moved off the user's request.
- **Compaction becomes possible.** Twenty edits to one property in a minute collapse into
  one outbound message (ADR-0003 §compaction). With inline serialisation you would emit twenty.
- **Contract v2 can be re-materialised from current state** without a database migration of
  historical payloads. This is a large, under-appreciated win: when the peer changes their
  schema (and they will — ADR-0004), we re-run the materializer, not a data migration.
- **A mapping bug is recoverable.** The change log still holds the fact that something
  changed; fix the mapper, reset the rows to `Pending`, re-materialise. If the bad JSON had
  been written transactionally at commit time, the correct payload would be unrecoverable
  for anything already delivered.

### 3. Fidelity: state transfer is the default; business events are opt-in

Deferred materialisation means the payload reflects the aggregate's state **at
materialisation time**, not at change time. If a property is edited twice within the
materialisation window, the consumer sees the final state once. We accept this and state it
explicitly in the contract:

> **Semantics: convergent state transfer, latest-wins.** Each message carries the full
> current state and a monotonically increasing `aggregateVersion`. Consumers apply a message
> only if its `aggregateVersion` is greater than the last one they applied. Intermediate
> states may be skipped. The consumer's converged state is always correct.

This is a genuine trade-off, so it is bounded:

| Need | Mechanism |
|---|---|
| "What does the property look like now?" | Default state-transfer message. Covers ~95 % of consumer needs. |
| "Tell me every price change, with old and new value" | **Opt-in explicit integration event.** Requires the Application layer to emit an `IntegrationEventRequest` in the same transaction — see below. |
| "I need an audit trail" | Out of scope for integration; use the existing audit facility. |

For the small number of cases where the *transition* is the payload (`PriceChanged`,
`Deactivated`, `Reassigned`), the **Application layer** — not the Domain — enqueues
an explicit event:

```csharp
// Acme.Application.Properties.ChangePrice
public async Task<Result> HandleAsync(ChangePriceCommand cmd, CancellationToken ct)
{
    var property = await _properties.GetByIdAsync(cmd.FooId, ct);
    if (property is null) return Result.Failure("Foo not found.");

    var previous = property.Price;
    var result = property.ChangePrice(cmd.NewPrice);      // domain unchanged
    if (result.IsFailure) return result;

    _integrationEvents.Enqueue(new PriceChangedIntegrationEvent(   // application concern
        property.Id, previous, cmd.NewPrice, _clock.GetUtcNow()));

    await _uow.SaveChangesAsync(ct);                       // one transaction, still atomic
    return Result.Success();
}
```

`IIntegrationEventEnqueuer` writes into the same change log with `ChangeKind = Event` and a
pre-built payload. **The domain still does not know integration exists.** The application
layer — whose job is orchestrating use cases — decides that this use case is externally
interesting.

## Alternatives considered

### A. Add domain events to aggregates (`AggregateRoot.RaiseEvent(...)`)

*Rejected — violates the stated constraint.* It is the textbook approach and it does give
perfect event fidelity, but it requires editing every aggregate, adding a base class, and
mixing an infrastructure concern into the domain. It also does not solve materialisation:
you still need to load related data to build a `Foo` contract, so you still end up with
a deferred stage. Revisit only if state-transfer semantics prove insufficient for many
consumers.

### B. PostgreSQL triggers writing to the outbox table

*Rejected.* Truly zero-touch on C# code, and transactional by construction. But: business
context (correlation id, actor, trace parent, "why did this change") is not available in a
trigger; logic lives in SQL outside the application's test suite and code review flow;
EF Core migrations and hand-written trigger DDL drift apart; and debugging a production
mapping issue in PL/pgSQL is markedly worse than in C#. The one place we *will* use a
trigger-like guarantee is the `xid8` visibility watermark (ADR-0003), which is data, not logic.

### C. Debezium / logical decoding

*Rejected for now* — see ADR-0001 §Alternative B. Same reasoning: capture is the cheap part.

### D. Periodic diffing (`updated_at > lastRun`)

*Rejected.* Requires a reliable `updated_at` on every table (we do not have one universally),
misses deletes entirely, cannot distinguish create from update, provides no correlation
context, and races with long transactions in exactly the way ADR-0003's watermark exists to
prevent. It is the approach that looks simplest on day one and produces the most
"why is this record missing" tickets on day ninety.

### E. Serialise the full payload inside the interceptor (single-stage outbox)

*Rejected as default*, for the five reasons in §2. Note this is what most outbox tutorials
show, because they assume the aggregate is already fully loaded and the payload is small.
Ours is neither.

## Consequences

### Positive

- **Domain assemblies are untouched.** Verifiable with an architecture test.
- **Capture is cheap** (~200-byte insert, no serialisation) and inside the business transaction.
- **Compaction and re-materialisation** are available — both are significant operationally.
- **Adding a new aggregate to the integration is a one-line registration** plus a mapper.
- **Forgetting to register an entity is a startup failure, not a silent data gap.**

### Negative / costs accepted

- **No intermediate-state fidelity by default.** Documented in the contract; opt-in events
  cover the exceptions.
- **A second read of the aggregate** per change (materialisation). Cost is real but off the
  request path; mitigated by compaction and by batching the load (`WHERE id = ANY(@ids)`).
- **Deletes need care.** A hard-deleted aggregate cannot be re-read at materialisation time.
  Mitigation: for `ChangeKind = Deleted` the interceptor captures the *original values* from
  the `ChangeTracker` into the change-log row (`snapshot_hint jsonb`), which is enough to emit
  a tombstone `{id, deletedAt}`. Soft deletes need no special handling.
- **`ChangeTracker` sees only EF-tracked writes.** `ExecuteUpdateAsync` / `ExecuteDeleteAsync`
  and raw SQL **bypass the interceptor entirely.** This is the sharpest edge in this ADR.
  Mitigations: (a) a Roslyn analyzer / banned-symbols entry flags `ExecuteUpdateAsync` and
  `ExecuteDeleteAsync` on integrated entity types, requiring an explicit
  `[BypassesIntegrationCapture(reason)]` attribute; (b) any deliberate bulk mutation must
  enqueue a corresponding backfill range (ADR-0006) in the same transaction; (c) the nightly
  reconciliation job (proposal §9.4) catches anything that slipped through.
