# SharedKanban product specification

Phase 0 document. Every behavior follows `docs/decisions/decision-register.md` (D01–D22), cited inline; contracts are in `docs/api.md`, layouts in `docs/design-system.md`. Written 2026-10-01; platform versions are those the register fixes (iOS 18 minimum, Swift 6, Vapor 4, Fluent 4, PostgreSQL 17), verified at Phase 1 (D11, D20).

## 1. Purpose, audience and mental model

SharedKanban is a native iPhone and iPad app for a small shared Kanban board hosted on the household's own Mac mini. The first audience is a two-person household; the second is any small group that trusts one owner to run the server. Apple is used only for Sign in with Apple and push delivery (D10).

Five concepts, nothing more:

- **Project**: a shared board with members, settings and an ordered workflow of exactly 3 or 4 active columns (D19).
- **Column**: one ordered stage in a project with a title, color and optional WIP limit; one column may be the "done" column (D19).
- **Task**: a card with a required title and optional notes, assignee, due date, attachments and comments, at an explicit position (rank) in one column (D06).
- **Member**: a user in a project with one role (owner, admin, editor, viewer) and personal notification settings (D03, D07).
- **Activity event**: an append-only, per-project, gap-free record of each change that drives realtime updates, the feed, notifications and conflict detection (D17).

## 2. Worked examples

Every card shows the same compact surface: title (2–3 lines), optional one-line notes preview, due date with neutral / upcoming / overdue treatment, assignee initials, attachment and comment counts, and a sync glyph only when needed (D01, D08).

### Home Projects (Backlog, To Do, In Progress, Complete)

Template id `home-projects`; "Complete" is the done column.

```text
┌ In Progress  2/2 ⚠ ────────────────┐
│ Fix the garage door opener         │
│ Spring is loose; part ordered…     │
│ Due Fri (upcoming)   SA   📎1  💬3  │
└────────────────────────────────────┘
```

`2/2 ⚠` means a WIP limit of 2 is reached; a third drop is still allowed and keeps the warning (D19).

### Restaurants to Try (Backlog, Scheduled, Complete)

Template id `restaurants-to-try`; "Complete" is the done column.

```text
┌ Scheduled  1 ──────────────────────┐
│ Lucia's Trattoria                  │
│ Booked for two, ask for the patio  │
│ Sat 19:00 (upcoming)   CD   📎0 💬1 │
└────────────────────────────────────┘
```

### Custom template

Template id `custom`; the user types 3 or 4 names, for example a trip board: Ideas, Booked, Packed, Done. Names are editable later and reorderable from Project settings → Columns (D19).

```text
┌ Booked  3 ─────────────────────────┐
│ Ferry tickets                      │
│ Due Oct 12 (overdue)  —  📎2  💬0   │
└────────────────────────────────────┘
```

## 3. Release scope

| Area | Required in release 1 | Deliberately deferred |
|---|---|---|
| Accounts | Native Sign in with Apple, per-installation sessions, in-app account deletion (D09, D16) | Password accounts, Google login |
| Projects | Create from template, rename, archive, soft delete with 30-day restore, ownership transfer (D03, D19) | Nested projects, portfolios |
| Columns | Built-in names, rename, reorder, color accent, optional WIP limit, done column, archive (D19) | Arbitrary workflow automation |
| Tasks | Title, notes, assignee, due date, attachments, comments, archive, restore, search (D05, D14, D19) | Dependencies, recurring tasks, subtasks |
| Collaboration | Invite, accept, decline, revoke, remove, roles, realtime changes, activity feed (D03, D04, D13) | Public boards, organization workspaces |
| Interaction | Drag within/between columns, Move to Column and Move Up/Down alternatives (D01, D02) | Bulk editing (except archive completed) |
| Notifications | Assignment, mention/comment, due-soon (local), invitation, membership, opt-in activity, quiet hours, mute (D07) | Email digest, SMS |
| Offline | Read cached projects, queue edits, reconnect and sync, conflict presentation (D08, D12) | Fully peer-to-peer operation |
| Claude | Read tools, proposals, confirm/cancel, audit log, loopback transport (D15) | Autonomous destructive actions |
| Hosting | Mac mini, Docker Compose, PostgreSQL, local files, encrypted backups, restore drill, Tailscale HTTPS ingress (D10, D11, D21) | Multi-region high availability |

## 4. User stories

Roles: owner, admin, editor, viewer, invitee, Claude via MCP. "Member" means any of the first four.

### Accounts

**US-01** As an invitee, I want to sign in with Apple so that I need no password.
- After sign-in the Keychain holds only SharedKanban access and refresh tokens; no Apple identity token is stored (D16).
- Access tokens expire after 60 minutes; refresh tokens rotate with a 90-day sliding and 365-day absolute limit, and reuse after the 60-second grace window revokes the session (D16).

**US-02** As a member, I want to see and revoke my sessions so that a lost iPad cannot keep access.
- Settings lists each installation with device name and last use; revoking one closes its WebSockets within 60 s (D13, D16).
- "Sign out everywhere" keeps the current session only when `keepCurrent` is true (D16).

**US-03** As an owner, I want to delete my account so that my data is removed and my projects are handed over.
- For each owned project with other members I must pick Transfer or Delete, else 409 `owned_projects_require_transfer` (D09).
- Deletion requires a fresh Sign in with Apple bound to a 5-minute server challenge; a stale or wrong-subject token returns 401 `reauth_required` (D09).
- After 204 the user reads "Deleted member", all sessions are revoked, and my comments and attachments remain in shared projects (D09).

### Projects

**US-04** As an editor, I want to create a project from a template so that the board is usable at once.
- Home Projects yields Backlog, To Do, In Progress, Complete; Restaurants to Try yields Backlog, Scheduled, Complete; Custom accepts 3 or 4 names of 1–40 characters (D19).
- Project and columns are created in one transaction with the last column as done column (D19).
- Creation is unavailable offline; the control is disabled with a footnote (D08).

**US-05** As an owner or admin, I want to rename or archive a project so that finished boards leave the list.
- Archived projects are read-only: every mutation except unarchive, leave, member removal and notification settings returns 409 `project_archived` (D03).
- Rename requires `expectedVersion`; a stale same-field edit returns 409 `version_conflict` (D05).

**US-06** As an owner, I want to delete or restore a project so that a mistaken tap is recoverable.
- Delete hides the project for everyone immediately, revokes pending invitations and drops subscriptions (D03).
- Settings → Deleted projects offers Restore for 30 days; the worker purges afterward (D03).
- Admins see "Only the owner can delete this project" (D03).

**US-07** As an owner, I want to transfer ownership so that I can leave without deleting the board.
- The target must be an existing member; in one transaction they become owner and I become admin, after a confirmation alert naming them (D03).
- An owner leaving with other members present gets 409 `owner_must_transfer`; the UI shows Transfer ownership and Delete project instead of Leave (D03).

### Columns

**US-08** As an owner or admin, I want to add, rename, recolor, reorder and archive columns so that the workflow matches our habit.
- A project always has 3 to 4 active columns: a fifth column or archiving down to two returns 422 `column_count_invalid`; Add Column is disabled at 4 with the footnote "Projects have three or four columns in this version" (D19).
- Archiving a column holding active tasks requires `moveTasksTo`, else 422 `column_not_empty`; columns are never deleted (D19).

**US-09** As an owner or admin, I want a WIP limit so that bottlenecks are visible without being blocked.
- Limit is 1–99 or none; the header shows `count/limit`, amber at the limit, red above, with a warning symbol and the VoiceOver suffix "over the limit of N" (D19).
- Neither client nor server ever rejects a move for WIP; `task.moved` records `wipExceeded` (D06, D19).

**US-10** As an owner or admin, I want to choose which column marks tasks complete so that completion is unambiguous.
- At most one column has `isDone`; moving in sets `completedAt`, moving out clears it; existing values are never rewritten when the flag moves (D19).
- Archiving the done column clears the flag and the board shows "No column marks tasks complete" (D19).

### Tasks

**US-11** As an editor, I want to create and edit tasks so that work is captured.
- Title is required (1–200 characters), notes up to 20,000; a new task appends to the bottom of the first column unless created from another column's Add button (D01, D15, D19).
- Different-field concurrent edits merge (`X-Merged-From-Version`); same-field edits with different values return 409 `version_conflict` (D05).
- Assignee must be a current member, else 422 `validation_failed`; viewers can be assigned (D03, D05).

**US-12** As an editor, I want to set a due date so that I am reminded.
- Due date is an instant; picking only a day defaults to 09:00 local (D07).
- Overdue styling applies unless the task is completed or archived (D19).

**US-13** As an editor, I want to attach photos and documents so that context travels with the card.
- 25 MB per file, 20 per task, 2 GB per project, allowlisted types only; mismatch returns 415, oversize 413 (D14).
- Uploads run in a background URLSession with a 24-hour upload token and finish after suspension; downloads require membership (D14).

**US-14** As an editor, I want to comment and mention a member so that discussion stays on the card.
- Bodies up to 5,000 characters; `mentionedUserIds` are validated as members (D07, D15).
- I may edit or delete my own comment; owners and admins may delete anyone's (D03).

**US-15** As an editor, I want to archive tasks rather than delete them so that nothing is lost by accident.
- `DELETE /v1/tasks/{id}` archives: the task leaves the board but stays searchable under an Archived filter; a repeat is a no-op with no event (D18, D19).
- Restore returns the task to the bottom of its column, or the first column if its column is archived (D06, D19).
- Permanent delete is owner/admin only, only for archived tasks, behind a confirmation alert; "Archive completed" is the only bulk action (D19).

### Collaboration

**US-16** As an owner or admin, I want to invite by link so that joining does not depend on an Apple email.
- The link is `https://<host>/invite#<token>` with a 256-bit single-use token stored only as a hash; expiry 1, 7 or 30 days, default 7 (D04).
- Admins invite editors and viewers; only the owner invites an admin (D03, D04).
- Creation is limited to 5 per project per hour; pending invitations are listable and revocable (D04).

**US-17** As an invitee, I want to preview and accept or decline so that I know what I am joining.
- After signing in I see project name, inviter, role and expiry with Accept and Decline (D04).
- Expired, revoked, used or declined tokens return 410 `invitation_unavailable` with a reason; unknown tokens return 404 (D04).
- If the inviter can no longer grant the role, acceptance fails with `inviter_unavailable`; an existing member gets `alreadyMember: true` and the token is consumed (D04).
- The inviter is notified on accept and decline (D07).

**US-18** As an owner or admin, I want to change roles and remove members so that access matches trust.
- Admins manage editors and viewers; only the owner promotes, demotes or removes admins (D03).
- Removal takes effect on the next request: REST returns 404, the socket receives `unsubscribed` within 1 s, queued offline operations fail, and assignments are cleared (D03, D08, D13).
- A role change arrives as `member.role_changed` and the client drops edit affordances immediately (D03).

**US-19** As a member, I want realtime updates and an activity feed so that we see the same board.
- Committed events reach subscribed devices over WSS `/v1/realtime`; nothing uncommitted is broadcast (D13).
- Events apply only in sequence order; a gap triggers REST catch-up; a gap over 2,000 events triggers a snapshot reload (D08, D13).
- The feed resolves names at render time so "Deleted member" appears after deletion (D17).

### Drag and non-drag movement

**US-20** As an editor, I want to drag cards within and between columns so that reordering is direct.
- On iPhone a drop on a column pill appends to that column; a drop on a card row inserts before or after it with a 2pt insertion bar (D01).
- Moves are optimistic and submit `destinationColumnId`, `afterTaskId`, `beforeTaskId`, `expectedTaskVersion` and a fresh `operationId` (D06).
- On rejection the card animates back and a 4-second banner states the reason, also announced to VoiceOver (D01).
- Concurrent drops into one gap both succeed in deterministic order (D06).

**US-21** As a viewer using VoiceOver or a keyboard, I want non-drag movement so that every move is reachable without a gesture.
- Every card's ellipsis button, context menu and accessibility actions offer Move to Column…, Move to Top, Move Up, Move Down and Move to Bottom (D01).
- On iPad, Cmd-Option arrows reorder and change column; Cmd-1 to Cmd-4 select columns; shortcuts are off while a detail text field has focus (D02).
- Viewers see the actions disabled; the server authorizer is the real boundary (D03).

### Notifications

**US-22** As a member, I want useful notifications with sensible defaults so that I am not spammed.
- Defaults: invitation, assignment, comment, mention and membership on; activity off; the actor never gets a push for their own event (D07).
- Per-project mute suppresses every category except membership events about me; activity pushes are throttled to one per project per 5 minutes (D07).
- Exactly one delivery per (event, user) (D07).

**US-23** As an assignee, I want due reminders so that deadlines are not missed.
- Reminders are local, scheduled for my assigned tasks and my unassigned created tasks; lead time none, 1 hour or 1 day (default 1 day); at most 50 pending (D07).
- Quiet hours shift local reminders to the window end; triggers are rescheduled on time-zone change and after every sync (D07).

### Offline and sync

**US-24** As an editor, I want to work offline so that edits are kept until I reconnect.
- Projects opened in the last 30 days are cached; task create, edit, move, archive, restore, comments and column rename/color/WIP queue in a durable outbox (D08).
- Replay is strict FIFO, one operation in flight; an "Offline, N changes waiting" capsule shows while the outbox is non-empty (D08).
- Membership, invitations, project creation, column structure and uploads are online-only (D08).

**US-25** As an editor, I want conflicts shown plainly so that nothing is lost silently.
- A 409 `version_conflict` keeps my draft visible, shows "Changed by <name> <time>", and offers Keep Mine (resubmit with the current version and a new operationId) or Use Server (D05, D08).
- For notes only, Compare shows server and local text side by side with the local text editable (D08).
- A rejected move shows no dialog; the card snaps to the server position with the banner (D08).
- Operations older than 30 days fail as "too old to apply" (D08, D18).

### Claude via MCP

**US-26** As a member, I want Claude to read my projects so that I can ask questions about them.
- A connection token is created in Settings → Connect Claude with chosen scopes (default `projects:read`, `tasks:read`), shown once, 90-day expiry, revocable (D15).
- Read tools run under my permissions; cross-project ids return not found (D15).

**US-27** As Claude via MCP, I want to propose a change and have the user confirm it so that nothing applies without review.
- Every mutation tool returns a diff, risk level, required scope, 10-minute expiry and a single-use confirmation token; nothing auto-applies (D15).
- `confirm_change` re-authorizes, checks captured versions (409 `stale_proposal` if changed), applies through the normal services and returns the sequence; a second confirm returns 409 `change_already_applied` (D15).
- `cancel_change` marks the proposal cancelled and writes no project event (D15).
- Invitations and removals require `members:manage`, never granted by default (D15).

**US-28** As an owner, I want an audit log of Claude's calls so that I can see what it did.
- Every call, including denials, is logged with tool, redacted arguments and outcome, retained 1 year, visible in Settings → Claude activity (D15).
- Confirmed changes appear in the feed as "<Name> via Claude" (D15).

### Hosting and operations

**US-29** As an owner, I want the server to run unattended with tested backups so that losing the Mac mini does not lose the board.
- `pg_dump` every 6 hours plus restic to an external SSD; nightly copy to iCloud Drive; retention 14 daily / 8 weekly / 12 monthly (D21).
- A monthly automated restore drill verifies row counts and attachment hashes; the result shows in Settings → Server status (D21).
- A push alert fires when the last backup is older than 30 hours, a check or drill fails, or free space is under 20 % (D21).

**US-30** As a member, I want remote access without exposing the server so that the app works away from home.
- Production uses Tailscale with `tailscale serve` TLS in front of a loopback-only API; nothing listens on the public internet (D10).
- Every request still carries a bearer token and passes project authorization (D10).

## 5. Kanban rules

1. Every project starts with an ordered workflow of 3 or 4 columns from a template (D19).
2. New tasks enter the first column at the bottom unless created from another column (D01, D19).
3. Moving a task changes its column, sets or clears `completedAt` when the done column is involved, and writes one `task.moved` event (D06, D19).
4. Card order inside a column is explicit, shared, and stored as fractional-indexing ranks sorted byte-wise (D06).
5. WIP limits warn, never block, on client and server (D19).
6. Completed tasks stay on the board with a checkmark, remain searchable, and may be archived later (D19).
7. The pull habit is encouraged only by the WIP indicator; no further nudges in release 1 (D19).

## 6. Non-goals

Everything in the deferred column of section 3, plus, never in release 1: CRDT or operational-transform merging (D05); automatic text merge (D08); multiple owners (D03); a per-task completion checkbox (D19); 2 or 5+ active columns (D19); badge counts (D07); server-scheduled due reminders (D07); JWT app tokens or App Attest (D16); MCP auto-apply or in-app approval mode (D15); remote MCP OAuth before Phase 9b (D15); attachment blobs in PostgreSQL, or SVG, HTML, zip, OOXML and CSV attachments (D14); multiple scenes (D02); any queue besides PostgreSQL (D11).

## 7. Release acceptance criteria

| # | Criterion | Verification |
|---|---|---|
| 1 | Two users sign in with Apple on their own devices; each sees only their own projects | Two-device manual; automated cross-project 404 tests |
| 2 | A second user joins via invitation link; expired, revoked and reused tokens are refused with the right reason | Automated lifecycle tests (D04); scenario 1 |
| 3 | Different-field edits merge; a same-field conflict offers Keep Mine / Use Server / Compare and never loses the draft | Automated ConcurrencyGate tests; scenario 3 |
| 4 | 100 consecutive cross-column drags per device apply without snap-back absent a conflict; every rejection shows the banner | Automated move tests incl. 1,000-insert renormalization (D06); scenario 2 |
| 5 | A 20 MB photo uploads in the background, completes after suspension, hash verified | Automated resume and hash tests; scenario 4 |
| 6 | Assignment, comment, mention, invitation and membership pushes arrive exactly once; activity is off by default; due reminders fire locally across DST | Fake push provider tests; scenarios 5 and 7 |
| 7 | Claude lists tasks, proposes a move, is denied, re-proposes, is confirmed; audit log and feed show it | Phase 9 flow tests; scenario 7 |
| 8 | A removed member loses REST and socket access within 1 s | Automated eviction tests; scenario 6 |
| 9 | A backup restores into a clean stack with matching hashes and working project access | Operator drill; scenario 8 |
| 10 | No onboarding tutorial: a new user creates a project, adds a task and moves it unaided | Moderated two-person usability check at Phase 11 |
| 11 | Dynamic Type at accessibility XL without clipped titles; VoiceOver labels include title, column, assignee and due state | Snapshot tests at D02 widths; VoiceOver manual pass |

## 8. Manual test scenarios

1. **Create and invite.** On iPhone A create Home Projects and share an invitation. Expected: four columns in template order, Complete marked done; the share sheet carries an `https://…/invite#…` link with the 7-day expiry (D04, D19).
2. **Accept and move concurrently.** On iPad B accept, then both devices move different tasks at once. Expected: the preview shows project, inviter, role and expiry; both moves commit; each device sees the other's move within 2 s (D06, D13).
3. **Offline conflict.** Put A in airplane mode and edit notes; on B edit the same notes online; reconnect A. Expected: A shows "Changed by B …" with Keep Mine, Use Server and Compare; the draft is intact; Keep Mine commits a new version (D05, D08).
4. **Background upload.** Attach a photo, suspend the app for 5 minutes, resume. Expected: progress then complete; server hash matches; B shows the attachment count (D14).
5. **DST reminder.** Set a due date across a daylight-saving transition. Expected: the local reminder fires at the chosen wall-clock time; no duplicate on B unless B has synced (D07).
6. **Remove while open.** On A remove B while B has the board open. Expected: B receives `unsubscribed` within 1 s, the project disappears, and B's queued edits fail as "no longer a member" (D03, D13).
7. **Claude flow.** Ask Claude to list tasks, propose a move, cancel it, propose again, confirm. Expected: cancel writes no event; confirm writes `task.moved` with `via = mcp`; the feed reads "<Name> via Claude"; the audit log holds every call (D15, D17).
8. **Restore drill.** Run `deploy/backup/restore.sh <snapshot>` into the drill stack. Expected: `/ready` passes, row counts match, every attachment re-hashes correctly, a fake login reads a project (D21).

## 9. Glossary

- **Activity event**: one row in the per-project, gap-free log (D17).
- **Admin**: manages editors, viewers and columns; cannot delete the project or manage admins (D03).
- **Confirmation token**: single-use secret required by `confirm_change` (D15).
- **Done column**: the one column whose `isDone` flag sets `completedAt` (D19).
- **Draft overlay**: unsynced local values rendered over server values (D08).
- **Editor**: edits tasks, comments and attaches; cannot change membership or columns (D03).
- **expectedVersion**: the version a client last saw, required on every PATCH (D05).
- **Invitee**: a signed-in user holding an invitation token who has not yet joined (D04).
- **operationId**: UUID v4 idempotency key on every POST and PATCH (D18).
- **Outbox**: durable queue of pending offline operations (D08, D12).
- **Owner**: the single member who can transfer ownership, manage admins and delete the project (D03).
- **PendingAIChange**: a Claude proposal awaiting confirmation, expiring after 10 minutes (D15).
- **Rank**: base-62 fractional-indexing string giving a card's position (D06).
- **Sequence**: per-project monotonic counter on each event (D17).
- **Viewer**: read-only member who may be assigned and notified (D03).
- **WIP limit**: optional per-column cap of 1–99 that warns, never blocks (D19).
