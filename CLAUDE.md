# CLAUDE.md — Cross-System Sharing Concept

## What this project is

A **documentation-only** repository holding the design for exchanging domain data between
`Acme` (a DDD ASP.NET Core service on EF Core + PostgreSQL) and other internal
company services over HTTP, without a message broker.

**The design is domain-agnostic.** Nothing in it depends on what the aggregates mean. Every name
here is a placeholder: substitute your own aggregates and the design holds unchanged.

## Naming — all names are placeholders

| Placeholder | Stands for |
|---|---|
| `Acme` | the service being integrated. Assemblies are `Acme.Domain`, `Acme.Application`, `Acme.Infrastructure`, `Acme.Api`, `Acme.Integration`, `Acme.Integration.Contracts` |
| `Foo` | the primary aggregate root — the one that is published |
| `Bar` | a second aggregate root that `Foo` references. Carries the fan-out case: a `Bar` change may require re-emitting every `Foo` that references it |
| `Baz` | a third root, referenced by `Foo`, possibly hierarchical. Carries the flattening case |
| `FooAttachment`, `FooTag`, `BarUnit` | child entities inside those aggregates — the collection-projection and deterministic-ordering cases |
| `FooKind` (`Alpha`…`Zeta`), `FooStatus` (`Draft`, `Active`, `Held`, `Closed`, `Retired`, `Archived`), `FooTagKind` | enums. `Zeta` is deliberately the unmapped one; `Archived`→`RETIRED` is deliberately many-to-one |
| `peer-a`, `peer-b`, `peer-c` | external services. `peer-b`/`peer-c` exist for the multi-hop loop case |

**Value concepts are kept concrete on purpose** — `Money`, `Quantity` with a unit, date-only versus
timestamp, coordinates, contact details, postal address. They are not domain-specific: every domain
has money, units, dates and personal data, and they carry the failure modes worth designing out
(precision loss, unit ambiguity as a silent 10× error, a timezone shifting a date, truncation as a
data-quality incident, personal data leaving without a legal basis). Replacing them with `Field1`
would delete the content and keep the structure.

When adding an example, use these names. Never introduce a real-world domain noun — the moment one
appears, readers start reasoning about that domain instead of the pattern.

The deliverable is decision records and specifications that an engineer implements from —
not code. Start at [README.md](README.md).

This repository is **standalone**. It shares nothing with any other project in the workspace;
instructions from other repositories do not apply here.

## Repository structure

```
.
├── README.md                 # index + the design in six sentences
├── TECHNICAL_PROPOSAL.md     # the full design: architecture, DDL, pipelines, ops, rollout, risks
├── MAPPING_MATRIX.md         # field-by-field mapping spec and losslessness checklist
├── IMPLEMENTATION_PLAN.md    # task detail per stage (staging itself is ADR-0007)
├── STAGE_1_READ_API.md       # stage 1 implementation guide: discovery, code, tests, EF traps
├── BACKLOG.md                # epics → features → stories → tasks, sizing, traceability
├── adr/
│   └── ADR-000N-kebab-case-title.md
└── openspec/                 # the design as verifiable requirements
    ├── project.md            # context an agent needs on every change
    ├── specs/                # what is deployed (empty — nothing is)
    └── changes/<change-id>/  # proposal.md, design.md, tasks.md, specs/<capability>/spec.md
```

OpenSpec deltas are **derived** from the ADRs and the proposal, never authoritative over them. A
requirement that contradicts a document is a defect to fix in that document — record it in the
change's `design.md` and amend the source; never let a spec silently override an ADR.

**Four documents describe the same work from four angles; each owns exactly one thing.** Writing
*how* in the backlog, or *what must be true* in the guide, is how copies start drifting:

| Document | Owns |
|---|---|
| `adr/ADR-000N` | decisions — why, and what it costs |
| `BACKLOG.md` | hierarchy, value, sizing, sequencing |
| `STAGE_N_*.md` | engineering detail — file by file |
| `openspec/changes/*/specs/` | verifiable requirements — `SHALL` plus scenarios |

Backlog tasks carry a **pointer** to the guide section and the OpenSpec scenario, never a copy.

Adding a top-level document means adding a row to the README table in the same commit.

## Language

**English only** — every document, heading, table, code sample, comment and commit message.
Conversation with the user may be in any language; committed output is English.

## ADR conventions

**Filename:** `adr/ADR-000N-short-kebab-case-title.md`, zero-padded to four digits, numbered
sequentially. Never reuse a number.

**Required sections, in this order:**

```markdown
# ADR-000N — Title

- **Status:** Proposed | Accepted | Superseded by ADR-000M | Deprecated
- **Date:** YYYY-MM-DD
- **Related:** links to other ADRs
- **Open question:** (only if the decision is conditional on an unanswered question)

## Context      — the forces, constraints, and the failure mode being designed out
## Decision     — what we will do, concretely enough to implement
## Alternatives considered  — each with *why it was rejected*, not just that it was
## Consequences — split into Positive and Negative / costs accepted
```

**ADRs are append-only once Accepted.** A decision that changes gets a *new* ADR that
supersedes the old one; the old file stays, with its status updated to
`Superseded by ADR-000M`. Rewriting history in place destroys the reasoning trail, which is
the only thing an ADR is for.

**Every ADR must state what it costs.** An ADR with an empty "Negative" section has not been
thought through — every real decision trades something away. Name it.

**Alternatives must be argued, not listed.** "Rejected: too complex" is not a reason.
"Rejected: reintroduces the dual write the outbox exists to prevent" is.

## Writing standards for all documents

- **Decisions live in ADRs. Everything else references them** — never restate a decision in
  the proposal, or the two will drift. Link instead.
- **Open questions are tracked explicitly**, with an owner and a default-if-unanswered, in
  `TECHNICAL_PROPOSAL.md §2.3` and `MAPPING_MATRIX.md §7`. An unresolved question buried in
  prose is an unresolved question that ships.
- **Mark what is unknown as unknown.** Placeholder field names from the peer's schema use
  `‹angle brackets›`. Never invent a partner's field name and let it read as fact.
- **Prefer a table to a paragraph** when comparing options, listing fields, or specifying
  behaviour per case.
- **SQL and C# samples must be runnable-plausible**: correct dialect, correct EF Core / .NET
  API surface, no pseudocode dressed as code.
- Be specific about failure modes. The valuable content in these documents is the bugs that
  get designed out (dual writes, cursor gaps, silent truncation, poison-message retry loops),
  and each is worth naming where it would otherwise occur.
- No filler. If a section says nothing an implementer can act on, delete it.

## Cross-document consistency

Changing any of the following requires updating **every** place it appears, in the same commit:

| Change | Also update |
|---|---|
| A decision in an ADR | README table (if the summary shifts), TECHNICAL_PROPOSAL sections that reference it |
| Table or column names | TECHNICAL_PROPOSAL §4 DDL, and any ADR quoting that SQL |
| Envelope fields | ADR-0004 §3, TECHNICAL_PROPOSAL §5, MAPPING_MATRIX |
| Retention / delivery guarantees | ADR-0003, TECHNICAL_PROPOSAL §9, the consumer-facing contract statements |
| A new open question | TECHNICAL_PROPOSAL §2.3 and/or MAPPING_MATRIX §7, plus the relevant ADR |
| An answered open question | Remove it from the tracking table **and** fold the answer into the ADR |

## Technical stack referenced in these documents

The design targets **.NET 10 / C# 14, EF Core 10, Npgsql, PostgreSQL 13+**. When naming a
library, state its licence and prefer permissive ones (MIT / Apache-2.0). Several common
.NET libraries went commercial and are deliberately avoided — `MediatR` v13+,
`AutoMapper` v14+, `MassTransit` v9+, `FluentAssertions` v8+. If a document proposes a new
dependency, it must say which licence it carries and why the free alternative was not chosen.

## Git discipline

- Conventional commits: `docs:`, `fix:`, `chore:`, `refactor:`.
- One logical change per commit. A new ADR and an unrelated typo fix are two commits.
- The commit body explains **why** the decision changed, not which files moved.
- Work on `main` directly for documentation changes; use a branch only when the user asks for
  a pull request.
- Never rewrite pushed history.

## Scope discipline

- This repository contains **no application code**. Illustrative snippets inside documents are
  fine; a compilable project is not — it belongs in the service's own repository.
- Do not add tooling, CI, linters, or generators unless asked. The value here is the prose.
- Noticed something worth changing that is outside the current request? Add it to the relevant
  document's open-questions table instead of fixing it unasked.
