# Deployed specs

Empty on purpose.

`openspec/specs/` holds the behaviour that is **deployed**. Nothing in this design is deployed:
every ADR is still `Proposed` and the repository contains no application code.

Requirements live in `openspec/changes/<change-id>/specs/<capability>/spec.md` until the change
is archived, at which point its deltas merge into this directory.

Current changes, in delivery order (ADR-0007):

| Change | Scope | Covers |
|---|---|---|
| [`add-foo-snapshot-projection`](../changes/add-foo-snapshot-projection/proposal.md) | Stage 1, **slice 1** | The snapshot read model and the one canonical projection, asserted in memory. **Self-contained** — implementable from its own folder, without the rest of this repository |
| [`add-foo-read-api`](../changes/add-foo-read-api/proposal.md) | Stage 1 | The whole read API: on-demand contract-shaped `GET` over `Foo`, no new tables. Slice 1 is split out above |
| [`add-cross-system-integration-layer`](../changes/add-cross-system-integration-layer/proposal.md) | Stages 2–7 | Capture, materialisation, feed, push, inbound, backfill, hardening |

`add-foo-snapshot-projection` is deliberately written to be handed to someone — or something — with
access to the service codebase and nothing else: it carries its own placeholder-substitution table,
discovery steps, design decisions and dependency list rather than pointing at documents outside the
folder.
