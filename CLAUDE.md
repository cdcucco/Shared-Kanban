# SharedKanban project rules

This file is read at the start of every session, so it stays short. The documents under `docs/` hold the detail and win whenever this file is vaguer than they are. The decision register (`docs/decisions/decision-register.md`, D01 to D22) is law for all code and documents; see "Decision register is law" below.

## Goal
Build a native SwiftUI iOS/iPadOS shared Kanban app. The backend, PostgreSQL database, files, worker, and MCP adapter run on a Mac mini. Apple services are limited to Sign in with Apple and APNs; secure ingress may use a tunnel.

## Product invariants
- A project has exactly 3 or 4 active columns in release 1.
- Built-in templates are editable and reorderable.
- A task requires a title and may contain notes, assignee, due date, attachments, and comments.
- Every API and MCP operation enforces project membership and role permissions.
- The server is authoritative; the app supports cached reads and queued offline mutations.
- No AI component receives raw database or filesystem access.
- Destructive and membership-changing MCP actions require explicit confirmation.

## Engineering rules
- Use SwiftUI, Swift concurrency, Codable DTOs, SwiftData caching, URLSession, AuthenticationServices, UserNotifications, Vapor, Fluent, and PostgreSQL.
- Prefer Apple frameworks and small, justified dependencies.
- Do not place secrets, signing material, tokens, or production URLs in source control.
- Add migrations; never modify an applied migration.
- Use idempotency keys for client mutations.
- Use optimistic concurrency and database transactions for moves and membership changes.
- Keep business logic in server application services shared by REST and MCP adapters.
- Keep views small and accessible. Supply non-drag alternatives for every drag action.

## Required workflow
1. Read relevant docs and existing tests.
2. State the plan and files to change before editing.
3. Implement the smallest coherent slice.
4. Add or update tests.
5. Run targeted tests, then the appropriate full build/test command.
6. Report commands, results, remaining risks, and manual checks.
7. Stop for review at the end of the requested phase.

## Definition of done
- Code builds with no new warnings.
- Tests cover success, authorization failure, invalid input, conflict, and retry paths.
- User-visible errors are understandable.
- Accessibility labels and Dynamic Type are considered.
- Docs and API contracts match implementation.

## Read first
Read in this order. Every session reads items 1, 6 and 9; the rest as the line says. Paths are repo-relative; a document that does not exist yet is still being written in Phase 0.

1. `docs/product.md` — user stories, non-goals, acceptance criteria; read before any feature or UI work.
2. `docs/architecture.md` — components, sequence diagrams, resolved versions (recorded at Phase 1); read before touching server, sync, realtime or deploy code.
3. `docs/design-system.md` — iPhone/iPad layouts, interaction states, accessibility rules, card hierarchy; read before any SwiftUI work (Phases 4, 6, 11).
4. `docs/threat-model.md` — authentication, authorization, invitations, attachments, MCP, ingress, backups, deletion; read before any security-relevant change and at every phase review.
5. `docs/api.md` — resource contracts, error envelope, WebSocket messages, fixture policy; read before adding or changing any endpoint, DTO or event.
6. `docs/decisions/decision-register.md` — the canonical decisions D01 to D22; read the sections relevant to the task every session, the whole file at a phase boundary.
7. `docs/decisions/*.md` — ADR-0001 to ADR-0005 expand D11 to D15; read the ADR for the subsystem you are changing.
8. `docs/operator-playbook.md` — who does what (cloud session, Mac mini Cowork, human) and how to hand work over; read before delegating or asking the owner for anything.
9. `docs/human-checklist.md` — the owner's one-time numbered steps (D20); read before asking the human for anything so you never ask for a step already listed or done.
10. `docs/reviews/` — phase reviews and Mac mini build/test results; read the latest review before starting a phase and the Mac mini results before claiming iOS work is verified.
11. `docs/source-brief.md` — the owner's original brief and the Prompt 0 to 11 sequence; read when a phase starts or when the register and a document disagree on intent (the register still wins).

## Where work happens
- Cloud Linux session (usually this one): writes all code, docs, DTOs, migrations and deploy files; builds and tests `shared/` and `server/` with `swift build` and `swift test` against a local PostgreSQL; never compiles iOS code (D20).
- Mac mini Cowork session: builds and tests the iOS app with `xcodebuild` at the D02 widths, signs and uploads to TestFlight, runs Docker Compose, Tailscale, restic, launchd and restore drills, and writes results under `docs/reviews/` (D20, D21).
- Human (the owner): only the nine-item one-time checklist in `docs/human-checklist.md` (Apple ID and Developer Program, portal keys, device trust, Tailscale admin clicks, admin passwords, UPS and SSD, password manager, optional B2) plus reading each phase review and saying "go" (D20).
- Escalation rule, because the owner wants minimal involvement: first try to do the thing yourself in this session; if it needs macOS, a device, Xcode or the Mac mini host, write a self-contained Mac mini Cowork prompt; only when a step needs the owner's identity, money, password or physical hands, ask the human with short numbered steps (what to tap, in order, nothing to transcribe).
- The full hand-over format, what each environment can and cannot run, and how results come back are in `docs/operator-playbook.md`.

## Phase gates
Work happens in vertical slices, one phase per prompt (`docs/source-brief.md`, "Prompt sequence"). Each phase ends with a phase review written to `docs/reviews/phase-NN-review.md` (commands run, results, deviations from the register, risks, manual checks) and a stop for the owner. A phase is not done because code compiles; it is done when its exit gate passes and the review is written. Mac mini results for a phase go to `docs/reviews/phase-NN-macmini.md`.

| Phase | Deliverable | Exit gate | Status |
|---|---|---|---|
| 0 | Product spec, architecture, design system, threat model, API doc, register, ADRs, playbook, checklist | Decisions reviewed by the owner; no application code | Complete, pending owner review |
| 1 | Monorepo, SharedDTOs and server packages, Xcode project, Compose dev services, test commands, versions recorded (D11, D20) | Empty app and API build; CI passes; human checklist steps 1 to 3 started | Next |
| 2 | Migrations, models, application services, idempotency, ranking, activity log (D05, D06, D17, D18, D19) | Unit and PostgreSQL integration tests for permissions and ordering pass | |
| 3 | Sign in with Apple, sessions, account deletion (D09, D16) | Real-device sign-in works; token and deletion tests pass | |
| 4 | Native design shell against the in-memory repository, sample data (D01, D02, D12) | iPhone/iPad layouts reviewed at the D02 widths | |
| 5 | API client, SwiftData cache, outbox, snapshot and event sync (D08, D12, D22) | Two devices create and edit shared data; replay and conflict tests pass | |
| 6 | Drag and drop, WebSocket realtime, conflict handling (D01, D06, D13) | Concurrent-move and reconnect tests pass on a real device | |
| 7 | Invitations, roles, comments, activity feed (D03, D04) | Permission matrix and invitation lifecycle tests pass | |
| 8 | Due dates, attachments, notifications (D07, D14) | Background upload and APNs sandbox tests pass | |
| 9 | MCP adapter, proposals, confirmation, audit, Connect Claude screen (D15); 9b remote OAuth only on demand | Read, propose, confirm, deny and audit tests pass | |
| 10 | Mac mini deployment, ingress profiles, backups, launchd escape hatch (D10, D11, D21) | Restore drill succeeds; remote HTTPS works over Tailscale | |
| 11 | Accessibility, security, performance, TestFlight beta | TestFlight checklist and release criteria pass | |

Stop at every gate. Never start the next phase in the same session without the owner's approval of the review.

## Repository layout
Intended tree (from the brief, adjusted by D11, D21 and D22). Lines marked "(Phase N)" do not exist yet and are created in that phase; everything else exists today or is being written in Phase 0.

```text
Shared-Kanban/
  CLAUDE.md                      this file
  README.md
  docs/
    product.md
    architecture.md              (Phase 0; versions filled in at Phase 1)
    design-system.md
    threat-model.md              (Phase 0)
    api.md                       (Phase 0)
    operator-playbook.md
    human-checklist.md           (Phase 0)
    source-brief.md
    decisions/
      decision-register.md
      0001-vapor-fluent-postgresql.md
      0002-swiftdata-as-cache.md
      0003-websockets-realtime.md
      0004-local-attachment-storage.md
      0005-mcp-proposal-confirmation.md
    reviews/                     (Phase 0 review first; one file per phase and per Mac mini run)
  ios/                           (Phase 1)
    SharedKanban.xcodeproj
    SharedKanban/                app target; @Model classes live only here (D12)
    SharedKanbanTests/
    SharedKanbanUITests/
    Config/Environment.example.xcconfig   the real xcconfig is git-ignored (D10)
  shared/                        (Phase 1)
    Package.swift                SharedDTOs: Codable DTOs, Patch<T>, rank generator, templates, error codes, coder (D06, D19, D20, D22)
    Sources/SharedDTOs/
    Fixtures/                    JSON fixtures decoded by contract tests on Linux and iOS (D22)
  server/                        (Phase 1)
    Package.swift                Vapor 4, Fluent 4, PostgreSQL 17 (D11)
    Sources/App/                 services, authorizer, migrations, REST, WebSocket hub, MCP layer
    Sources/Run/                 one binary: serve, worker, migrate, mcp-stdio, mcp-token, admin (D11)
    Tests/AppTests/
  deploy/                        (Phase 1 dev Compose; Phase 10 the rest)
    compose.yaml                 api, worker, postgres, one-shot migrate (D11)
    .env.example                 the real .env lives outside the repository (D11)
    launchd/                     escape-hatch plists and build.sh (D11)
    backup/                      backup.sh, restore.sh, health.sh, export-secrets.sh (D21)
    runbook.md                   (Phase 10)
```

## Commands
Not yet; Phase 1 adds scripts and documents exact commands here. When they exist, one CI script runs in both environments with "requires macOS" markers (D20), and this section lists the format, build, migrate, server-test and iOS-test commands verbatim.

## Git
- `main` is the integration branch and always holds a passing build. Work on short-lived branches named `phase-NN/<topic>` (for example `phase-01/scaffold`), `docs/<topic>` for documentation only, or `fix/<topic>`; when the environment has already assigned a branch, keep it and put the phase and topic in the PR title.
- Small commits with descriptive messages in the form `<area>: <imperative summary>` where area is `docs`, `ios`, `server`, `shared`, `deploy` or `ci`; one coherent slice per commit, never a "WIP" dump. Commit only when asked or when the workflow says to; never force-push a shared branch.
- Never commit secrets: no `.env`, `.p8` keys, App Store Connect API keys, restic passwords, Claude connection tokens, device tokens, or any hostname of the Mac mini, tailnet or tunnel. Commit `.env.example` and `Environment.example.xcconfig` only (D10, D11, D21). If a secret lands in history, stop and report it; do not rewrite history silently.
- Never modify an applied migration. Add a new migration; migrations run only through the explicit `migrate` subcommand, never automatically (D11).
- Update docs in the same commit as a contract change: an endpoint, DTO, event, error code or WebSocket message change updates `docs/api.md` and its fixture under `shared/Fixtures/` together with the code (D22); a behavior change that touches a decision updates the register and ADR in the same PR.
- A phase ends with a commit that adds `docs/reviews/phase-NN-review.md`; the Mac mini session commits its results as `docs/reviews/phase-NN-macmini.md`.

## Decision register is law
- Code and documents must follow `docs/decisions/decision-register.md` (D01 to D22). ADR-0001 to ADR-0005 expand D11 to D15 and carry the same authority. Where this file and the register differ, the register wins.
- To change a decision: add a new ADR `docs/decisions/NNNN-<slug>.md` (next number; Status, Context, Decision, Consequences, superseded IDs), update the register's summary row and section, and update every affected document, all in the same PR. Never edit a decision silently and never implement a contradiction "for now".
- Deviations from the brief that the register already made (invitation token in the body, attachment `/complete` folded into the PUT, `DELETE /v1/account` as `POST /v1/account/delete`, `DELETE /v1/columns/{id}` archives, due reminders local-only, no App Attest in release 1) are decided; do not revert them to the brief's shape (D04, D07, D09, D14, D16, D22).
- Versions fixed by the register: iOS/iPadOS 18.0 minimum, Swift 6 language mode with strict concurrency, Vapor 4, Fluent 4, PostgreSQL 17, latest shipping Xcode with no betas (D11, D20). Exact minors are recorded at Phase 1 in `docs/architecture.md` and `Package.resolved`; do not assert other current versions.
- Where the register is silent, choose the conservative option, mark it "verify at Phase N" in the code comment or document, and list it in the phase review. If you believe a decision is wrong, follow it and raise it in the review.

| Need | Read |
|---|---|
| iPhone board, drag, non-drag menu | D01 |
| iPad split view, layout modes, keyboard | D02 |
| Roles, capability matrix, ownership | D03 |
| Invitation links and tokens | D04 |
| Versions, PATCH merge, 409 rules | D05 |
| Rank keys, move contract, renormalization | D06 |
| Local vs APNs notifications, quiet hours | D07 |
| Offline cache, outbox, conflict UX | D08 |
| Account deletion, Apple revocation | D09 |
| Ingress profiles, ATS, what touches Apple | D10 |
| Server stack, Compose, one binary, launchd | D11, ADR-0001 |
| SwiftData stores, repository seam | D12, ADR-0002 |
| WebSocket protocol, catch-up, eviction | D13, ADR-0003 |
| Attachments, upload tokens, allowlist | D14, ADR-0004 |
| MCP tools, proposals, scopes, audit | D15, ADR-0005 |
| Sessions, token classes, Keychain | D16 |
| Activity events, sequence, action vocabulary | D17 |
| operationId, replay, DELETE no-op | D18 |
| Column count, templates, done column, WIP, archive | D19 |
| Targets, toolchain, who runs what, human checklist | D20 |
| Backups, restore drill, host settings | D21 |
| Error envelope, codes, pagination, headers, rate limits | D22 |
