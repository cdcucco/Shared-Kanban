# SharedKanban design system

This document fixes the SharedKanban interface on iPhone and iPad precisely enough that two engineers would build the same screens. It applies `docs/decisions/decision-register.md` (cited inline as D01 to D22) and `docs/product.md`; where the register is silent, this document decides and says so. Phase 4 builds this shell against the in-memory repository fake (D12); Phases 5 to 8 wire it to the server without changing the layout.

## 1. Design principles

1. **Calm.** One board, one column in focus on iPhone, plain system backgrounds. No dashboard look, dense toolbars, decorative gradients, tiny text or excessive borders.
2. **Apple-native.** System navigation, sheets, menus, materials, semantic colors, Dynamic Type text styles and SF Symbols. The current SDK's design language is adopted as-is; nothing here opts out of system chrome (D01, D20).
3. **Understandable without training.** Every screen answers, in this order: what is this task (title), which project am I in (navigation title), which stage is it in (column), what can I do next (Add task, Move, Done). Nothing else competes.
4. **Honest about state.** Pending, offline, conflict and error are visible but quiet; nothing is silently lost (D08); nothing is signalled by color alone.
5. **Every drag has a non-drag twin.** Menus, keyboard shortcuts and accessibility actions reach every outcome a drag can (D01, D02).

## 2. Navigation

### 2.1 iPhone: project list to board (D01)

The project list is a `List` with Active and Archived sections and a "+" menu (New project, Join with code). Tapping a project pushes the board.

The navigation bar shows the project title. Directly below it a persistent **column switcher bar** shows one pill per active column: title, active-task count, and `count/limit` when a WIP limit is set. The selected pill is accent-tinted and underlined; pills are at least 44pt tall; the bar exposes the adjustable trait and the selected state. The board is a horizontal `ScrollView` with a `LazyHStack`, view-aligned paging and `scrollPosition` bound to `BoardSelection.columnId`. Each page is the container width minus 24pt so the next column peeks. Cards live in a vertical `ScrollView` plus `LazyVStack`, never in `List`, so there are no swipe actions; the 44pt ellipsis button and the context menu replace them.

A drag reaches another column two ways: dropping on a switcher pill appends to that column (the guaranteed path), or dwelling 400 ms on a 44pt leading or trailing edge zone pages to the neighbor while the drag continues (built last, removed if it cancels the drag on a real device). Details in section 7.

```text
┌──────────────────────────────────────┐
│ ‹ Projects     Home Projects      +  │  navigation bar
│ [Backlog 4] [To Do 3] [In Prog 2/2!] │  switcher bar (pills, 44pt)
│ ─────────── ^ selected ───────────── │
│ To Do                         3   …  │  column header
│ ┌──────────────────────────────┐  ┆  │
│ │ Fix the gate latch           │  ┆  │  card: title (2–3 lines)
│ │ Buy hinge first              │  ┆  │  notes preview (1 line)
│ │ ⏰ Tomorrow   JD   📎1  💬2   │  ┆  │  meta row
│ └──────────────────────────────┘  ┆  │
│ ┌──────────────────────────────┐  ┆  │  ┆ = 24pt peek of next
│ │ Plan the garden beds         │  ┆  │    column
│ └──────────────────────────────┘  ┆  │
│                                   ┆  │
│ [ + Add task ]                       │
│ Offline · 3 changes waiting          │  status capsule (when needed)
└──────────────────────────────────────┘
```

### 2.2 iPad: NavigationSplitView (D02)

`NavigationSplitView` with `.balanced` style and `preferredCompactColumn`. The **sidebar** lists projects (Active, Archived) with an Account row at the bottom; visibility stays `.automatic` with the system collapse button. The **detail column** is `BoardView`, the same view as the iPhone board, in `.multiColumn` mode when the pane is wide enough and `.paged` otherwise (section 2.3). **Task detail** is a trailing `.inspector` (min 320, ideal 360, max 420pt) when the size class is regular and either `fitsAll(boardWidth − 360)` holds or the board is already paged; otherwise a large-detent sheet, so opening a task never flips the board mode. Both presentations share one `TaskDetailView`.

In multi-column mode columns sit in a centered row, each `clamp((w − 16 × (n + 1)) / n, 240, 380)` points wide with 16pt gutters, each scrolling vertically. One `BoardSelection { projectId, columnId, taskId, isDetailPresented }` survives width changes; `projectId` and `taskId` persist with `@SceneStorage`. Release 1 is single-scene.

```text
┌───────────┬───────────────────────────────────────────────┬────────────┐
│ Projects  │ Home Projects                          + ⌘N   │ Task       │
│───────────│ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────┐ │────────────│
│ ● Home    │ │Backlog 4…│ │To Do 3 … │ │In Prog   │ │Com…│ │ Fix the    │
│   Restau… │ │          │ │          │ │ 2/2 ! …  │ │    │ │ gate latch │
│           │ │┌────────┐│ │┌────────┐│ │┌────────┐│ │┌──┐│ │────────────│
│ Archived  │ ││card    ││ ││card    ││ ││card    ││ ││  ││ │ Notes      │
│   2024 …  │ │└────────┘│ │└────────┘│ │└────────┘│ │└──┘│ │ Due date   │
│           │ │┌────────┐│ │          │ │          │ │    │ │ Assignee   │
│           │ ││card    ││ │ Nothing  │ │          │ │    │ │ Attachments│
│───────────│ │└────────┘│ │ here yet │ │          │ │    │ │ Comments   │
│ Account   │ │ + Add    │ │ + Add    │ │ + Add    │ │ +  │ │ Activity   │
└───────────┴───────────────────────────────────────────────┴────────────┘
   sidebar          board, 4 columns of 240–380pt, 16pt gutters        inspector
```

### 2.3 Compact iPad windows (D02)

The board measures its own pane with `onGeometryChange`; device idiom and `UIScreen` are never consulted. With `n` active columns, `fitsAll(w) = w ≥ n × 240 + 16 × (n + 1)`: 784pt for three columns, 1040pt for four.

| Condition | Sidebar | Board mode | Task detail |
|---|---|---|---|
| Regular size class, `fitsAll(boardWidth)` | Split-view sidebar | `.multiColumn` | Inspector if `fitsAll(boardWidth − 360)`, else sheet |
| Regular size class, does not fit (one-half Split View with 4 columns, narrow Stage Manager) | Sidebar still available | `.paged` (iPhone board in the detail column) | Inspector (board already paged) |
| Compact size class (Slide Over, one-third Split View) | Collapsed into stack navigation | `.paged` | Sheet |

There is no intermediate two-column mode. Mode transitions animate only when Reduce Motion is off. If `.inspector` proves unreliable in compact windows at Phase 4, the fallback is `navigationDestination` in compact width (D02).

```text
Slide Over / one-third (compact)         One-half with 3 columns (regular, fits at 784pt)
┌──────────────────────┐                 ┌──────────────────────────────────────────┐
│ ‹ Projects  Home   + │                 │ ☰ Home Projects                      +   │
│ [Backlog][To Do][In…]│                 │ ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│ To Do           3  … │                 │ │Backlog 4│ │Sched. 2 │ │Complete │    │
│ ┌──────────────┐  ┆  │                 │ │┌───────┐│ │┌───────┐│ │┌───────┐│    │
│ │ card         │  ┆  │                 │ ││card   ││ ││card   ││ ││ ✓ card││    │
│ └──────────────┘  ┆  │                 │ │└───────┘│ │└───────┘│ │└───────┘│    │
│ [ + Add task ]       │                 │ │ + Add   │ │ + Add   │ │ + Add   │    │
└──────────────────────┘                 └──────────────────────────────────────────┘
```

## 3. Board surface

### 3.1 Column header

One `ColumnHeader` component serves both modes (on iPhone it tops each page, under the switcher bar). Left to right:

- **Title** (`.headline`, up to 2 lines). Owner and admin tap it to rename inline (1 to 40 characters, D19); others see static text. A 4pt bar in the column's color token (section 8.4) runs under the title, which is always present so color never identifies a column alone.
- **Count**: active tasks in `.subheadline` secondary; `count/limit` with a WIP limit.
- **WIP indicator** (D19): plain below the limit; amber (`.orange`) with `exclamationmark.triangle` at the limit; red (`.red`) with the same symbol above it. VoiceOver appends "over the limit of N". Nothing is ever blocked.
- **Done mark**: `checkmark.circle` on the `isDone` column, labelled "marks tasks complete".
- **Overflow** `ellipsis.circle` (44pt), owner/admin only (D03), in order: Rename, Color, WIP Limit…, Mark as Done Column / Unmark Done Column, Columns… (opens Project settings → Columns; iPad only, D19), Archive Column.

**Add task** is a full-width tinted button (`plus.circle`) at the bottom of every column; the toolbar "+" and Cmd-N create in the selected column, the visible page on iPhone and the first column by default on iPad (D01, D02, D19). New tasks append to the bottom. The button is hidden for viewers and disabled while the project is archived.

### 3.2 Drop target feedback (D01)

A targeted column tints its background `Color.accentColor.opacity(0.08)` (0.16 under Increase Contrast). The insertion position is a 2pt rounded accent bar (3pt under Increase Contrast) above or below the hovered row, chosen by pointer y against the row midpoint. A targeted pill tints like the selected pill. `.sensoryFeedback(.selection)` fires on target change, `.success` on commit. When a drop would exceed a WIP limit, the pill and insertion bar show `count/limit` in the warning color with `exclamationmark.triangle`, VoiceOver appends "over the limit of N", and the drop still proceeds.

### 3.3 Empty states

| Where | Copy |
|---|---|
| Empty column | "Nothing here yet. Tap Add task or move a card here." (viewer: "Nothing here yet.") |
| Empty project list | "No projects yet." with Create a project and Join with code |
| Archived tasks, empty | "No archived tasks." |
| Activity feed, empty | "No activity yet." |

### 3.4 Column limits (D19)

A project always has three or four active columns. **Add Column** (Project settings → Columns, owner/admin) is disabled at four with the footnote "Projects have three or four columns in this version." **Archive Column** is disabled at three with the footnote "A project needs at least three columns." and while the column holds active tasks, where "Move all tasks to…" is offered first and performs the `moveTasksTo` archive. The New Project sheet shows Home Projects, Restaurants to Try and Custom as cards with a tiny column preview, then an editable list of names; Custom allows a fourth row, never a fifth. Archiving the done column shows the board footnote "No column marks tasks complete." until one is chosen.

### 3.5 Reorder and rename flows

Rename: inline field; Return saves, Escape cancels; blank is refused. Reorder: Project settings → Columns, a `List` with drag handles plus Move Up / Move Down menu items; saving sends the full ordered id list with `expectedProjectVersion` (D06). Rename, color and WIP changes work offline; add, archive and reorder are online-only and disabled offline with the footnote "Available when online." (D08).

## 4. Card information hierarchy

Order inside a card, top to bottom, 12pt inner padding, 8pt between rows:

1. **Title** (`.body`, primary, 2 lines, 3 at accessibility sizes, tail truncation). A completed task shows `checkmark.circle.fill` before a secondary-colored title (D19).
2. **Notes preview** (`.subheadline`, secondary, 1 line, first non-empty line; hidden when notes are empty).
3. **Meta row** (`.footnote`), populated items only: due date, assignee, attachment count, comment count, sync indicator.

Due date treatments (the register is silent on the upcoming threshold; 48 hours is decided here):

| State | Rule | Treatment |
|---|---|---|
| Neutral | more than 48 h away | `calendar`, secondary text, short date ("Oct 14") |
| Upcoming | within the next 48 h | `clock`, primary text, medium weight ("Tomorrow 9:00") |
| Overdue | past `dueAt`, not completed | `exclamationmark.circle.fill`, red text, "Overdue" before the date |
| Completed | task in the done column | neutral regardless of date (D19) |

**Assignee**: a 24pt circle (28pt at accessibility sizes) with up to two initials (`.caption2` semibold) on a secondary fill; "Deleted member" shows `person.slash` (D09). **Counts**: `paperclip` and `bubble.left` with numbers, omitted at zero. **Sync/error indicator**, only when needed (D08): `clock.arrow.circlepath` with VoiceOver value "waiting to sync" while an operation is pending; amber `exclamationmark.triangle.fill` for a conflict; red `xmark.octagon` for a failed operation. One indicator at a time; conflict and failed outrank pending.

Never on the card: full notes, comment text, activity, creation or update timestamps, versions, identifiers, member lists, the column title, priority (does not exist), progress bars, or a color fill as the only status cue.

## 5. Task detail

Sections in order: **Title** (multi-line field), **Notes** (`TextEditor`, grows with content), **Due date** (treatment from section 4; `DatePicker` sheet; a day-only pick defaults to 09:00 local, D07; Clear), **Assignee** (picker over all members including viewers, D03; None), **Attachments** (thumbnail, name, size, progress with Retry and Cancel; Add from Photos or Files; size and type errors inline, D14), **Comments** (newest at the bottom, composer above the keyboard; mentions via a member picker fill `mentionedUserIds`, D07), **Activity** (per-action sentences, consecutive events by one actor within a minute coalesced, D17). The toolbar carries the ellipsis with the full move menu (section 7.4); viewers get only Close.

**Edit and save.** Edits are optimistic (D08). Title and notes save when editing ends (focus leaves, Done, dismissal, or the app backgrounds); each save is one PATCH carrying only the changed field with `expectedVersion` and a fresh `operationId` (D05, D18). Due date and assignee save on choice. There is no Save button and no unsaved-changes alert; the draft overlay keeps the user's text visible while pending, and a pending field shows the clock glyph after its label.

**Conflict presentation (D05, D08).** On 409 `version_conflict` a banner appears above the conflicting section: title "Changed by <name> <relative time>", the server message as detail ("Sam changed the notes while you were editing."), and **Keep Mine** (resubmit the draft against the current version under a new operationId), **Use Server** (discard the draft), and, for notes only, **Compare**, the manual merge: server and local text side by side, stacked in compact width, local text editable, Save submits a new operation. 409 `task_archived` offers **Restore and apply** or **Discard**. A 404 drops the change with the toast "This task was deleted." The card shows the amber conflict glyph meanwhile. Rejected moves never show a dialog.

## 6. Interaction states catalog

| State | Visual treatment | VoiceOver | Reduced Motion variant |
|---|---|---|---|
| Idle | Card on `secondarySystemGroupedBackground`, 12pt radius, no border (1pt `separator` border under Increase Contrast) | Label per section 9 | Same |
| Pressed | `tertiarySystemFill` overlay for the press; `.hoverEffect(.lift)` on pointer | No change | Same, no scale |
| Dragging (preview) | System drag preview at 0.9 opacity with system shadow; source row stays at 0.4 opacity | Announcement "Dragging <title>"; VoiceOver users use the Move actions | No lift spring |
| Valid drop target | Column tint 0.08 accent (0.16 high contrast); targeted pill tinted like selected | Announcement "Drop to move to <column>" on target change | Instant tint |
| Insertion indicator | 2pt (3pt high contrast) accent bar between rows, slides with the pointer | Announcement "Between <above> and <below>" or "At the bottom" | Bar crossfades |
| Pending (optimistic) | Card at its new place at once; `clock.arrow.circlepath` in the meta row | Value "waiting to sync" | Same |
| Syncing | "Syncing…" capsule under the navigation title with an indeterminate `ProgressView` | Capsule label "Syncing" | No capsule slide-in |
| Queued offline | "Offline · N changes waiting" capsule; pending cards keep the clock glyph; online-only controls disabled with footnotes | Static text; count changes announced politely | Same |
| Conflict | Amber `exclamationmark.triangle.fill` on the card; banner in the detail (section 5) | Card value "needs your attention"; banner first in the detail | Banner appears without slide |
| Rejected move | Card springs back to the server position; 4 s bottom `safeAreaInset` banner with the reason (section 10) | Banner posted as an Announcement | Card crossfades back; no banner motion |
| Error | Failed operation: red `xmark.octagon` on the card, detail banner with reason, Retry and Discard; screen-level failure: centered message with Retry | Card value "could not be saved"; banner first | Same |
| Read-only (viewer) | No Add task, drag, ellipsis or edit fields; footer "You can view this project." | Cards open a read-only detail; no move actions | Same |
| WIP-exceeded warning | `count/limit` amber at the limit, red above, with `exclamationmark.triangle`; pill mirrors it | Suffix "over the limit of N" | Same |
| Archived project | Read-only board with the banner "This project is archived." and Unarchive for owner/admin (D03) | Banner first in reading order | Same |

Every state carries a symbol or text in addition to color.

## 7. Drag and drop and its non-drag alternatives

### 7.1 Starting a drag (D01, D02)

Touch: long-press then movement starts the drag; long-press without movement opens the context menu. Pointer or trackpad on iPad: a direct drag starts immediately. The payload is `TaskDragItem {projectId, taskId}` under the custom type `com.sharedkanban.task` with no text representation; "Copy title" is the explicit alternative.

### 7.2 Within-column reorder

Each card row is a drop destination; before/after comes from pointer y against the row midpoint and is shown by the insertion bar. 44pt top and bottom zones in each column nudge the vertical scroll while targeted, because SwiftUI drag has no vertical auto-scroll.

### 7.3 Cross-column move

iPhone (paged): drop on a switcher pill to append, or dwell 400 ms on an edge zone to page and keep dragging. iPad (multi-column): drag into the other column's body to append or onto a row for an exact position; there are no pills. A drop whose `projectId` differs from the board's is refused; if a realtime event archives the dragged task or the destination column mid-drag, the drop is refused and the rejection banner shows.

Every drop and menu move computes `(destinationColumnId, afterTaskId, beforeTaskId)` from the cached column order and calls one `MoveCoordinator.move(...)`, which applies the move optimistically and submits the D06 contract with a fresh `operationId`. On 409 `neighbors_changed` the client retries once with neighbors from `details.currentOrder`, then snaps to the server state (D06).

### 7.4 Context menu and ellipsis menu contents (D01)

Identical in the context menu, the ellipsis button, the detail toolbar and `accessibilityAction`s, in this order: **Move to Column…** (sheet listing active columns with the current one checked and a Top / Bottom placement choice), **Move to Top**, **Move Up**, **Move Down**, **Move to Bottom**, divider, **Assign…**, **Due Date…**, divider, **Copy title**, **Archive** (destructive). Move Up at the top and Move Down at the bottom are disabled, never hidden. Viewers get only Copy title. Menu archive is not confirmed because it is restorable (D19); permanent delete from Archived tasks is owner/admin and always confirmed.

### 7.5 Keyboard shortcuts on iPad (D02)

| Shortcut | Action |
|---|---|
| Cmd-N | New task in the selected column |
| Cmd-1 … Cmd-4 | Select column 1 to 4 |
| Arrow keys | Move card focus within and across columns |
| Cmd-Option-Up / Down | Move focused card up or down |
| Cmd-Option-Left / Right | Move focused card to the adjacent column (appended at the bottom) |
| Return | Open task detail |
| Cmd-Delete | Archive focused task, with confirmation |
| Escape | Close inspector or sheet |

Cards are `.focusable()` under one `@FocusState<UUID?>` shared by Full Keyboard Access and VoiceOver; board shortcuts are disabled while a detail text field has focus. Focus uses the system ring.

## 8. Visual language

### 8.1 Color and materials

Only Apple semantic colors: `systemBackground` for boards and detail, `secondarySystemGroupedBackground` for cards, `systemGroupedBackground` for column wells in multi-column mode, `label` / `secondaryLabel` / `tertiaryLabel` for text, `separator` for the rare divider, `Color.accentColor` (system default; no brand tint in release 1) for selection, drop tint and primary buttons, `.orange` for warnings, `.red` for overdue, errors and destructive actions. Bars and capsules use the SDK's system materials (D01). No custom shadows, gradients or illustrations.

### 8.2 Typography

| Element | Text style |
|---|---|
| Project title | navigation title (large on the list, inline on the board) |
| Column title, pill title | `.headline`; pill count `.subheadline` |
| Card title | `.body`, 2 lines (3 at accessibility sizes) |
| Notes preview, detail values | `.subheadline` secondary |
| Due date, counts, status capsule, footnotes, empty states | `.footnote`; symbols `.imageScale(.small)` |
| Initials | `.caption2` semibold in a 24pt circle (28pt at accessibility sizes) |
| Banners | `.subheadline` on a `.regularMaterial` capsule |

No fixed point sizes anywhere; every style scales with Dynamic Type, including inside pills and buttons.

### 8.3 Spacing and radii

Spacing scale 4, 8, 12, 16, 24, 32pt. Card padding 12pt; 8pt between cards; 16pt gutters and column gaps (D02); 24pt page peek (D01). Radii: cards 12pt continuous, column wells 16pt, pills capsule, insertion bar 1pt, sheets and inspectors system. Column widths 240 to 380pt (D02). Minimum control size 44 × 44pt.

### 8.4 Column accent color

`columns.color` is a named token, not hex (D19). Token set, decided here: `gray`, `red`, `orange`, `yellow`, `green`, `teal`, `blue`, `indigo`, `purple`, mapped to the same-named system colors so dark mode and Increase Contrast adapt automatically. Template defaults: `gray` Backlog, `blue` To Do and Scheduled, `orange` In Progress, `green` Complete. The color appears only as the 4pt header bar and the pill underline, never as a card fill or the only distinction between columns; the Color picker names each token beside its swatch.

### 8.5 Dark mode and high contrast

Dark mode needs no custom palette; cards stay distinguishable from the board through the grouped background pairing, not borders. Under Increase Contrast cards gain a 1pt `separator` border, the drop tint doubles to 0.16, the insertion bar is 3pt, and secondary pill text becomes primary. Under Reduce Transparency material bars fall back to opaque system backgrounds (verify at Phase 4 that nothing custom sits on a material).

## 9. Accessibility rules

1. **Dynamic Type without clipped titles.** Card titles get three lines at accessibility sizes, column titles two, pill titles wrap and the switcher bar grows; no title uses `lineLimit(1)`. The 240pt column must hold two or three title lines at accessibility XL, tested at every D02 width.
2. **VoiceOver card label**, exactly: "<title>, in <column>, assigned to <name>, due <relative>", suffixed ", overdue" or ", completed" as applicable; the value is "waiting to sync", "needs your attention" or "could not be saved" when applicable; hint "Double tap to open." Omit "assigned to" when unassigned and "due" without a date. Every card exposes the section 7.4 items as custom actions (D01).
3. **Keyboard and pointer** on iPad per section 7.5; hover lifts cards and highlights drop zones.
4. **Reduced-motion drag feedback**: no springs; the insertion bar crossfades; rejected moves crossfade back; mode changes do not animate (D01, D02).
5. **Minimum 44pt targets** for pills, the ellipsis, Add task, attachment rows and every toolbar item.
6. **Destructive confirmations** (`confirmationDialog` with a destructive button and Cancel): Cmd-Delete archive, permanent task delete, column archive with task moves, member removal, project deletion, ownership transfer naming the new owner (D03), session revoke, account deletion (D09). Menu archive is not confirmed because it is restorable.
7. **Color is never the sole status signal**: every colored state carries a symbol or a word.
8. **Announcements**: drop target changes, rejected moves and conflicts post `AccessibilityNotification.Announcement`; the offline capsule is a static element, never an announcement storm.
9. **Reading order**: banners, switcher bar, column header, cards, Add task.

## 10. Copy and tone

Short, plain, no jargon, no blame, no exclamation marks. Sentences name the other person when the server knows who acted. Exact wording of key messages:

| Situation | Exact copy |
|---|---|
| Rejected move, `position_conflict` | "Moved by <name> <relative time>. Your move was not applied." (D01) |
| Rejected move, `neighbors_changed` after the retry | "The column changed. Your move was not applied." |
| Rejected move, `task_archived` or archived column | "This task was archived. Your move was not applied." |
| Offline capsule | "Offline · 3 changes waiting" (singular "1 change waiting"; none pending: "Offline") (D08) |
| Reconnecting capsule, after 5 s | "Reconnecting…" (D13) |
| Conflict banner | Title "Changed by <name> <relative time>"; detail is the server message, e.g. "Sam changed the notes while you were editing."; buttons Keep Mine, Use Server, Compare (D05, D08) |
| Operation too old | "Too old to apply. This change was made more than 30 days ago." (D08) |
| Invitation preview | Title "You've been invited"; "<Inviter> invited you to <Project name> as <role>. Expires <date>."; Accept, Decline (D04) |
| Invitation share message | "<Inviter> invited you to the SharedKanban project “<name>”. Open this link on your iPhone or iPad (expires <date>)" (D04) |
| Invitation unavailable | "This invitation has expired." / "…was revoked." / "…was already used." / "…was declined." / "This project is no longer available." (D04) |
| Delete account, consequences sheet | Title "Delete Account"; "Your account and sign-in are deleted right away and cannot be recovered. Comments, attachments, tasks and activity you added to shared projects stay there, shown as Deleted member. Your Sign in with Apple authorization is revoked." Per owned project: "Transfer to…" or "Delete project"; solo-owned: "Will be deleted permanently."; if pending: "N unsynced changes will be discarded." (D09) |
| Delete account, confirmation | Destructive "Delete" and Cancel; if the Apple sheet is cancelled: "Deleting your account requires signing in with Apple again." (D09) |
| Forced sign-out | "Signed out for your safety" (D16); Apple revocation: "Sign in again to continue" (D09) |
| Deep link to removed content | "No longer available" (D07) |
| Owner leaving | Leave becomes "Transfer ownership" and "Delete project"; footnote "Only the owner can delete this project." (D03) |
| First-run network hint | "The app needs the Tailscale VPN to be on." (D10) |

## 11. Screens inventory for Phase 4

Every screen is built against the in-memory fake with Home Projects and Restaurants to Try data (D19) and has a preview per listed state.

| Screen | States |
|---|---|
| Sign in | idle (one Sign in with Apple button, one sentence of purpose), in progress, failed ("Sign in didn't complete. Try again."), revoked |
| Project list | populated, empty, refreshing ("Updated N min ago", D12), offline, archived section, Deleted projects entry (owner) |
| New project | template cards, editable names with the 3/4 rule, offline-disabled, creating |
| iPhone board | populated, empty column, WIP at and over limit, viewer, archived project, offline with pending, reconnecting, drop targeted, rejected-move banner |
| iPad board | 3 and 4 columns, paged in compact, inspector open, sheet detail, keyboard focus, hover lift |
| Task card | idle, pressed, dragging source, completed, overdue, upcoming, with and without assignee, counts, pending, conflict, failed |
| Task detail | editable, viewer, pending field, conflict banner (notes, title, due date), Compare view, archived with Restore, attachment uploading / failed / complete, comment with mention, coalesced activity |
| Move to Column sheet | current column checked, Top / Bottom placement, WIP warning |
| Project members and roles | list by role, role change where allowed, remove with confirmation, transfer ownership, leave and the owner-must-transfer variant |
| Invitation sharing | create (role, expiry 1 / 7 / 30 days), share sheet, pending list, revoke, offline-disabled |
| Invitation received | signed out, preview, accepting, already a member, each unavailable reason |
| Project settings → Columns | reorder, add at 3, add disabled at 4, archive disabled at 3 or with tasks, Move all tasks to…, color and WIP editors, done-column picker |
| Activity feed | populated, coalesced, empty, "via Claude" rows |
| Notification settings | categories and lead time, quiet hours, per-project mute and overrides, permission pre-prompt |
| Settings | account, sessions list, Delete Account flow, Connect Claude and Server status placeholders |
| Offline, syncing, conflict and error states | the capsules, banners and glyphs of section 6 in one preview gallery |

## 12. Manual design review checklist

The Phase 4 reviewer (Cowork on the Mac mini, with screenshots for the owner) checks every item at 320, 375, 507, 744, 834, 1024, 1194 and 1366pt, in Slide Over, one-third and one-half Split View, full screen and a narrow Stage Manager window (D02).

1. All columns appear only when `fitsAll` holds; otherwise the board is paged with a visible 24pt peek; no two-column band exists.
2. Opening a task never changes the board mode; inspector only in regular width where it fits, otherwise a sheet.
3. Dynamic Type at default, XL and accessibility XL: no clipped card, column or pill title; the 240pt column holds two to three title lines; nothing overlaps.
4. Dark mode and Increase Contrast: cards distinguishable from the board; symbols legible; under Increase Contrast card borders appear, drop tint and insertion bar strengthen, pill secondary text becomes primary.
5. Reduce Motion: no springs on drag, rejection or mode change; the insertion bar crossfades.
6. VoiceOver: card labels match section 9 word for word; custom actions list every move item; the switcher bar is adjustable; banners read first; target changes and rejected moves are announced.
7. Keyboard and pointer: every section 7.5 shortcut works with a visible focus ring and is disabled while a detail field has focus; hover lifts cards; pointer drag starts without a long-press.
8. Every drag outcome is reachable from the ellipsis menu, context menu and detail toolbar; Move Up / Down are disabled, not hidden, at the ends.
9. Viewer role: no edit affordances; cards open a read-only detail; the footer is shown.
10. WIP: `count/limit` amber at the limit, red above, with symbol and VoiceOver suffix; drops never blocked.
11. Column rules: Add Column disabled at four, Archive Column disabled at three or with tasks, footnotes present; Custom accepts exactly three or four names.
12. States gallery: offline, syncing, conflict with Keep Mine / Use Server / Compare, failed with Retry / Discard, rejected-move and archived-project banners all render and read correctly.
13. Copy: every string in section 10 appears verbatim; no color-only status anywhere.
14. Touch targets: every interactive element measures at least 44 × 44pt.
15. Previews exist for every row of section 11.
