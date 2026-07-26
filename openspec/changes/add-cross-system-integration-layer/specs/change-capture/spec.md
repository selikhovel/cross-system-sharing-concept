# Change capture

Derived from ADR-0002 and `TECHNICAL_PROPOSAL.md` §5.1.

## ADDED Requirements

### Requirement: Transactional change capture

The system SHALL record every change to an integrated aggregate into
`integration.outbox_change_log` within the same database transaction as the business write, so
that business state can never be committed without its integration record.

The capture row SHALL be narrow — aggregate type, aggregate id, change kind, timestamps and
correlation context — and SHALL NOT contain a serialised contract payload.

#### Scenario: Business transaction commits

- **WHEN** a command handler modifies a `Foo` and the transaction commits
- **THEN** exactly one `outbox_change_log` row exists for that `(aggregate_type, aggregate_id)`
- **AND** it carries `change_kind = 'Updated'`, `occurred_at`, `correlation_id`, `causation_id`,
  `trace_parent` and `actor_id` from the ambient request context

#### Scenario: Business transaction rolls back

- **WHEN** a command handler modifies a `Foo` and the transaction rolls back
- **THEN** no `outbox_change_log` row exists for that aggregate

#### Scenario: Several entity rows of one aggregate change in one transaction

- **WHEN** a single transaction modifies a `Foo`, adds two `FooAttachment` rows and removes
  one `FooTag` row
- **THEN** exactly one `outbox_change_log` row is written, for the `Foo` root

#### Scenario: Capture is not on the payload-building path

- **WHEN** a change is captured
- **THEN** no aggregate is loaded, no related entity is queried, and no contract is serialised
  inside `SavingChangesAsync`

### Requirement: Aggregate root resolution is explicit and validated at startup

The system SHALL resolve each tracked entity to its owning aggregate root through an explicitly
registered map, and SHALL fail application startup if any entity type in the EF model is neither
registered as a root, nor registered as owned by a root, nor explicitly excluded.

#### Scenario: Owned entity resolves to its root

- **WHEN** a `BarUnit` is modified
- **THEN** the capture row names `aggregate_type = 'Bar'` and the parent Bar's id

#### Scenario: A new entity type is added without registration

- **WHEN** an entity type is added to the EF model and is not registered or excluded
- **THEN** application startup fails, naming the unregistered type

### Requirement: Deleted aggregates are captured from original values

The system SHALL capture the original values of a deleted aggregate into
`outbox_change_log.snapshot_hint` at capture time, because the row no longer exists when
materialisation runs.

#### Scenario: Hard delete

- **WHEN** a `Foo` is hard-deleted
- **THEN** the capture row carries `change_kind = 'Deleted'` and a `snapshot_hint` sufficient to
  emit a tombstone without reading the domain table

#### Scenario: Soft delete

- **WHEN** a `Foo` is soft-deleted by setting its deletion flag
- **THEN** it is captured as `change_kind = 'Updated'` and the tombstone decision is taken later
  by the materialisation filter rules

### Requirement: Writes that bypass the change tracker are blocked or compensated

The system SHALL prevent silent data gaps caused by `ExecuteUpdateAsync`, `ExecuteDeleteAsync`
and raw SQL against integrated entity types, which never reach the `SaveChangesInterceptor`.

#### Scenario: Unannotated bulk mutation

- **WHEN** code calls `ExecuteUpdateAsync` on an integrated entity type without an explicit
  bypass annotation
- **THEN** the build fails through the banned-symbols or analyzer rule

#### Scenario: Deliberate bulk mutation

- **WHEN** a bulk mutation is annotated with an explicit, reasoned bypass
- **THEN** it enqueues a corresponding backfill range for the affected aggregates in the same
  transaction

### Requirement: Business transitions are opt-in and enqueued by the application layer

The system SHALL support explicit integration events for the cases where the transition, not the
resulting state, is the payload — and SHALL keep the domain model unaware of integration.

#### Scenario: Price change emits a transition event

- **WHEN** the `ChangePrice` application handler enqueues a `PriceChangedIntegrationEvent` and
  saves
- **THEN** one capture row with `change_kind = 'Event'` and a pre-built payload in
  `snapshot_hint` is written in the same transaction as the domain change

#### Scenario: The domain stays clean

- **WHEN** the architecture tests run
- **THEN** no type in `Acme.Domain` references the integration assembly, the contracts
  assembly or `System.Net.Http`
- **AND** no `ICommandHandler<,>` implementation resolves `IHttpClientFactory`

### Requirement: Capture records inbound provenance

The system SHALL record, on every capture row, the provenance of the change: the external system
whose inbound message caused it, if any, and the ordered propagation path the change has already
traversed. Locally originated changes SHALL carry empty provenance.

> Defect **D6**. `TECHNICAL_PROPOSAL.md` §6.2 requires echo suppression based on
> `causationSource`, which exists in no table and in no envelope field. Suppression by immediate
> source is also insufficient for loops longer than one hop, so the propagation path — not a
> single source id — is what must be recorded.

#### Scenario: Change caused by an inbound message

- **WHEN** the inbox worker applies a message from `peer-a` through an application command
  handler
- **THEN** the resulting capture row records `peer-a` as the causing source and appends it
  to the propagation path

#### Scenario: Change caused by a user

- **WHEN** a user edits a `Foo` through the API
- **THEN** the capture row's causing source is null and its propagation path is empty
