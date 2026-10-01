# Phase 0 register amendments

Applied on 2026-10-01 to `docs/decisions/decision-register.md` after the Phase 0 writer review. Every amendment is also listed in the register's amendment log with its concern ids.

## What changed and why

- S1-A01: backups mount read-only into containers; backup status is handed over through `data/backups/backup-status.json`, no table.
- S1-A02: the launchd path is a cold standby with `switch.sh to-launchd|to-compose` moving data by dump and restore.
- S1-A03: Vapor 4 is the decision; the Vapor 5 look defaults to staying.
- S1-A04: `/ready` separates `database_unreachable` from `migrations_pending` (503 `not_ready`); `health.sh` never restarts for pending migrations.
- S1-A05: Phase 10 validation runs on Docker Desktop; OrbStack is revalidated only if ever adopted.
- S1-A06: optimistic edits write the outbox first, then the cache; a launch-time pass repairs any crash in between.
- S1-A07: snapshot replace and the schema wipe keep rows that have pending operations.
- S1-A08: the client caches the newest 200 events and reads older feed pages through.
- S1-A09: a project with pending operations is never evicted from the cache.
- S1-A10: only membership removal and project deletion evict realtime subscribers; archived projects stay subscribable.
- S1-A11: one socket, at most 20 subscriptions, by most recently opened.
- S1-A12: live frames arriving during a REST catch-up are buffered, not dropped.
- S2-A02: a staged upload is not a queued operation; initiate runs automatically; token expiry re-initiates up to three times.
- S2-A03: count, quota and global-cap refusals are 422 `validation_failed` with fixed reasons.
- S2-A04: the sniffed-type table is spelled out per family.
- S2-A05: client-side SHA-256 hashing is an accepted cost.
- S2-A09: the MCP Swift SDK is pre-approved; protocol errors are JSON-RPC errors, service outcomes are `isError` tool results.
- S2-A11: a confirmed change applies all-or-nothing in one transaction.
- S2-A12: `remote_addr` is null on the loopback transport.
- S2-A15: the secrets image goes to the SSD, last 3 kept; restic gets explicit paths, never excludes.
- S2-A16: the checklist names the fixed secrets path and the three .p8 file names.
- S2-A17: `deploy/runbook.md` is the one operations document; `docs/deployment.md` is never created.
- S2-A18: CLAUDE.md owns branch, commit and review naming; the owner approves in chat and the session merges and tags.
- S2-A19: Phase 9 ships without OAuth; that is a named deviation from the brief.
- S3-A01: one task-entry rule: a column button targets that column, "+" and Cmd-N target the selected column, the API defaults to the first column.
- S3-A02: the WIP indicator is orange at the limit and red above it, everywhere.
- S3-A03: the conflict banner title is client-built; the server message is its detail line.
- S3-A04: the manual notes merge is called Compare everywhere.
- S3-A05: the done column carries a checkmark marker wherever its title appears.
- S3-A06: Reduce Motion means no springs anywhere, through one `AppMotion` value.
- S3-A08: nine named column colors with per-template defaults.
- S3-A10: any signed-in user can create a project and owns it.
- S3-A11: an `operator` push category carries server alerts to flagged users.
- S3-A12: the human veto is the Claude client's per-call approval; setup forbids auto-approving `confirm_change`.
- S3-A13: Phase 11 thresholds fixed: 100 moves per device, a 20 MB upload, a 26 MB refusal.
- S3-A14: the no-tutorial criterion is checked by one moderated session.
- S3-A15: the login keychain never locks; Cowork never runs `security unlock-keychain` or sees a password.
- S3-A16: one standard macOS account `kanban` from Phase 1; the owner's admin account answers prompts.
- S3-A17: the Associated Domains value and the universal-link experiment move to Phase 10.
- S3-A18: Program License Agreement at enrollment, then any App Store Connect banner.
- S3-A19: SSD at least 1 TB, iCloud Drive at least 50 GB free, alert at 80 % use.
- S4-A01: fixtures are the exhaustive contract; api.md shows shapes and unique examples.
- S4-A03: every `confirm_change` and `cancel_change` outcome is enumerated using existing codes.
- S4-A05: the closed list of POSTs without an `operationId`.
- S4-A06: no archive risk clause; scope rules for confirm and cancel; `destructiveHint` on `confirm_change`.
- S4-A07: `Retry-After` plus `details.reason` is the only rate-limit signal.
- S4-A08: the account, archive, task list, grant and server-status paths are ratified.
- S4-A09: `column.reordered` is a project-entity event that merges with a rename.
- S4-A10: the deviations table has five rows, including devices.
- S4-A11: task proposal tools take 1 to 50 items in one project.
- S4-A12: `X-Client-Version` is optional on the wire, mandatory for the app.
- S4-A13: the stdio bridge reads `SHAREDKANBAN_MCP_TOKEN` and `SHAREDKANBAN_MCP_URL`.
- S4-A16: FileVault OFF is applied without asking; Tailscale toggles are checklist item 5.
- S5-A01: security pushes ride a generalized worker outbox keyed by `(source, source_id, user_id)`.
- S5-A02: account deletion locks its projects in ascending id order.
- S5-A04: invitation mutations lock the project row, then the invitation row.
- S5-A05: notification settings are typed jsonb with `PUT` endpoints.
- S5-A06: layouts for `auth_challenges`, `apple_revoke_retries` and `security_events`.
- S5-A07: attachment hash mismatch is 409 `attachment_mismatch`, not `version_conflict`.
- S5-A08: pending AI changes are purged 30 days after their terminal state.
- S5-A09: Apple revoke runs inline after commit, bounded, before the 204.
- S5-A10: sign-in nonces are client-generated; replay is a primary-key conflict.
- S5-A13: `GET /v1/server/status` and a "Server needs attention" banner back up push alerts.
- S5-A15: Compose images pinned by digest, Dependabot on, TestFlight is the only crash telemetry.
- S5-A16: `APPLE_TOKEN_KEY` protects exported artifacts only.
- S5-A17: unauthenticated routes get per-IP limits and a global ceiling; `/ready` is never on Funnel.
- S5-A18: the unlocked `kanban` keychain is an accepted, scoped risk.
- S5-A19: Tailscale signs in with the owner's Apple ID; device approval is on.
- S5-A20: host hardening defaults (sharing off, firewall on, standard user).
- S5-A21: the fragment fallback is a path token with redaction, decided now; retention is disclosed in-app.
- A901 to A905: new contradictions resolved: `docs/deployment.md` references retargeted to the runbook; three error codes added in one edit; the two re-initiating 4xx codes named in D14; S5-A18 applied after S3-A15; `sk.projectId` and `sk.sequence` made optional.

## Choices you can override with one word

- S3-A07 Upcoming window. Default: a card is "upcoming" within 48 hours of its due date, fixed. Alternatives: 24 hours, 7 days, or tied to the reminder lead time. Why: covers "tomorrow" without turning the board amber and needs no setting.
- S3-A09 Save model. Default: title and notes autosave when editing ends; no Save button, no discard alert. Alternatives: an explicit Save button, or debounced autosave while typing. Why: matches Notes and Reminders and sends one PATCH per field.
- S4-A14 iOS build automation. Default: none beyond Cowork's `xcodebuild` after every slice. Alternatives: a billed GitHub macOS job, or a self-hosted runner on the Mac mini. Why: zero cost and zero steps; either can be enabled later.
- S4-A15 Linux toolchain. Default: asked once in the Phase 1 review; until then server tests run in CI and on the Mac mini. Alternatives: never, or before Phase 1. Why: the fallback already compiles everything.
- S4-A18 (absorbs S2-A13) Claude client. Default: Claude Desktop with the stdio bridge, configured unattended by Cowork. Alternatives: Claude Code, or both. Why: Cowork runs inside Claude Desktop, and Cowork can write its configuration without you transcribing a token.
- S5-A11 Sign-up. Default: `SIGNUP_MODE=invite_only`, first user bootstrapped automatically. Alternative: `open`. Why: closes stranger sign-up with no action from you.
- S5-A12 Local dumps. Default: FileVault OFF, last 2 plaintext dumps kept locally. Alternatives: FileVault ON (manual unlock after restarts), or zero local dumps. Why: unattended availability was the D21 choice; fewer copies cost nothing.
- S5-A14 (absorbs S2-A08) MCP veto. Default: the Claude client's per-call approval, plus a security push and a visible grant whenever a Claude token is created. Alternative: in-app Approve before `confirm_change` succeeds, pulled into Phase 9. Why: always-confirm plus audit is already safe; notification gives visibility without delaying Phase 9.

## Concerns rejected

- S2-A20 (C037): D20 already says to start enrollment first and the risks say Phase 1.
- S3-A20 (C074, C075): the register asserts no current portal or console labels; Cowork verifies them while driving the browser.
- S3-A21 (C067, C076): word counts and the cloud network allowlist are document and environment matters.
- S4-A02 (C095): word-count accounting is an orchestration setting; S4-A01 removes the pressure.
- S5-A22 (C129, C133): document length and orchestrator commits are workflow questions.
- S5-A23 (C132): Linux toolchain availability is already an assumption with an `XCTSkip` rule.
- Merged, not rejected: S2-A01 into S5-A07; S2-A06 and S2-A07 into S4-A06; S2-A08 into S3-A12; S2-A10 into S4-A03; S2-A13 into S4-A18; S2-A14 into S4-A13; S4-A04 and S5-A03 into S1-A10; S4-A17 into S3-A16; S4-A19 into S2-A18.
