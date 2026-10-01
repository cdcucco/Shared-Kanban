# SharedKanban operator playbook

Who does what for every phase of SharedKanban, written for an owner who wants to be involved as little as possible and for the Claude sessions that implement Phases 1 to 11 without guessing. It applies docs/decisions/decision-register.md (cited inline as D01 to D22) and the phase table in the owner's brief. Step-by-step human instructions live in docs/human-checklist.md, the architecture in docs/architecture.md, the repository rules in CLAUDE.md.

## 1. The three executors

| Executor | Can do | Cannot do |
|---|---|---|
| (A) Cloud: Claude Code in a cloud Linux container (Ubuntu 24.04, Node 22, Python 3.11, local PostgreSQL 16, Docker CLI without a daemon) | Write all code and docs; run Node and Python tooling and local PostgreSQL; commit, push, open and merge pull requests; read CI. With section 4 done, also `swift build` and `swift test` for `shared/` and `server/` (D20). | Compile Swift today (no toolchain; download.swift.org and github.com file downloads are denied by the network policy; registry.npmjs.org and PyPI are allowed); run Docker containers; run Xcode, simulators or devices; sign apps; reach the Mac mini or Apple. |
| (B) Mac mini Cowork: Claude Cowork sessions the owner directs on the Mac mini | Install and run Xcode and tools; `xcodebuild build/test` on simulators; `xcrun simctl` and `devicectl`; screenshots; Docker Desktop or OrbStack and Compose; native `swift test`; migrations; Tailscale, restic, cloudflared, launchd; restore drills; drive a browser while the human types credentials; commit and push the results note (D20). | Type Apple ID passwords or 2FA codes; pay; accept legal agreements; physically connect, trust or unlock devices; create third-party accounts; tap a phone. |
| (C) Human: the owner | Read a phase review and reply "approved" or list changes; do the one-time actions in docs/human-checklist.md; hold a phone when a test needs a finger. | Nothing else is asked. |

Rule (D20): the cloud writes everything and tests what Linux can test; the Mac mini builds, tests and operates; the human reads and approves.

## 2. Handoff protocol

Branches. Each phase lives on one branch `phase-N-<slug>` (for example `phase-1-scaffold`) cut from `main`; the cloud session commits there in small commits and opens a pull request. Nobody commits to `main` directly; the cloud session merges after the human's "approved" and tags `phase-N-done`.

Results note. When a phase needs the Mac mini, a Cowork session pulls the branch, runs what its prompt says, commits exactly one file, `docs/reviews/phase-N-macmini-results.md` (plus screenshots where the prompt says so), and pushes to the same branch. Standard shape:

```text
# Phase N Mac mini results
Date, macOS, Xcode, Swift versions, branch, commit SHA
## Commands run      (exact command, PASS or FAIL)
## Checks            (check | result | evidence)
## Screenshots       (paths under docs/reviews/phase-N-screenshots/)
## Resolved "verify at Phase N" items
## Needs the human   (exact tap, credential or purchase, or "none")
## Verdict: PASS | PARTIAL | FAIL
```

Phase review. The cloud session reads the results note and writes `docs/reviews/phase-N-review.md`: what was built, what the gate required, evidence it passed, open risks, at most one question. The human reads only that file and replies "approved" or lists changes; that reply is the done signal for every phase.

Secrets. Never in chat, the repository, CLAUDE.md, a results note or a review. On the Mac mini they live in `/Users/kanban/SharedKanban/secrets/` (D11, D21): `.env` (a committed `.env.example` lists every variable name), the Sign in with Apple, APNs and App Store Connect API keys as `.p8` files, and `restic-password`. Until the `kanban` service user exists (Phase 10), Cowork keeps the identical layout at `~/SharedKanban/secrets/`. The iOS base URL lives in the git-ignored `Config/Environment.xcconfig` (D10). A Cowork session that needs a secret names the file and waits; it never asks for a value in chat.

## 3. Automation that keeps the human out of the loop

GitHub Actions. The cloud session writes `.github/workflows/ci.yml` in Phase 1: on every push and pull request, one job on `ubuntu-latest` in the official `swift:6.x` container (exact tag recorded at Phase 1, D11) with a `postgres:17-alpine` service, running `swift build` and `swift test` in `shared/` and `server/` with `DATABASE_URL` set (D20). Integration tests `XCTSkip` without `DATABASE_URL`, so one script serves the cloud container, the Mac mini and CI, with "requires macOS" markers for iOS steps. Branch protection on `main` requires the job. Human action: none. Actions is enabled by default for a repository, and the workflow needs no secrets because automated tests never touch live Apple endpoints (D09, D20).

iOS build automation. Per D20 there are no macOS runners in release 1; the Mac mini Cowork session is the macOS CI. Two optional upgrades exist, not enabled in release 1. A GitHub-hosted macOS job runs iOS builds on every push without the Mac mini, but macOS minutes are billed at a multiplier, so accepting the cost is a human payment decision. A Mac mini self-hosted runner does the same unattended; it needs one sitting in which a Cowork session installs the runner as a launchd service and pastes the registration token from the repository settings page, opened in the browser while the human signs in to GitHub. Say so in any phase reply to enable either.

## 4. Optional one-time human action: let the cloud compile Swift

Today the cloud session writes the Vapor server and SharedDTOs but cannot build them; GitHub Actions and the Mac mini compile instead (the fallback D20 allows). Enabling Swift in the cloud saves one round trip per phase from Phase 2 on. Five minutes, once; verify every step at Phase 1, where the exact Swift version is chosen (D11).

1. In a Claude Code cloud session, open the session title bar menu and choose Edit the cloud environment.
2. Under Network access, add `download.swift.org` (or choose a broader level if a single-domain allowlist is not offered); for swiftly also allow `swift.org`.
3. Under Setup script, add one install approach (chosen at Phase 1, recorded in docs/architecture.md):
   - swiftly (the swift.org installer): download `swiftly-$(uname -m).tar.gz` from `download.swift.org/swiftly/linux/`, run `./swiftly init --assume-yes` and `swiftly install 6.x`, add swiftly's bin directory to `PATH`.
   - Toolchain tarball: download the Ubuntu 24.04 tarball for the pinned Swift 6.x release from `download.swift.org/swift-6.x-release/ubuntu2404/`, extract it, add its `usr/bin` to `PATH`.
   Both need the toolchain's apt dependencies listed on swift.org, so the script also runs `apt-get install`; if apt mirrors are blocked, the Phase 1 cloud session says which domain to add.
4. Save. The Phase 1 cloud session proves it by running `swift test` in `shared/` and `server/` against the local PostgreSQL (D11, D20).

Skipping this blocks nothing.

## 5. Per-phase plan

The done signal is always the human's reply on docs/reviews/phase-N-review.md; the column lists what must be true before that review is written. "Nothing" means read the review and reply. "Results note" means `docs/reviews/phase-N-macmini-results.md` with Verdict PASS; "Prompt N" refers to section 6.

### Phase 0: Product and architecture

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Decision register, ADRs, product, architecture, design, threat-model and API docs, this playbook, docs/human-checklist.md, CLAUDE.md | Decisions reviewed; no application code | Writes every document, opens the PR, writes the review | Nothing | Read the review, reply | Reply; cloud merges and tags `phase-0-done` |

### Phase 1: Repository scaffold

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Monorepo, SharedDTOs, Vapor service with `/health` and `/ready`, iOS shell, Compose `dev` profile (D11), example config files (D10), CI workflow, test scripts | Empty app and API build; CI passes | Writes everything; runs `swift test` if section 4 is done; records resolved versions (D20) | Prompt 1: Xcode and Docker install, simulator and native server tests, device iOS versions (D20) | One sitting: Apple ID on the Mac mini and in Xcode, Xcode license, Docker Desktop first launch, admin password; start Developer Program enrollment (takes days, gates Phase 3); optionally section 4 | CI green, results note, versions recorded, reply |

### Phase 2: Domain and database

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Migrations, column-count trigger (D19), `ProjectAuthorizer` (D03), `Mutation.commit` (D17), idempotency (D18), ranks and moves (D06), `ConcurrencyGate` (D05) | Unit and integration tests for permissions and ordering pass | Writes code and the test lists in D05, D06, D18, D19; CI runs them | Native `swift test` on PostgreSQL 17; confirms `COLLATE "C"` is byte-wise and the savepoint and two-pass renormalization work through Fluent plus SQLKit (D06, D18) | Nothing | CI green, results note, reply |

### Phase 3: Sign in with Apple and sessions

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Sign in with Apple and sessions (D16), account deletion with challenge and Apple revocation (D09), `AppleIdentityProvider` fake, Settings screen | Real-device sign-in works; token and deletion tests pass | Writes client and server code, the D09 and D16 tests, portal and variable docs without values | Prompt 3: portal keys with the human, device installs and checklist, Apple server flow checked (D09) | One sitting: enrollment active; Developer Mode and trust on both devices; with Cowork driving the portal, create the three keys and download each `.p8` into the secrets folder; tap Sign in with Apple when asked | Results note with sign-in and deletion observed on a device, reply |

### Phase 4: Native design shell and local sample data

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Every screen on the in-memory repository fake (D12) with the D01 and D02 layouts, keyboard focus model, offline and conflict states (D08), templates (D19), design review checklist | iPhone and iPad layouts reviewed at multiple sizes | Writes views, previews, snapshot and UI tests at the D02 widths | Prompt 4: tests, screenshots at every D02 width, inspector and switcher behavior verified (D01, D02), device OS versions (D20) | Look at the screenshots linked from the review; reply | Results note with screenshots committed, reply |

### Phase 5: Core client and sync

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| `APIClient`, repositories, two SwiftData containers with `SyncEngine` (D12), outbox and draft overlay (D08), Keychain session store (D16), catch-up, conflict UI, fixture contract tests (D22) | Two devices can create and edit shared data | Writes code and the D08 and D12 tests, including "server event arrives for an entity with a pending draft" | Server on the LAN through the Compose `dev` profile (D10), two simulators with separate fake accounts, UI tests, two-store setup verified on the minimum OS (D12) | Nothing | Results note with the two-simulator demonstration, reply |

### Phase 6: Drag and drop, realtime, offline queue

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Drop targets 1 to 4 plus the edge-dwell extra (D01), `MoveCoordinator` (D06), `WSS /v1/realtime` hub and reconnect (D13), menu alternatives and shortcuts (D02), two-device script | Concurrent moves and reconnect tests pass | Writes code and the D06 and D13 test lists | Prompt 6: two-simulator scenario and WebSocket tests; edge-dwell paging (D01) and WebSocket findings (D13) | Nothing; an optional ten-minute real-device drag-feel check is offered in the review | Results note, reply |

### Phase 7: Invitations, roles, comments, activity

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Invitations, `/invite` page, AASA, code field (D04); roles, removal, ownership transfer, project delete and restore (D03); comments (D07); activity feed (D17); security review | Permission matrix and invite lifecycle pass | Writes code, the full role-by-capability tests (D03) and the D04 lifecycle tests | Runs tests; tries Associated Domains developer mode on the devices (D04); installs the build on both | One five-minute check: open an invitation link received in Messages on the other device (fragment survival, D04) | Results note including the fragment finding, reply |

### Phase 8: Due dates, attachments, notifications

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Local reminders, devices, notification outbox and worker with the `PushProvider` fake (D07, D17); attachments with background PUT, sniffing, quotas and cleanup (D14); device checklist | Background upload and APNs sandbox tests pass | Writes code, the D07 and D14 tests, and the APNs setup notes | Prompt 8: tests, APNs sandbox on a real device with the human, Focus handling of passive pushes (D07) | Present about fifteen minutes: tap Allow, attach a photo, lock the phone, report whether the push and the reminder arrived | Results note with the sandbox push observed, reply |

### Phase 9: MCP connector

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| MCP server inside `api` on loopback, `mcp-stdio` bridge, `mcp-token`, read and proposal tools, confirm and cancel, scopes, audit log, Connect Claude screen (D15), setup instructions | Read, propose, confirm, deny and audit tests pass | Writes code and the D15 adversarial tests; documents the per-call approval dependency (D15) | Creates a token with `Run mcp-token create`, pairs the Claude client through the stdio bridge, demonstrates read, propose, deny, propose again, confirm and the audit row | Nothing, unless Cowork cannot paste the token into the Claude client (checklist item 9) | Results note, reply. Phase 9b (remote OAuth) only if the owner asks for it (D15) |

### Phase 10: Mac mini deployment and backups

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Production Compose and launchd path (D11); backup, restore, health and export-secrets scripts (D21); deploy/runbook.md with both ingress profiles (D10), incident steps, monthly drill, status screen | Restore drill succeeds; remote HTTPS works | Writes every deploy file and the runbook; validation is entirely on the Mac mini | Prompt 10: install, deploy, `tailscale serve`, launchd path exercised once, restore drill into `sharedkanban-drill`, a week of restic copies to iCloud Drive (D21) | One sitting: Tailscale account and sign-in on all three devices; admin-console toggles with Cowork driving the browser (D10); admin password when asked; plug in the UPS and SSD; save the restic password and passphrase in a password manager; accept FileVault OFF in the reply or ask for ON (D21) | Results note with `/ready` reached over https from both devices and the drill's integrity checks, reply |

### Phase 11: Accessibility, security, performance, beta

| Deliverable | Exit gate | Cloud does | Mac mini Cowork does | Human does | Done signal |
|---|---|---|---|---|---|
| Release-candidate audit per the brief's Prompt 11, blocker fixes, TestFlight checklist, privacy inventory including "Deleted member" retention (D09, D17), support runbook, release criteria | TestFlight checklist and release criteria pass | Writes audits, fixes, checklist and inventory; labels non-blocking follow-ups | Prompt 11: every suite, `@Query` timing with about 1,000 tasks (D12), TestFlight upload with the App Store Connect API key (D20) | Accept the TestFlight invite on each device, use the app for a few days, reply with anything that felt wrong | Both devices on the TestFlight build, results note, "approved" closes release 1 |

## 6. Ready-to-paste Cowork prompts

Paste one prompt into a Claude Cowork session on the Mac mini. All use `~/Developer/SharedKanban` as the working clone and the results-note shape from section 2; no secret, key id, team id or hostname ever enters a note.

### Prompt 1: Phase 1 scaffold build

Human must be present: yes, for the first ten minutes (Apple ID sign-in, Xcode license, Docker Desktop first launch, admin password; D20), then the session runs alone.

```text
Mac mini session for SharedKanban, Phase 1. Read CLAUDE.md and docs/operator-playbook.md section 2 first.
1. Clone the SharedKanban repository to ~/Developer/SharedKanban if absent; check out phase-1-scaffold and pull.
2. Install the latest shipping Xcode (no betas) and the command line tools; run `xcodebuild -runFirstLaunch` and `-downloadPlatform iOS`. When the Apple ID or license needs the human, say exactly what to click and wait.
3. Install Docker Desktop (or OrbStack); ask the human once for the first-launch approval and admin password.
4. Run the documented scripts: `swift build` and `swift test` in shared/ and server/ against the Compose dev PostgreSQL 17; `xcodebuild build` and `test` on an iPhone and an iPad simulator; `docker compose --profile dev up` and curl /health and /ready.
5. Record macOS, Xcode, Swift, Vapor and PostgreSQL versions and the iOS version of any cable-connected device (both must be 18 or later).
6. Write docs/reviews/phase-1-macmini-results.md in the standard format; commit as "docs: phase 1 Mac mini results" and push. Commit nothing else.
```

### Prompt 3: Phase 3 real-device sign-in

Human must be present: yes. Sign in with Apple cannot run in the Simulator, the portal keys need the Apple ID session, and someone must tap the sign-in button and Face ID (D20).

```text
Mac mini session for SharedKanban, Phase 3. Read CLAUDE.md, docs/operator-playbook.md section 2 and docs/human-checklist.md first.
1. In ~/Developer/SharedKanban check out phase-3-sign-in-with-apple and pull. Run the server and iOS tests; stop and report any failure.
2. Confirm the Apple Developer Program is active in Xcode > Settings > Accounts; if not, stop and tell the human.
3. Open the developer portal in the browser and guide the human through the checklist's portal steps (App ID capabilities, Sign in with Apple key, APNs key, App Store Connect API key). Move each downloaded .p8 into ~/SharedKanban/secrets/ under the checklist's names; never read their contents into chat.
4. Fill ~/SharedKanban/secrets/.env from .env.example, start the server with the Compose dev profile on the LAN, set the LAN base URL in Config/Environment.xcconfig, and install the Debug build on both cable-connected devices with `xcrun devicectl device install app`.
5. Walk the human through the branch's device checklist and record each outcome as reported.
6. Write docs/reviews/phase-3-macmini-results.md in the standard format with the outcomes and the resolved D09 Apple-flow items; commit as "docs: phase 3 Mac mini results" and push.
```

### Prompt 4: Phase 4 design review screenshots

Human must be present: no. Everything runs on simulators; the owner reads the screenshots later.

```text
Mac mini session for SharedKanban, Phase 4. Read CLAUDE.md and docs/operator-playbook.md section 2 first.
1. In ~/Developer/SharedKanban check out phase-4-design-shell and pull. Run the iOS unit, snapshot and UI tests.
2. With `xcrun simctl`, capture every screen in the branch's design review checklist at widths 320, 375, 507, 744, 834, 1024, 1194 and 1366 points, light and dark, default and accessibility XL type, plus Slide Over, one-third and one-half Split View and a narrow Stage Manager window.
3. Save PNGs under docs/reviews/phase-4-screenshots/ as <screen>-<width>-<light|dark>-<default|axxl>.png; keep the set under 40 MB.
4. Check and record: no clipped titles at accessibility XL in a 240pt column; opening the inspector never flips the board mode; onGeometryChange behaves during live resize; the switcher bar renders with the SDK's chrome. Record the iOS versions of any listed devices.
5. Write docs/reviews/phase-4-macmini-results.md in the standard format; commit the note and screenshots as "docs: phase 4 Mac mini results and screenshots" and push.
```

### Prompt 6: Phase 6 two-device test

Human must be present: no. The scenarios run on two simulators with fake accounts; the real-device drag-feel check is optional.

```text
Mac mini session for SharedKanban, Phase 6. Read CLAUDE.md and docs/operator-playbook.md section 2 first.
1. In ~/Developer/SharedKanban check out phase-6-drag-realtime and pull. Start the server with the Compose dev profile. Run the server and iOS tests; stop and report any failure.
2. Boot an iPhone and an iPad simulator as two fake accounts on one Home Projects board and run the branch's two-device script end to end (simultaneous moves, same-card conflict with snap-back, offline edits replayed, member removal closing the socket).
3. Record whether edge-dwell paging survives a live drag, whether URLSessionWebSocketTask sends the Authorization header on upgrade, and what happens 5 seconds after backgrounding.
4. Screenshot the drag preview, insertion bar, WIP warning and rejection banner into docs/reviews/phase-6-screenshots/.
5. Write docs/reviews/phase-6-macmini-results.md in the standard format; under "Needs the human" list the optional real-device drag-feel check. Commit as "docs: phase 6 Mac mini results" and push.
```

### Prompt 8: Phase 8 APNs sandbox

Human must be present: yes, about fifteen minutes: a real device must grant the notification permission and pick a photo, and only a person can report what the lock screen showed (D07, D20).

```text
Mac mini session for SharedKanban, Phase 8. Read CLAUDE.md and docs/operator-playbook.md section 2 first.
1. In ~/Developer/SharedKanban check out phase-8-dates-files-notifications and pull. Run the server and iOS tests; stop and report any failure.
2. Confirm ~/SharedKanban/secrets/ holds the APNs .p8 and that .env names its key id and team id; never print them. Start the server and worker with the Compose dev profile, APNs pointed at the sandbox environment.
3. Install the Debug build on both cable-connected devices and run the branch's device checklist with the human: assignment push (tap Allow at the prompt), comment push with the iPhone locked, due reminder, photo upload completing after the app is swiped away, passive-level push under a Focus mode.
4. Record the delivery rows (ids only), the APNs response status, the device row's environment value and the sniffed MIME of the uploaded HEIC.
5. Write docs/reviews/phase-8-macmini-results.md in the standard format with each check as the human reported it; commit as "docs: phase 8 Mac mini results" and push.
```

### Prompt 10: Phase 10 deployment and restore drill

Human must be present: yes, for one sitting at the start (admin password, Tailscale sign-in and admin console, UPS and SSD; D10, D20, D21); the drill and the backup observation run alone.

```text
Mac mini session for SharedKanban, Phase 10. Read CLAUDE.md, docs/operator-playbook.md section 2 and deploy/runbook.md first.
1. In ~/Developer/SharedKanban check out phase-10-deployment and pull.
2. With the human, follow the runbook's installation section: the `kanban` service user with auto-login; Docker Desktop (or OrbStack), Tailscale and restic under it; secrets moved to /Users/kanban/SharedKanban/secrets/ (0700, files 0600); Tailscale sign-in and the admin-console toggles (HTTPS certificates on, key expiry off); UPS on USB, SSD mounted at /Volumes/KanbanBackup; power settings. Record the FileVault choice (default OFF).
3. Alone, follow the deploy section: .env from .env.example, `docker compose run --rm migrate`, api and worker up, api on 127.0.0.1:8080 only and postgres unpublished, `tailscale serve` to 127.0.0.1:8080, /ready reachable through the tailnet hostname. Exercise the launchd escape hatch once, then return to Compose.
4. Install the backup launchd jobs, run backup.sh once, run restore.sh into the sharedkanban-drill stack and confirm every integrity check it prints; confirm health.sh restarts the stack after three failures. Leave the nightly restic copy to iCloud Drive running seven days and add the outcome in a second commit.
5. Ask the human to open the app on both devices with the Tailscale VPN on and confirm the board loads over https.
6. Write docs/reviews/phase-10-macmini-results.md in the standard format, including the launchd exercise, the WebSocket and idle-timeout findings for tailscale serve and the FileVault choice; commit as "docs: phase 10 Mac mini results" and push.
```

### Prompt 11: Phase 11 TestFlight

Human must be present: no; the App Store Connect API key lets Cowork archive and upload unattended (D20). The human accepts the TestFlight invitation on each device later.

```text
Mac mini session for SharedKanban, Phase 11. Read CLAUDE.md and docs/operator-playbook.md section 2 first.
1. In ~/Developer/SharedKanban check out phase-11-release-candidate and pull. Run the full release-candidate suite the branch documents, including the two-user end-to-end scenario on two simulators. Stop and report any failure.
2. Seed a project with about 1,000 tasks through the API and measure board scroll and @Query timing on iPhone and iPad simulators; record the numbers.
3. Run `security unlock-keychain` if signing needs it. Archive the Release configuration with automatic signing, export for TestFlight, and upload with the App Store Connect API key from /Users/kanban/SharedKanban/secrets/. Never print the key.
4. In App Store Connect, add both household testers to the internal group, confirm the build finishes processing, and answer export compliance as the runbook documents.
5. Walk the branch's TestFlight checklist and mark each item from evidence you produced.
6. Write docs/reviews/phase-11-macmini-results.md in the standard format with test results, performance numbers, build number and the checklist; under "Needs the human": accept the TestFlight invite on each device. Commit and push.
```

## 7. One-time human actions

Step-by-step instructions live in docs/human-checklist.md. "Cowork beside the human" means a Cowork session opens the page and explains each click; the human only types credentials, approves prompts or pays.

| Action | Phase that needs it | Who drives it |
|---|---|---|
| Apple Developer Program enrollment and payment; App Store Connect agreements | Start in Phase 1; needed by Phase 3 | Human alone |
| Apple ID sign-in on the Mac mini and in Xcode; 2FA; Xcode license | Phase 1 | Cowork beside the human |
| Docker Desktop first launch and macOS admin password | Phase 1; again in Phase 10 for the service user | Cowork beside the human |
| App ID capabilities: Sign in with Apple, Push Notifications, Associated Domains | Phase 3 (Associated Domains used from Phase 7) | Cowork beside the human (automatic signing creates most of it) |
| Sign in with Apple key, APNs key, App Store Connect API key, each `.p8` stored once in the secrets folder, never the repo | Phase 3; used in Phases 8 and 11 | Cowork beside the human |
| Developer Mode and trusting the Mac on the iPhone and iPad; allowing notifications and Sign in with Apple when the app asks | Phases 3 and 8 | Human alone (Cowork says when) |
| Tailscale account; sign-in on the Mac mini, iPhone and iPad; HTTPS certificates on and key expiry off in the admin console | Phase 10 | Cowork beside the human for the console; device sign-ins human alone |
| Optional Cloudflare account and domain | Only if Cloudflare Tunnel replaces Funnel for the public profile (D10) | Human alone |
| Optional cloud environment change for Swift (section 4) | Phase 1 or any time | Human alone, five minutes |
| TestFlight acceptance on each device | Phase 11 | Human alone |
| UPS and external backup SSD plugged in | Phase 10 | Human alone |
| restic password and secrets-image passphrase saved in a password manager | Phase 10 | Human alone (Cowork generates both) |
| Optional: Backblaze B2 account if iCloud Drive proves unreliable; Claude connection token pasted into the Claude client if Cowork cannot | After Phase 10; Phase 9 | Human alone |
| Optional: GitHub sign-in for a self-hosted runner token | Only if the runner is wanted | Cowork beside the human |

## 8. What you need to do right now

For Phase 0, nothing except this: read docs/reviews/phase-0-review.md and reply "approved", or reply with the changes you want. No purchases, accounts or installs are needed until the Phase 1 review asks for the single sitting in section 5.
