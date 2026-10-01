# Shared Kanban for iOS and iPadOS

## Executive decision

Build a native SwiftUI app for iPhone and iPad, backed by a Swift/Vapor API, PostgreSQL, WebSockets, local attachment storage, and a separate MCP interface. Run the API, database, file storage, notification worker, and MCP service on the Mac mini. Use Apple only for Sign in with Apple and APNs push delivery, plus a narrowly scoped HTTPS tunnel or reverse proxy so the two users’ devices can reach the Mac mini away from home.

The first complete release should support shared projects, three- or four-column editable workflows, draggable cards, task titles and notes, members and roles, due dates, attachments, comments/activity, realtime updates, offline use, and configurable notifications. Claude should access the same application service layer as the iOS app through MCP tools; it should never receive raw database access.

The project should be built in vertical slices rather than asking Claude to generate the entire system in one pass. Anthropic recommends separating planning from implementation, keeping persistent project rules in `CLAUDE.md`, and giving Claude checks it can actually run, such as builds and tests.[^1][^2]

## Product definition

### Product goal

Create a simple shared Kanban app for small groups, especially a household, that feels native on Apple devices and remains under the owner’s control. A typical project might use:

- **Home Projects:** Backlog, To Do, In Progress, Complete.
- **Restaurants to Try:** Backlog, Scheduled, Complete.
- **Custom workflow:** Choose three or four starter columns, then rename and reorder them.

The core mental model should stay small:

- A **project** is a shared board with members and settings.
- A **column** is an ordered stage or category in that project.
- A **task** is a card with a required title and optional notes, due date, assignee, attachments, and comments.
- A **member** has a project role and notification preferences.
- An **activity event** records important changes for synchronization, notifications, and accountability.

### Release scope

| Area | Required in first complete release | Deliberately deferred |
|---|---|---|
| Accounts | Native Sign in with Apple, session management, account deletion | Password accounts, Google login |
| Projects | Create, rename, archive, select a 3- or 4-column template | Nested projects, portfolios |
| Columns | Built-in names, rename, reorder, color accent, optional WIP limit | Arbitrary workflow automation |
| Tasks | Title, notes, assignee, due date, attachments, comments, archive | Dependencies, recurring tasks, subtasks |
| Collaboration | Invite, accept, remove, roles, realtime changes, activity feed | Public boards, organization workspaces |
| Interaction | Drag within/between columns, accessible “Move to” alternative | Bulk editing |
| Notifications | Assignment, mention/comment, due-soon, invitation, configurable project activity | Email digest, SMS |
| Offline | Read cached projects, queue edits, reconnect and sync | Fully peer-to-peer operation |
| Claude | Read tools, proposed mutations, confirmed mutation tools, audit log | Autonomous destructive actions |
| Hosting | Mac mini, Docker Compose, PostgreSQL, local files, backup, HTTPS ingress | Multi-region high availability |

### Success criteria

The app is successful when both household members can sign in, join a shared project, make changes concurrently, drag tasks reliably, attach a photo or document, receive useful notifications, and ask Claude to make a clearly reviewed change. The interface should explain itself without an onboarding tutorial.

## Kanban design findings

A board should visualize a process rather than merely arrange notes. Trello describes lists as workflow stages or information categories and cards as tasks or ideas that move through the lists. Notion broadens the model by allowing boards to group items by status, assignee, priority, or other properties, while retaining drag-and-drop reordering. The proposed app should remain opinionated around workflow columns but preserve enough flexibility for household uses such as restaurants, shopping, maintenance, and trip planning.[^3][^4]

### Patterns worth adopting

| Product pattern | What it teaches | Application here |
|---|---|---|
| Trello | Boards, vertical lists, cards, drag-and-drop, a board menu, and board-level roles are easy to understand.[^5][^6] | Keep the hierarchy Project → Column → Task; put member and project controls in one predictable menu. |
| Notion | Columns can represent flexible properties; users can rename/reorder columns and choose what appears on a card.[^4] | Let users edit column titles and keep card surfaces concise. |
| Linear | Default workflows reduce setup work, while custom status names and ordering support different teams.[^7] | Provide polished three- and four-column templates, editable after creation. |
| Asana | Cards can expose due date, assignee, attachments, and collaboration details without opening every task.[^8] | Show only compact metadata on the card; open a detail view for notes and history. |

### Kanban rules

- Every project starts with an ordered workflow.
- New tasks enter the first column unless the user chooses another.
- Moving a task changes its workflow stage and records an activity event.
- Card order inside each column is explicit and shared.
- Optional work-in-progress limits may be configured per active column. WIP limits cap the number of items in a stage and make bottlenecks visible; they should warn rather than hard-block in a household app.[^9]
- Completing work should be easy, but completed cards should remain searchable and may be archived later.
- The app should support a “pull” habit: finish an in-progress item before pulling another from To Do. This should be encouraged through gentle WIP indicators, not rigid process enforcement.

## Experience design

### Navigation

Use a native adaptive SwiftUI layout:

- **iPhone:** A project list opens a board. The board shows one primary column at a time with horizontal paging or controlled horizontal scrolling. Each column scrolls vertically. A persistent project title and column switcher maintain orientation.
- **iPad:** Use `NavigationSplitView`: projects in the sidebar, the board in the main pane, and task details in an inspector or trailing pane when space permits. Apple describes split views as an appropriate way to show hierarchy and notes that iPad layouts must adapt to changing window widths.[^10]
- **Compact iPad windows:** Collapse progressively to the iPhone pattern rather than squeezing four unusable columns onto the screen.

### Board surface

Each column should have:

- Editable title.
- Task count and optional WIP indicator.
- “Add task” action.
- Drop target feedback.
- Empty state with a short instruction.
- Overflow menu for rename, reorder, WIP limit, and archive/delete when allowed.

Each card should show:

- Required title, limited to two or three lines.
- Optional one-line notes preview.
- Due date with neutral, upcoming, or overdue treatment.
- Assignee avatar or initials.
- Small attachment and comment counts.
- A subtle sync/error indicator only when needed.

Tapping a card opens a focused detail view with the complete notes editor, due date, assignee, attachments, comments, and activity. Trello similarly separates quick information on a card’s front from detailed content behind it.[^6]

### Drag and drop

Use SwiftUI’s native drag-and-drop APIs. Apple’s recommended pattern is a `Transferable` model with `draggable(_:)` and `dropDestination(for:action:isTargeted:)`.[^11][^12]

Required behavior:

- Long-press or direct pointer drag starts movement.
- Dragging within a column reorders the card.
- Dragging to another column changes stage and position.
- The destination visibly highlights and shows an insertion location.
- The UI updates optimistically, then reconciles with the server.
- If the server rejects the move, the card returns to the authoritative location and presents a plain-language message.
- Every card also offers **Move to Column…** and **Move Up/Down** actions for accessibility, keyboard use, and compact screens.

### Visual language

Use Apple-native typography, spacing, sheets, menus, materials, and semantic colors. Avoid a dashboard look, dense toolbars, decorative gradients, tiny text, and excessive borders. The interface should prioritize title, current project, current stage, and next action.

Accessibility requirements:

- Dynamic Type without clipped titles.
- VoiceOver labels that include task title, column, assignee, and due state.
- Keyboard and pointer support on iPad.
- Reduced-motion-safe drag feedback.
- Color never serves as the only status signal.
- Minimum comfortable touch targets and clear destructive confirmations.

## Collaboration model

### Project roles

| Role | Read | Edit tasks | Invite/remove | Edit workflow | Delete project |
|---|---:|---:|---:|---:|---:|
| Owner | Yes | Yes | Yes | Yes | Yes |
| Admin | Yes | Yes | Yes | Yes | No; may archive if permitted |
| Editor | Yes | Yes | No | No | No |
| Viewer | Yes | No | No | No | No |

Only owners and admins should add or remove members. Established products distinguish full members, limited observers/viewers, and board admins; board admins normally control membership and settings. A small household app does not need workspace-level administration.[^5][^13]

### Invitations

Do not depend solely on an email-address match, because Sign in with Apple can provide a private relay address. Use a project invitation record containing a cryptographically random, single-use token, expiration, intended role, inviter, and project. Present it through the system share sheet as a universal link; after signing in, the recipient reviews and accepts the invitation.

### Concurrent edits

Use the server as the authority and include `version`, `updatedAt`, and `updatedBy` on mutable records. Mutating requests include the expected version:

- Different-field edits can normally merge.
- Same-field conflicts return HTTP 409 with the current server record.
- Task moves are serialized in a database transaction.
- WebSocket events notify connected project members after commit.
- The activity log records actor, action, entity, timestamp, and compact before/after metadata.

The app should not implement a complex CRDT for this scale. Optimistic concurrency, transactions, and deterministic card positioning are easier to test and operate.

## Recommended architecture

### System map

```text
iPhone/iPad SwiftUI app
  ├─ AuthenticationServices (Sign in with Apple)
  ├─ SwiftData local cache + pending operation queue
  ├─ URLSession REST client
  ├─ WebSocket realtime client
  ├─ UserNotifications / APNs registration
  └─ Background URLSession attachment transfers
                 │ HTTPS / WSS
                 ▼
Public hostname or private VPN endpoint
                 │
                 ▼
Mac mini
  ├─ Vapor API and application service layer
  ├─ Realtime WebSocket hub
  ├─ Background notification/job worker
  ├─ MCP server adapter
  ├─ PostgreSQL
  ├─ Attachment volume
  └─ Encrypted backup process
                 │ outbound HTTPS
                 ├─ Apple identity/token endpoints
                 └─ Apple Push Notification service
```

### iOS and iPadOS client

Recommended components:

- SwiftUI for the interface.
- Observation-based state containers or a modest MVVM structure; avoid a large “clean architecture” framework.
- SwiftData as an offline cache, not as the shared source of truth.
- `AuthenticationServices` for native Sign in with Apple.
- `URLSession` with async/await for REST calls.
- `URLSessionWebSocketTask` for project event delivery.
- Keychain for app session tokens; Keychain provides encrypted storage intended for credentials and other small secrets.[^14][^15]
- `PhotosPicker` and `fileImporter` for attachments.
- Background `URLSession` file uploads so transfers can continue when the app is suspended; background upload tasks must be file-based.[^16][^17]
- `UserNotifications` for local due reminders and remote collaboration notifications.

### Mac mini backend

Use Swift and Vapor so both client and server share language knowledge and Codable DTO conventions. Vapor’s Fluent ORM recommends PostgreSQL among its supported drivers, and Vapor provides native WebSocket endpoints for two-way communication.[^18][^19]

Run the system with Docker Compose:

- `api`: Vapor REST and WebSocket service.
- `worker`: same codebase, separate command for APNs and maintenance jobs.
- `postgres`: durable database volume.
- `cloudflared` or a reverse proxy/tunnel process.
- Optional `backup`: scheduled encrypted database dumps and attachment snapshots.

Store attachments on a mounted Mac mini volume for the first release. Keep metadata, ownership, MIME type, byte size, hash, and storage key in PostgreSQL. Use random storage keys instead of user filenames, validate type and size, and never serve arbitrary filesystem paths.

Docker makes the deployment reproducible and can orchestrate the app, PostgreSQL, and related services on macOS. Keep an escape hatch for running the Vapor binary as a `launchd` service if Docker Desktop resource use becomes undesirable.[^20]

### Connectivity

The backend can remain physically hosted on the Mac mini, but remote iPhones need a secure route to it. Apple’s App Transport Security expects `URLSession` traffic to use HTTPS and blocks connections that fail its TLS requirements.[^21][^22]

Recommended order:

1. **Development:** local network address plus a development-only configuration.
2. **Private household beta:** Tailscale on both users’ devices and the Mac mini; no public API exposure.
3. **Convenient remote production:** a dedicated hostname through Cloudflare Tunnel or a carefully configured reverse proxy. Cloudflare Tunnel maps a public hostname to a local service over an outbound-only tunnel, avoiding inbound firewall rules.[^23][^24]
4. **Alternative:** Tailscale Funnel supplies a public HTTPS endpoint for a local service, but it makes that endpoint reachable from the public internet and therefore still requires robust application authentication.[^25][^26]

A tunnel is transport, not authorization. Every API request must still carry a valid app session and pass project-level access checks.

### Authentication flow

1. The native app requests Sign in with Apple with a nonce.
2. The app sends Apple’s authorization code, identity token, nonce, and app attestation context to the backend over HTTPS.
3. The backend verifies token signature, issuer, audience, nonce, and expiry; it exchanges the short-lived authorization code with Apple.
4. The backend maps Apple’s stable subject identifier to an internal user.
5. The backend returns a short-lived access token and a rotating refresh token for this app installation.
6. The app stores only its own session credentials in Keychain.

Apple documents that native apps use Authentication Services while app servers use the Sign in with Apple REST API to validate identity and authorization tokens. Name and email may only be available during the initial authorization, so store approved profile values at first sign-in rather than assuming Apple will return them later.[^27][^28]

Include in-app account deletion from the start. App Store apps that create accounts must let users initiate account deletion, and Sign in with Apple apps should revoke the associated tokens.[^29][^30]

### Notifications

Use two paths:

- **Local notifications:** personal due-date reminders calculated on-device when possible.
- **Remote notifications:** invitations, assignments, comments/mentions, and relevant project changes generated by the Mac mini worker and delivered through APNs.

For remote notifications, the app registers with APNs, forwards its device token to the backend, and the backend sends authenticated HTTP/2 requests to APNs. APNs remains an Apple cloud dependency even though the application backend and data stay on the Mac mini.[^31][^32]

Default notification policy:

- Invitation: on.
- Assigned to the user: on.
- Mention/comment on a participating task: on.
- Due soon and overdue: on, configurable.
- Every card move: off by default.
- Project-specific mute and quiet hours: available.

### Offline and synchronization

Treat offline behavior as an explicit subsystem:

- Cache user-visible projects, columns, tasks, members, and recent activity in SwiftData.
- Give every pending mutation a UUID idempotency key.
- Queue create/edit/move/comment operations when offline.
- Replay in creation order after authentication and connectivity return.
- Let the server return a monotonic project event sequence.
- On reconnect, request events after the last sequence; if retention was exceeded, refresh a complete project snapshot.
- Upload attachments only when online and show resumable state.

The first version should not claim seamless offline conflict resolution. It should preserve edits, surface conflicts clearly, and avoid silent data loss.

## Data model

| Entity | Important fields |
|---|---|
| User | `id`, `appleSubject`, `displayName`, `email`, `createdAt`, `deletedAt` |
| Session | `id`, `userId`, hashed refresh token, device name, expiry, revoked date |
| Project | `id`, `name`, `ownerId`, `archivedAt`, `version`, timestamps |
| ProjectMember | `projectId`, `userId`, `role`, notification settings, joined date |
| Invitation | `id`, `projectId`, token hash, role, inviter, expiry, accepted/revoked date |
| Column | `id`, `projectId`, `title`, `rank`, `color`, optional `wipLimit`, `version` |
| Task | `id`, `projectId`, `columnId`, `title`, `notes`, `rank`, `assigneeId`, `dueAt`, `completedAt`, `version` |
| Attachment | `id`, `taskId`, uploader, storage key, original name, MIME type, size, SHA-256 hash |
| Comment | `id`, `taskId`, `authorId`, body, edited date, deleted date |
| ActivityEvent | `id`, `projectId`, actor, action, entity type/id, metadata, sequence, timestamp |
| Device | `id`, `userId`, APNs token, environment, last seen, notification status |
| PendingAIChange | `id`, `userId`, project, proposed operations, expiry, confirmation state |

Use opaque UUIDs. Use a rank representation that permits insertion between neighbors; if ranks become too dense, renormalize one column inside a transaction. Never let clients directly set another project’s IDs without server ownership validation.

## API outline

### REST endpoints

```text
POST   /v1/auth/apple
POST   /v1/auth/refresh
POST   /v1/auth/logout
DELETE /v1/account

GET    /v1/projects
POST   /v1/projects
GET    /v1/projects/{projectId}
PATCH  /v1/projects/{projectId}
DELETE /v1/projects/{projectId}

POST   /v1/projects/{projectId}/invitations
POST   /v1/invitations/{token}/accept
GET    /v1/projects/{projectId}/members
PATCH  /v1/projects/{projectId}/members/{userId}
DELETE /v1/projects/{projectId}/members/{userId}

POST   /v1/projects/{projectId}/columns
PATCH  /v1/columns/{columnId}
POST   /v1/projects/{projectId}/columns/reorder
DELETE /v1/columns/{columnId}

POST   /v1/projects/{projectId}/tasks
PATCH  /v1/tasks/{taskId}
POST   /v1/tasks/{taskId}/move
DELETE /v1/tasks/{taskId}

POST   /v1/tasks/{taskId}/attachments/initiate
POST   /v1/attachments/{attachmentId}/complete
DELETE /v1/attachments/{attachmentId}
POST   /v1/tasks/{taskId}/comments
PATCH  /v1/comments/{commentId}
DELETE /v1/comments/{commentId}

POST   /v1/devices
DELETE /v1/devices/{deviceId}
GET    /v1/projects/{projectId}/events?after={sequence}
WS     /v1/realtime
```

### Move contract

A move request should include:

```json
{
  "operationId": "UUID",
  "destinationColumnId": "UUID",
  "beforeTaskId": "UUID or null",
  "afterTaskId": "UUID or null",
  "expectedTaskVersion": 12
}
```

The response returns the canonical task, updated column ranks if renormalization occurred, and the committed event sequence. This avoids transmitting arbitrary numeric positions and makes retrying an operation safe.

## Claude MCP connector

MCP is appropriate because it standardizes how an AI client discovers and calls application tools. The MCP adapter should call the same authorization and application services used by REST endpoints; it must not connect to PostgreSQL with unrestricted queries.[^33]

### Tool design

Read-only tools:

```text
list_projects
get_project
list_tasks
get_task
search_tasks
list_project_members
```

Mutation proposal tools:

```text
propose_create_task
propose_update_task
propose_move_task
propose_comment
propose_create_project
propose_invite_member
propose_remove_member
```

Commit tools:

```text
confirm_change(change_id, confirmation_token)
cancel_change(change_id)
```

The proposal response should show exact affected project, card, fields, old values, new values, and expiration. Destructive or membership-changing proposals always require explicit confirmation. The MCP specification recommends clear tool visibility, input validation, access controls, and a human opportunity to deny invocations.[^34][^35]

### Authorization

For a remote MCP endpoint, implement the current MCP authorization profile with OAuth, audience-bound access tokens, narrow scopes, and TLS. The specification requires token validation for the target MCP server and protection against returning data to unauthorized clients.[^36][^37]

Suggested scopes:

```text
projects:read
tasks:read
tasks:write
comments:write
members:read
members:manage
```

Do not grant `members:manage` by default. Bind every MCP action to the authenticated user, then reapply normal project role checks. Log tool name, inputs after secret redaction, actor, project, result, and confirmation identity.

For an early personal beta, a local MCP process using `stdio` may be simpler, but the final connector should use HTTPS OAuth if Claude runs anywhere other than the trusted Mac mini. Never store static API keys in the repository or `CLAUDE.md`.

## Build phases

| Phase | Deliverable | Exit gate |
|---|---|---|
| 0 | Product specification, architecture decisions, threat model, UX flows | Decisions reviewed; no application code yet |
| 1 | Monorepo, Swift packages, Xcode project, Vapor service, test commands | Empty app and API build; CI checks pass |
| 2 | Database schema and application service layer | Unit and integration tests for permissions and ordering pass |
| 3 | Sign in with Apple and sessions | Real-device sign-in works; token and deletion tests pass |
| 4 | Native design shell and local sample data | iPhone/iPad layouts reviewed at multiple sizes |
| 5 | Projects, workflows, tasks, notes | Two devices can create/edit shared data |
| 6 | Drag/drop, realtime, offline queue | Concurrent moves and reconnect tests pass |
| 7 | Invitations, roles, comments, activity | Permission matrix and invite lifecycle pass |
| 8 | Due dates, attachments, notifications | Background upload and APNs sandbox tests pass |
| 9 | MCP connector | Read, propose, confirm, deny, and audit tests pass |
| 10 | Mac mini deployment and backups | Restore drill succeeds; remote HTTPS works |
| 11 | Accessibility, security, performance, beta | TestFlight checklist and release criteria pass |

Each phase should end with a human review and a small commit. Do not let Claude start the next phase merely because code compiles.

## Repository instructions

Suggested layout:

```text
SharedKanban/
  CLAUDE.md
  docs/
    product.md
    architecture.md
    design-system.md
    threat-model.md
    api.md
    deployment.md
    decisions/
  ios/
    SharedKanban.xcodeproj
    SharedKanban/
    SharedKanbanTests/
    SharedKanbanUITests/
  server/
    Package.swift
    Sources/App/
    Sources/Run/
    Tests/AppTests/
  shared/
    Package.swift
    Sources/SharedDTOs/
  deploy/
    compose.yaml
    Caddyfile-or-tunnel-config/
    backup/
```

### Persistent `CLAUDE.md`

Use the following as the initial repository-level file:

```markdown
# SharedKanban project rules

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
```

Anthropic recommends keeping `CLAUDE.md` concise, checking it into version control, and using it for commands, structure, coding conventions, and workflows rather than secrets or transient task detail.[^2][^38][^1]

## Prompt sequence

Copy one prompt at a time into Claude Code. Use Plan Mode for the planning prompt and require a stop for approval at each exit gate.

### Prompt 0 — Product and architecture

```text
Act as the lead product architect and senior Apple-platform engineer for SharedKanban.

The product is a native SwiftUI iPhone/iPad app for small shared Kanban projects. The complete release needs Sign in with Apple, shared projects, exactly three or four active editable columns per project, draggable tasks, title and notes, assignee, due date, attachments, comments/activity, invitations, member removal, roles, realtime synchronization, offline caching/queued edits, APNs notifications, and a Claude connector through MCP. The API, PostgreSQL database, attachment files, background worker, and MCP adapter will ultimately run on a Mac mini. External services should be limited to Apple identity/APNs and the minimum secure ingress needed for remote devices.

Do not write production code yet.

Create or refine:
- docs/product.md with user stories, non-goals, and measurable acceptance criteria;
- docs/architecture.md with component and sequence diagrams in Mermaid;
- docs/design-system.md with iPhone/iPad layouts, interaction states, accessibility rules, and card information hierarchy;
- docs/threat-model.md covering authentication, authorization, invitations, attachments, MCP, tunnel exposure, backups, and account deletion;
- docs/api.md with resource contracts and error conventions;
- architecture decision records for Vapor/PostgreSQL, SwiftData-as-cache, WebSockets, local attachment storage, and MCP proposal/confirmation semantics.

Use these required product examples:
1. Home Projects: Backlog, To Do, In Progress, Complete.
2. Restaurants to Try: Backlog, Scheduled, Complete.

Make explicit decisions for:
- iPhone navigation and cross-column dragging;
- iPad NavigationSplitView and compact window behavior;
- roles: owner, admin, editor, viewer;
- invitation links that do not rely on Apple email matching;
- optimistic concurrency and card ranking;
- local versus remote notifications;
- offline conflict behavior;
- account deletion and token revocation;
- what can remain local and what necessarily uses Apple or a secure tunnel.

Return a short decision summary, unresolved risks, and the exact acceptance checklist. Stop for review without scaffolding code.
```

### Prompt 1 — Repository scaffold

```text
Implement only Phase 1 of SharedKanban after reading CLAUDE.md and all docs.

Create a monorepo containing:
- a native SwiftUI iOS/iPadOS Xcode project;
- a SharedDTOs Swift package containing only transport-safe Codable request/response/event types;
- a Vapor server package;
- unit-test and UI-test targets;
- Docker Compose development services for the API and PostgreSQL;
- environment configuration examples with no real secrets;
- scripts or documented commands for formatting, building, migrations, server tests, and iOS tests.

The iOS app should initially show a simple adaptive shell with sample projects, not real features. The server should expose /health and /ready and connect to a disposable test database. Add CI-compatible test commands, but do not implement authentication or domain features.

Before editing, show the planned directory tree and dependency choices. Prefer Apple and Vapor first-party facilities; justify every third-party dependency. After implementation, run all commands available in the current environment and clearly distinguish tests that require macOS/Xcode from tests that ran here. Update docs with exact local setup steps. Stop after the scaffold builds.
```

### Prompt 2 — Domain and database

```text
Implement the SharedKanban server domain layer and PostgreSQL persistence, without authentication UI, APNs, attachments, or MCP.

Create migrations and models for User, Project, ProjectMember, Invitation, Column, Task, Comment, ActivityEvent, Device, and idempotent Operation records. Use UUIDs, timestamps, soft deletion where required, record versions, and ordered ranks. Preserve the invariant that a release-1 project has three or four active columns.

Implement application services for:
- creating a project from the Home Projects or Restaurants to Try template;
- renaming/reordering columns;
- creating, editing, moving, and archiving tasks;
- notes, assignee, and due dates;
- project role checks;
- generating ordered activity events and project sequence numbers.

Task moves must be transactional, accept beforeTaskId/afterTaskId plus expected version and operationId, and return canonical state. Retries with the same operationId must not duplicate a change.

Add comprehensive unit and PostgreSQL integration tests for ordering, rank renormalization, 3/4-column validation, role permissions, stale versions, idempotent retries, and concurrent moves. Expose no unauthenticated production routes yet. Run tests and stop for review.
```

### Prompt 3 — Sign in with Apple

```text
Implement end-to-end Sign in with Apple for SharedKanban.

Client requirements:
- native AuthenticationServices flow with a cryptographic nonce;
- send authorization code, identity token, and nonce to the backend;
- store only SharedKanban access/refresh credentials in Keychain;
- session refresh, logout, revoked-credential handling, and clear user-facing failures;
- Settings screen with device/session information and an obvious Delete Account action.

Server requirements:
- verify Apple JWT signature, issuer, audience, expiry, and nonce;
- exchange and validate the authorization code with Apple;
- persist the Apple subject identifier and first-login profile data safely;
- issue short-lived access tokens and rotating hashed refresh tokens;
- rate-limit authentication endpoints;
- revoke Apple authorization when possible and delete/anonymize required data during account deletion;
- retain project data safely when another member still owns or needs it, with explicit ownership-transfer rules.

Build an AppleIdentityProvider protocol and a deterministic fake for tests. Never use live Apple endpoints in automated tests. Add tests for replayed nonce, wrong audience, expired token, refresh reuse, revoked session, and account deletion. Document all Apple Developer portal capabilities and environment variables without committing secrets. Stop after a real-device manual test checklist is produced.
```

### Prompt 4 — Design prototype

```text
Build the SharedKanban native UI against an in-memory repository only. Do not connect production networking yet.

Create polished SwiftUI screens and previews for:
- Sign in;
- project list and empty state;
- project creation with a three- or four-column template and editable column names;
- iPhone board with clear column navigation;
- iPad board using NavigationSplitView;
- task card and task detail editor;
- project members/roles;
- invitation sharing;
- activity and notification settings;
- offline, syncing, conflict, and error states.

Design goals:
- clean, calm, Apple-native, and understandable without training;
- card surface shows title, optional notes preview, due state, assignee, and compact attachment/comment counts;
- full notes and activity appear in task detail;
- usable at all iPad window widths;
- Dynamic Type, VoiceOver, keyboard, pointer, reduced motion, high contrast, and dark mode;
- every drag action has Move To and reorder menu alternatives.

Provide previews populated with Home Projects and Restaurants to Try. Add snapshot or UI tests where practical. Generate a manual design review checklist and stop before networking.
```

### Prompt 5 — Core client and sync

```text
Connect the SharedKanban SwiftUI client to the authenticated REST API.

Implement:
- typed APIClient using URLSession and async/await;
- repository interfaces separating views from transport;
- SwiftData cache for projects, columns, tasks, members, and recent activity;
- Keychain-backed session store;
- pending operation queue with UUID idempotency keys;
- project snapshot loading, incremental event synchronization, retry/backoff, reachability handling, and logout cleanup;
- optimistic create/edit behavior with understandable failure states.

The server remains authoritative. Do not directly expose SwiftData models to API DTOs. Do not silently overwrite conflicts. A 409 response must preserve the local draft, show the server value, and offer Keep Mine, Use Server, or manually merge notes.

Add contract tests with a stub URLProtocol, repository tests, offline queue/replay tests, and tests proving one user cannot load another project. Demonstrate two simulator sessions using separate accounts/fakes. Stop before implementing drag/drop and WebSockets.
```

### Prompt 6 — Drag, realtime, conflicts

```text
Implement production-quality Kanban movement and realtime synchronization.

Client:
- use SwiftUI Transferable, draggable, and dropDestination APIs;
- support reordering inside a column and movement across columns;
- show drag previews, valid drop targets, and insertion feedback;
- optimistically update the board and call the move contract with before/after IDs, expected version, and operationId;
- animate reconciliation and handle rejection without losing the user’s context;
- add Move To Column, Move Up, and Move Down alternatives;
- support keyboard and pointer interaction on iPad.

Server:
- expose authenticated move endpoints;
- publish committed project events over an authenticated WebSocket;
- authorize every subscription against project membership;
- maintain project sequence order;
- support reconnect/catch-up through GET events after sequence;
- never broadcast uncommitted data.

Test same-card concurrent moves, different-card moves into the same gap, duplicate delivery, stale versions, disconnect/reconnect, revoked membership during a connection, and rank renormalization. Provide a two-device manual test script. Stop for review.
```

### Prompt 7 — Sharing and roles

```text
Implement SharedKanban collaboration features.

Add secure expiring invitation links, system Share Sheet support, invitation preview/accept/decline, member list, role changes, member removal, ownership transfer, comments, and a readable activity feed.

Rules:
- owner/admin can invite and remove;
- editor can edit tasks and comment;
- viewer is read-only;
- only owner can transfer ownership or permanently delete a project;
- an owner cannot leave without transferring ownership or deleting the project;
- invitation tokens are random, stored only as hashes, single-use, revocable, and time-limited;
- acceptance requires an authenticated user and a confirmation screen;
- removal immediately invalidates REST access and WebSocket subscriptions.

Do not use email as the sole invitation identity. Add tests for the complete permission matrix and invitation lifecycle. Add rate limits and audit events for membership changes. Stop after producing a concise security review.
```

### Prompt 8 — Due dates, files, notifications

```text
Implement due dates, attachments, and notifications for SharedKanban.

Attachments:
- PhotosPicker and fileImporter;
- background URLSession file uploads;
- local preview and progress;
- server-side size limit, allowlist, MIME sniffing, random storage keys, SHA-256 hash, ownership checks, and safe Content-Disposition;
- delete authorization and orphan cleanup;
- attachment files stored on a mounted Mac mini volume, never in PostgreSQL.

Notifications:
- local reminders for the signed-in user’s due tasks when practical;
- APNs registration and device-token lifecycle;
- backend worker for invitations, assignments, comments/mentions, due-soon, and overdue events;
- project mute, category preferences, and quiet hours;
- no push for every routine card move by default;
- deep links that re-check authorization before opening content.

Create a fake push provider for tests and document APNs sandbox/production setup. Test background transfer recovery, unauthorized download, oversized file, malformed type, duplicate push generation, token invalidation, time-zone changes, and permission denial. Stop with a real-device test checklist.
```

### Prompt 9 — MCP connector

```text
Implement a secure MCP adapter for SharedKanban so Claude can assist with projects and tasks.

The MCP adapter must call existing application services; it must never query PostgreSQL directly and must never receive filesystem access.

Expose read tools:
- list_projects
- get_project
- list_tasks
- get_task
- search_tasks
- list_project_members

Expose proposal tools:
- propose_create_task
- propose_update_task
- propose_move_task
- propose_comment
- propose_create_project
- propose_invite_member
- propose_remove_member

Expose confirm_change and cancel_change. A proposal stores a short-lived PendingAIChange and returns the exact old/new values, affected project, required scope, and risk level. Any destructive action, membership action, or multi-task edit requires confirmation. Confirmation tokens are single-use and bound to the authenticated user, MCP client, pending change, and expiry.

Implement current MCP HTTP authorization requirements with OAuth, TLS, audience-bound short-lived tokens, narrow scopes, PKCE where applicable, per-tool authorization, strict JSON schemas, rate limits, and an audit log. Default access is projects:read and tasks:read; writes require explicit grant; members:manage is separate.

Add adversarial tests for cross-project IDs, prompt-injected tool arguments, overlong strings, duplicate confirmation, expired confirmation, token intended for another audience, membership removal between proposal and confirmation, and attempts to call undeclared parameters. Produce Claude connector setup instructions with no embedded secrets. Stop after demonstrating read, propose, deny, and confirmed-change flows.
```

### Prompt 10 — Mac mini deployment

```text
Prepare SharedKanban for durable Mac mini hosting.

Create production Dockerfiles and a Compose configuration for API, worker, PostgreSQL, ingress/tunnel, and backup jobs. Bind PostgreSQL and internal services only to the private Docker network or localhost. Store secrets outside the repository. Add health checks, restart policies, structured logs with redaction, database migrations as an explicit deployment step, disk-space monitoring, attachment quotas, and graceful shutdown.

Provide two documented ingress profiles:
1. private household beta through Tailscale;
2. public HTTPS hostname through Cloudflare Tunnel or an equivalent reverse proxy.

Create:
- deploy/runbook.md for installation, deploy, rollback, certificate/tunnel setup, Apple credentials, and troubleshooting;
- automated encrypted PostgreSQL dumps and attachment snapshots;
- retention and off-device copy instructions;
- a restore script and quarterly restore-drill checklist;
- update procedure with minimal downtime;
- incident steps for stolen refresh token, compromised invitation, lost Mac mini, disk failure, and expired Apple/APNs key.

Validate the Compose stack from a clean environment. Perform and document a restore into a disposable stack. Do not call deployment complete unless the restored app passes health, login-fake, project, and attachment integrity checks.
```

### Prompt 11 — Final hardening

```text
Perform a release-candidate review of SharedKanban. Do not add major features.

Audit:
- authentication and token lifecycle;
- every authorization boundary;
- invitation and ownership edge cases;
- attachment handling and path traversal;
- WebSocket authorization and reconnect behavior;
- offline queue/idempotency/conflicts;
- MCP scopes and confirmation bypass attempts;
- account deletion and Apple token revocation;
- logs and privacy disclosures;
- accessibility, Dynamic Type, VoiceOver, keyboard, pointer, dark mode, and reduced motion;
- iPhone and iPad performance with realistically large boards;
- backup/restore and Mac mini restart behavior.

Run static analysis, dependency review, server tests, iOS unit/UI tests, migration tests, and a two-user end-to-end scenario. Fix only release blockers and clearly label non-blocking follow-ups. Produce a TestFlight checklist, App Store privacy-data inventory, support runbook, and signed-off release criteria with evidence from commands and test results.
```

## Testing strategy

### Automated tests

- **Domain:** workflow invariants, ranks, WIP warnings, permissions, invitations, ownership.
- **API:** authentication, malformed input, cross-project identifiers, stale versions, idempotency.
- **Database:** migrations from empty and previous release, transactions, rollback, concurrent moves.
- **Client:** repositories, cache mapping, offline replay, conflict presentation, deep links.
- **UI:** project creation, add/edit/move task alternatives, member changes, account deletion.
- **Security:** token audience, scope enforcement, invitation entropy/expiry, attachment traversal, MCP confirmation bypass.
- **Operations:** health checks, backup creation, clean restore, Mac mini reboot.

### Manual scenarios

1. Create Home Projects on one iPhone and invite the second user.
2. Accept on iPad and move different tasks simultaneously.
3. Edit the same task notes offline on one device while changing them online on another; verify conflict handling.
4. Attach a photo, suspend the app, resume, and verify upload completion.
5. Assign a due date across a daylight-saving transition and verify reminders.
6. Remove a project member while that member has the board open; verify immediate denial and socket closure.
7. Ask Claude to list tasks, propose a move, deny it, propose again, confirm it, and verify the audit event.
8. Restore database and attachments into a clean stack and verify hashes and project access.

## Operational cautions

The Mac mini becomes a single point of failure. Power loss, storage failure, home internet loss, or a failed update can make the app unavailable, so a UPS, encrypted off-device backups, disk monitoring, and tested restores are part of the product rather than optional administration.

The backend is local, but the system is not entirely cloud-free. Sign in with Apple communicates with Apple’s identity servers, APNs carries remote notifications, and remote access normally traverses a VPN/tunnel or public HTTPS endpoint. If absolute local-only operation becomes a priority, remove remote collaboration and APNs or require every device to join a private VPN; that would trade convenience for isolation.[^32][^24][^27]

Cloud coding environments can write the Swift code and run server tests, but native iOS signing, simulator/device validation, Sign in with Apple entitlements, and final Xcode builds require a compatible macOS/Xcode environment. The Mac mini can serve as both the final host and the trusted Apple build/test machine, while the cloud environment remains a temporary coding workspace.

## Recommended first action

Start with Prompt 0 and review the resulting product, architecture, design, and threat-model documents before any scaffold is accepted. The highest-risk decisions are not card rendering; they are authentication, remote reachability, offline conflicts, member removal, attachment exposure, backup recovery, and the authority granted to Claude. Locking those decisions first will prevent a polished prototype from becoming an insecure or difficult-to-host system.

---

## References

1. [Best practices for Claude Code](https://code.claude.com/docs/en/best-practices)

2. [claude.com › blog › using-claude-md-filesUsing CLAUDE.MD files: Customizing Claude Code for your ...](https://claude.com/blog/using-claude-md-files) - Learn how to use CLAUDE.md files to give Claude Code persistent context about your project structure...

3. [Trello 101: How to Use Trello Boards & Cards](https://www.trello.com/guide/trello-101) - ... board. The menu is where you manage members' board permissions, control settings, search cards, ...

4. [Board view (Kanban) in Notion | Notion Help](https://www.notion.com/help/boards) - Open the settings menu at the top right of your database → Sub-groups . Find and select the property...

5. [How to Create Trello Projects and Invite Members](https://www.trello.com/guide/enterprise/creating-projects-inviting-members) - Create a board from scratch or from a template; Invite members to collaborate; Set membership permis...

6. [Learn Trello board basics](https://trello.com/en/guide/enterprise/trello-basics) - This guide will cover all the essentials of Trello to get you up and running as fast as possible. We...

7. [Issue status – Linear Docs](https://linear.app/docs/configuring-workflows) - Issue statuses define the type and order of states that issues can move through from start to comple...

8. [Kanban Software & Project Management Tool • Asana](https://asana.com/uses/kanban-boards) - Kanban boards are a great way to run the Scrum methodology and manage Agile projects. With Asana Kan...

9. [Working with WIP limits for kanban - Atlassian](https://www.atlassian.com/agile/kanban/wip-limits) - Learn how to use work in progress limits, the 4 goals for agile teams using WIP limits, and why WIP ...

10. [Split views | Apple Developer Documentation](https://developer.apple.com/design/human-interface-guidelines/split-views) - It's common to use a split view to display a sidebar for navigation, where the leading pane lists th...

11. [Adopting drag and drop using SwiftUI | Apple Developer ...](https://developer.apple.com/documentation/SwiftUI/Adopting-drag-and-drop-using-SwiftUI) - Enable drag-and-drop interactions in lists, tables and custom views.

12. [Making a view into a drag source](https://developer.apple.com/documentation/swiftui/making-a-view-into-a-drag-source) - Adopt draggable API to provide items for drag-and-drop operations.

13. [Change permissions on a board | Trello](https://support.atlassian.com/trello/docs/changing-permissions-on-a-board/) - Set the visibility and permissions of features by going to the menu on the right side of the board, ...

14. [Using the keychain to manage user secrets - Apple Developer](https://developer.apple.com/documentation/security/using-the-keychain-to-manage-user-secrets) - Relieve the user of remembering small secrets by storing them in the keychain.

15. [Keychain data protection](https://support.apple.com/guide/security/keychain-data-protection-secb0694df1a/web) - The various Apple operating systems use differing mechanisms to enforce the guarantees associated wi...

16. [URLSession | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/urlsession) - An object that coordinates a group of related, network data transfer tasks.

17. [Downloading files in the background](https://developer.apple.com/documentation/foundation/downloading-files-in-the-background) - Create tasks that download files while your app is inactive.

18. [Fluent - Vapor Docs](https://docs.vapor.codes/fluent/overview/) - Fluent is an ORM framework for Swift. Fluent currently has four officially supported drivers. Postgr...

19. [WebSockets - Vapor Docs](https://docs.vapor.codes/advanced/websockets/) - WebSockets allow for two-way communication between a client and server. Unlike HTTP, which has a req...

20. [Docker Deploys - Vapor Docs](https://docs.vapor.codes/deploy/docker/) - Using Docker to deploy your Vapor app has several benefits: Your dockerized app can be spun up relia...

21. [Preventing Insecure Network Connections](https://developer.apple.com/documentation/security/preventing-insecure-network-connections) - Enforce secure network links in your app by relying on App Transport Security.

22. [NSAppTransportSecurity | Apple Developer Documentation](https://developer.apple.com/documentation/bundleresources/information-property-list/nsapptransportsecurity) - A description of changes made to the default security for HTTP connections.

23. [Routing](https://developers.cloudflare.com/tunnel/concepts/routing/) - Route traffic to private networks and services through Cloudflare Tunnel.

24. [Cloudflare Tunnel](https://developers.cloudflare.com/tunnel/) - Securely connect your origin servers, APIs, and services to Cloudflare with post-quantum encrypted t...

25. [tailscale funnel command](https://tailscale.com/docs/reference/tailscale-cli/funnel) - Review the `tailscale funnel` CLI command.

26. [Tailscale Funnel](https://tailscale.com/docs/features/tailscale-funnel) - Securely route internet traffic to local services using Tailscale Funnel.

27. [Sign in with Apple REST API | Apple Developer Documentation](https://developer.apple.com/documentation/signinwithapplerestapi) - Communicate between your app servers and Apple’s authentication servers.

28. [developer.apple.com · documentation · signinwithappleAuthenticating users with Sign in with Apple - Apple Developer](https://developer.apple.com/documentation/signinwithapple/authenticating-users-with-sign-in-with-apple) - Securely authenticate users and create accounts for them in your app.

29. [Offering account deletion in your app - Support](https://developer.apple.com/support/offering-account-deletion-in-your-app/) - Apps submitted to the App Store that support account creation must also include an option in the app...

30. [TN3194: Handling account deletions and revoking tokens ...](https://developer.apple.com/documentation/technotes/tn3194-handling-account-deletions-and-revoking-tokens-for-sign-in-with-apple) - Learn the best techniques for managing Sign in with Apple user sessions and responding to account de...

31. [Registering your app with APNs](https://developer.apple.com/documentation/usernotifications/registering-your-app-with-apns) - Overview. Apple Push Notification service (APNs) must know the address of a user's device before it ...

32. [Setting up a remote notification server](https://developer.apple.com/documentation/usernotifications/setting-up-a-remote-notification-server) - Send test notifications and access delivery logs to test your app's integration with Apple Push Noti...

33. [Introducing the Model Context Protocol \ Anthropic](https://www.anthropic.com/research/model-context-protocol) - The Model Context Protocol (MCP) is an open standard for connecting AI assistants to the systems whe...

34. [Tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)

35. [Specification](https://modelcontextprotocol.io/specification/2025-06-18)

36. [Authorization](https://modelcontextprotocol.io/specification/draft/basic/authorization)

37. [Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)

38. [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices?curius=1051&rut=3deefe08072d1ab79e05e5bdff916ae91ce3d813d54da40f30f8381a31d75f4e) - A blog post covering tips and tricks that have proven effective for using Claude Code across various...

