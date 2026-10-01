# SharedKanban

A native SwiftUI Kanban app for iPhone and iPad, built for a household (two people first, small groups second). Projects, columns and cards are shared in real time; the API, PostgreSQL database, attachment files, background worker and a Claude connector (MCP) all run on a Mac mini. Apple services are used only for Sign in with Apple and push notifications.

## Status

Phase 0 (product and architecture) is complete and waiting for the owner's review. No application code exists yet. The phase review lives at `docs/reviews/phase-0-review.md`.

## Start here

| If you are | Read |
|---|---|
| The owner, deciding whether to approve Phase 0 | `docs/reviews/phase-0-review.md`, then `docs/human-checklist.md` |
| A Claude session (cloud or Mac mini) about to do work | `CLAUDE.md`, then the documents it lists under "Read first" |
| Curious about the product | `docs/product.md` |
| Curious about how it is built | `docs/architecture.md` and `docs/decisions/decision-register.md` |

## Documents

- `docs/product.md` — purpose, worked examples, user stories, acceptance criteria, manual scenarios.
- `docs/architecture.md` — components, data model, sequence diagrams, concurrency model, Mac mini topology, where work runs.
- `docs/design-system.md` — iPhone and iPad layouts, board and card specification, interaction states, accessibility, copy.
- `docs/threat-model.md` — assets, actors, trust boundaries, threat tables, phase security requirements, incident playbooks.
- `docs/api.md` — REST, WebSocket and MCP contracts, error codes, the move contract, event catalog.
- `docs/decisions/decision-register.md` — the canonical decisions D01 to D22; `docs/decisions/README.md` indexes the ADRs.
- `docs/operator-playbook.md` — who does what per phase: cloud Claude, Mac mini Cowork sessions, or the human; ready-to-paste Cowork prompts.
- `docs/human-checklist.md` — the only list the owner must act on personally, step by step.
- `docs/reviews/` — one review per phase, plus Mac mini results.
- `docs/source-brief.md` — the original brief this project was planned from.

## Planned layout

```text
CLAUDE.md
docs/        product, architecture, design system, threat model, API, decisions, playbook, checklist, reviews
ios/         SwiftUI app and tests (Phase 1)
shared/      SharedDTOs package: transport types, rank generator, templates, error codes (Phase 1)
server/      Vapor API, worker, MCP adapter, migrations, tests (Phase 1)
deploy/      Docker Compose, backup and restore scripts, runbook (Phase 1 and Phase 10)
```
