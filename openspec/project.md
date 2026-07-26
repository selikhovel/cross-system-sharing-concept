# Project — Cross-System Sharing Concept

## Purpose

Design documentation for exchanging domain data (`Foo`, `Bar`, `Baz`) between
**Acme** — a DDD ASP.NET Core service on EF Core + PostgreSQL — and other internal
company services over HTTP, **without a message broker**.

This repository ships **prose, not code**. The deliverable is decision records and
specifications an engineer implements from, in the service's own repository. OpenSpec artifacts
here describe the behaviour the implementation must exhibit; they do not describe files in this
repository.

## Domain independence and naming

**Every requirement in these specs is domain-agnostic.** A requirement that only holds for a
particular subject matter is a requirement written wrong — the design is selected by structure
(transactional change, an external observer, a peer-owned wire format, no broker), not by what the
data means.

All names are placeholders: `Acme` is the service, `Foo` the published aggregate root, `Bar` and
`Baz` aggregates it references (present so the fan-out and flattening cases exist),
`FooAttachment`/`FooTag`/`BarUnit` their child entities, `peer-a`/`peer-b`/`peer-c` external
services. Full table in [CLAUDE.md](../CLAUDE.md#naming--all-names-are-placeholders).

Value concepts stay concrete — money, quantities with units, date-only versus timestamp,
coordinates, contact details — because the scenarios that matter are about them: precision loss,
unit ambiguity, a shifted date, a truncated string, personal data with no legal basis. When writing
a new scenario, reach for one of those rather than inventing a domain noun.

## Source of truth

| Artefact | Role |
|---|---|
| `adr/ADR-000N-*.md` | **Decisions.** Append-only once Accepted; a changed decision gets a new ADR that supersedes the old one. |
| `TECHNICAL_PROPOSAL.md` | The design: architecture, DDL, pipelines, security, ops, capacity, rollout, risks. References ADRs, never restates them. |
| `MAPPING_MATRIX.md` | Field-by-field mapping spec and the losslessness checklist. |
| `IMPLEMENTATION_PLAN.md` | Day-by-day build order. |
| `openspec/` | Requirements derived from the above, in verifiable `SHALL` + scenario form. |

**Precedence:** an ADR outranks the proposal; the proposal outranks OpenSpec deltas. Where an
OpenSpec requirement contradicts a document, it is flagged in the change's `design.md` as a
defect to resolve in the source document — OpenSpec never silently overrides an ADR.

## Target stack (of the system being designed)

- .NET 10 / C# 14, EF Core 10, Npgsql, **PostgreSQL 13+** (`xid8` / `pg_snapshot_xmin` are a
  hard prerequisite — see ADR-0003 §5).
- Workers are `BackgroundService`; queue claiming is `FOR UPDATE … SKIP LOCKED`. No broker,
  no Redis, no leader election, no job framework.
- Mapping is compile-time (`Mapperly`); resilience is `Microsoft.Extensions.Http.Resilience`
  (Polly v8); telemetry is OpenTelemetry.

## Conventions

- **Licensing is a design constraint.** Prefer MIT / Apache-2.0. Deliberately avoided because
  they went commercial: `MediatR` v13+, `AutoMapper` v14+, `MassTransit` v9+,
  `FluentAssertions` v8+ (pin v7). A proposed dependency must state its licence and why the
  free alternative was rejected.
- **English only** in every committed document, heading, table, sample and commit message.
- **Unknowns are marked unknown.** Placeholder fields from the peer's schema use
  `‹angle brackets›`. Never invent a partner field name and let it read as fact.
- **Every decision states its cost.** An empty "Negative" section means the decision has not
  been thought through.
- Prefer a table to a paragraph when comparing options or specifying behaviour per case.
- SQL and C# samples must be runnable-plausible: correct dialect, correct API surface, no
  pseudocode dressed as code.
- Conventional commits (`docs:`, `fix:`, `chore:`, `refactor:`); one logical change per commit;
  the body explains **why**. Work on `main` for documentation changes.

## Cross-document consistency

Changing any of these requires updating **every** place it appears, in the same commit:

| Change | Also update |
|---|---|
| A decision in an ADR | README table, TECHNICAL_PROPOSAL sections referencing it, affected OpenSpec deltas |
| Table or column names | TECHNICAL_PROPOSAL §4 DDL, any ADR quoting that SQL, affected deltas |
| Envelope fields | ADR-0004 §3, TECHNICAL_PROPOSAL §5, MAPPING_MATRIX, `contract-mapping` delta |
| Retention / delivery guarantees | ADR-0003, TECHNICAL_PROPOSAL §9, consumer-facing contract statements |
| A new open question | TECHNICAL_PROPOSAL §2.3 and/or MAPPING_MATRIX §7, plus the relevant ADR |
| An answered open question | Remove it from the tracking table **and** fold the answer into the ADR |

## Scope discipline

No application code, no tooling, no CI, no generators unless asked. Something worth changing
that is outside the current request goes into the relevant document's open-questions table
(TECHNICAL_PROPOSAL §2.3 / MAPPING_MATRIX §7), not into an unasked fix.
