# Deployed specs

Empty on purpose.

`openspec/specs/` holds the behaviour that is **deployed**. Nothing in this design is deployed:
every ADR is still `Proposed` and the repository contains no application code.

Requirements live in `openspec/changes/<change-id>/specs/<capability>/spec.md` until the change
is archived, at which point its deltas merge into this directory.

Current changes, in delivery order (ADR-0007):

| Change | Stage | Covers |
|---|---|---|
| [`add-foo-read-api`](../changes/add-foo-read-api/proposal.md) | 1 | On-demand contract-shaped `GET` over `Foo`. No new tables. |
| [`add-cross-system-integration-layer`](../changes/add-cross-system-integration-layer/proposal.md) | 2–7 | Capture, materialisation, feed, push, inbound, backfill, hardening. |
