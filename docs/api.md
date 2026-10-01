# SharedKanban API contract

This document is the Phase 0 contract for the REST API, the realtime WebSocket and the MCP tools. It applies the decisions in `docs/decisions/decision-register.md` (cited inline as D01..D22); where the two disagree the register wins. Component and sequence diagrams live in `docs/architecture.md`. Every JSON example below becomes a fixture under `shared/Fixtures/` decoded by SharedDTOs contract tests on Linux and iOS, so the document cannot drift from the types (D22). Hostnames use `example.invalid`; every token shown is a placeholder.

Where the brief's REST outline and the register differ, the register's shape is used (D22):

| Brief outline | Contract here | Why |
|---|---|---|
| `POST /v1/invitations/{token}/accept` | `POST /v1/invitations/accept {operationId, token}` | Token never in a path (D04) |
| `POST /v1/attachments/{id}/complete` | Folded into `PUT /v1/attachments/{id}/content` | The upload is the completion (D14) |
| `DELETE /v1/account` | `POST /v1/account/delete` | DELETE carries no body (D18, D09) |
| `DELETE /v1/columns/{id}` deletes | Archives; columns are never deleted | D19 |
| `DELETE /v1/devices/{deviceId}` | `PUT`/`DELETE /v1/devices/{installationId}` | Device rows are keyed by installation (D07) |

## 1. Conventions (D22)

### 1.1 Base path, transport, format

- Every route is under `/v1`; the WebSocket is `WSS /v1/realtime`. Outside `/v1`: `GET /health` and `GET /ready` (unauthenticated, D11), `GET /invite` and `GET /.well-known/apple-app-site-association` (D04), and `/mcp` (loopback-only in release 1, D15).
- Authentication is `Authorization: Bearer <token>` on every request and on the WebSocket upgrade. Token classes (D16): `skat_` access tokens for REST and the upgrade, `skut_` upload tokens accepted only by the attachment content `PUT`, `skct_` Claude connection tokens accepted only by `/mcp`. Tokens never travel in query strings or cookies.
- JSON only: requests with a body and every JSON response use `Content-Type: application/json; charset=utf-8`; keys are camelCase; bodies are limited to 1 MB except attachment content (413 `payload_too_large`); any other content type on a JSON route is 415 `unsupported_media_type`.
- Timestamps are ISO-8601 UTC with milliseconds and `Z` (`2026-10-01T12:34:56.789Z`), decoded leniently with or without fractional seconds. UUIDs are version 4, documented lowercase, compared case-insensitively. Clients may supply `id` on create for projects, columns, tasks and comments; a collision is 409 `duplicate_id`.
- Versioning: a breaking change creates `/v2` and `/v1` stays for at least 90 days after the app that uses it is replaced; additive changes (new optional fields, new enum cases) stay in `/v1`. Clients ignore unknown fields and decode unknown enum cases as `.unknown`.
- Pagination: lists are `{ "items": [...], "nextCursor": "<opaque base64url>" | null }` with `?cursor=&limit=` (default 50, max 200). Events use `?after=<sequence>&limit=` (max 500) and the activity feed `?before=<sequence>&limit=` (default 50); both are addressed by sequence, not cursor.
- Successful single-resource mutations return the DTO as the top-level object. A `DELETE` returns 204 with no body.

### 1.2 Headers

| Header | Direction | Meaning |
|---|---|---|
| `Authorization: Bearer <token>` | request | Required on every authenticated route |
| `X-Request-Id` | both | Client may send up to 64 characters of `[A-Za-z0-9-]`; otherwise generated or replaced; echoed, logged, and included in every error body |
| `X-Client-Version` | request | App version `major.minor.patch`; below the configured minimum is 426 `upgrade_required`; absent is accepted (operator tooling) |
| `Idempotency-Replayed: true` | response | The response is a stored replay (D18) |
| `X-Project-Sequence` | response | Sequence of the event a single-resource mutation wrote; not reproduced on replays |
| `X-Merged-From-Version` | response | A stale `expectedVersion` was merged (D05) |
| `Retry-After` | response | Seconds to wait, on every 429; no other rate-limit header is defined in release 1 |
| `Strict-Transport-Security`, `X-Content-Type-Options: nosniff`, `Cache-Control: no-store` | response | On every response; attachment downloads use `Cache-Control: private, no-store` plus the headers in section 10 |

### 1.3 Rate limits

Limits are keyed by user first; IP limits apply only where the client address reaches the origin (D10). Exceeding one is 429 `rate_limited` with `Retry-After`.

| Scope | Limit |
|---|---|
| `/v1/auth/*` | 10 per minute per IP and 5 per minute per user |
| `/v1/auth/refresh` | 30 per minute per session |
| Invitation preview, accept, decline | 10 per minute per user and 30 per minute per IP |
| Invitation creation | 5 per project per hour |
| Attachment initiate | 30 per hour per user; at most 10 pending uploads per user (`details.reason = "pendingUploads"`) |
| MCP | 60 read calls and 10 proposal calls per minute per grant (confirm and cancel count as proposals) |
| WebSocket inbound | 10 messages per second, 64 KB per frame; exceeding closes the socket with 4429 |
| Everything else | 600 per minute per user |

### 1.4 Error envelope

Every error, from every route, has exactly this shape. `message` is a human sentence safe to display; `details` is `null` or an object whose shape depends on the code; `requestId` matches `X-Request-Id`.

```json
{
  "error": {
    "code": "validation_failed",
    "message": "The title must be between 1 and 200 characters.",
    "details": { "fields": [ { "path": "title", "reason": "length" } ] },
    "requestId": "7c1f3e52-5a0b-4f1e-9c3d-2b8e6f4a1d90"
  }
}
```

### 1.5 Error codes

The codes are one Swift enum in SharedDTOs shared by server, app and MCP adapter. Any id outside the caller's projects returns `not_found` with the same status and body as a genuinely missing id, so existence never leaks; the server log records the real reason under the `requestId`.

| Code | HTTP | When | Client action |
|---|---|---|---|
| `invalid_request` | 400 | Malformed JSON, missing `operationId` or `expectedVersion`, bad query value; `expectedVersion` ahead of the server (`details.reason = "expectedVersionAhead"`, `details.current`) | Fix the request; on `expectedVersionAhead` reload the project snapshot |
| `unauthenticated` | 401 | Missing, malformed or unknown token; Apple identity token failed verification; invalid upload token | Show sign-in |
| `token_expired` | 401 | Access token past its expiry, or refresh token past its sliding or absolute window | Refresh once and retry; if refresh fails, show sign-in |
| `session_revoked` | 401 | Session revoked by logout elsewhere, revoke-all, refresh reuse detection or account deletion | Forced sign-out keeping the outbox (D08) |
| `reauth_required` | 401 | Account deletion without a valid fresh Apple sign-in bound to the challenge | Re-run the Apple sign-in step |
| `forbidden` | 403 | Caller is a member but the role is insufficient; MCP confirmation re-authorization failed or the confirmation token is wrong | Show "you don't have permission"; refresh membership |
| `not_found` | 404 | Entity does not exist, is purged, or belongs to a project the caller is not a member of (same status either way) | For a project: remove it from the cache (D12); otherwise drop the local change with a toast |
| `version_conflict` | 409 | Same-field conflict on PATCH; stale member version on a role change; stale `expectedProjectVersion` on reorder; attachment re-PUT with a different hash. `details.current` holds the server record | Conflict UI (D08), then a new `operationId` |
| `position_conflict` | 409 | Move with a stale `expectedTaskVersion` after another `task.moved` | Snap the card to the server position and show the banner |
| `neighbors_changed` | 409 | Both move neighbors are gone; `details.currentOrder` lists the destination column's task ids | Recompute neighbors, retry once with a new `operationId`, then snap |
| `task_archived` | 409 | Mutation on an archived task; `details.current` | Offer Restore-and-apply or discard |
| `project_archived` | 409 | Mutation on an archived project (except unarchive, leave, member removal, notification settings) | Render read-only; fail queued operations |
| `owner_must_transfer` | 409 | The owner tries to leave while other members exist | Offer Transfer ownership or Delete project |
| `owned_projects_require_transfer` | 409 | Account deletion missing a decision for an owned project; `details.projects: [{id, name, memberCount}]` | Re-show the consequences sheet |
| `change_already_applied` | 409 | Second `confirm_change` for the same change | Treat as success |
| `stale_proposal` | 409 | Captured versions changed, or the change expired or was cancelled (`details.reason ∈ versions_changed|expired|cancelled`) | Re-propose |
| `duplicate_id` | 409 | Client-supplied `id` already exists | Mint a new id |
| `invitation_unavailable` | 410 | `details.reason ∈ expired|revoked|used|declined|project_archived|inviter_unavailable` | Show the reason; stop |
| `sequence_too_old` | 410 | Catch-up gap exceeds 2,000 events | Reload the snapshot |
| `payload_too_large` | 413 | JSON body over 1 MB; attachment over 25 MB or over its declared size | Show the limit |
| `unsupported_media_type` | 415 | Non-JSON content type on a JSON route; attachment type outside the allowlist or sniffed type mismatch | Show the allowed types |
| `validation_failed` | 422 | Field rules broken; `details.fields: [{path, reason}]` | Show field errors; mark the operation failed |
| `column_count_invalid` | 422 | Would leave fewer than 3 or more than 4 active columns | Disable the control (D19) |
| `column_not_empty` | 422 | Archiving a column with active tasks without `moveTasksTo` | Ask for a target column |
| `idempotency_mismatch` | 422 | Same `operationId`, different payload | Client bug; mint a new id |
| `upgrade_required` | 426 | `X-Client-Version` below the minimum | Show the update screen |
| `rate_limited` | 429 | A limit in 1.3 was exceeded; `Retry-After` | Back off |
| `internal` | 500 | Unexpected failure; generic message, `requestId` points to logs | Retry with backoff; show the `requestId` |
| `insufficient_storage` | 507 | Attachment volume below 10 % or 10 GB free | Show "the server is out of space" |

Every authenticated route may return 400, 401, 404, 426, 429 and 500, every project mutation may return 403 and 409 `project_archived`, and every body-carrying mutation may return 422 `validation_failed` and `idempotency_mismatch`; the endpoint tables list only the codes beyond these.

## 2. Idempotency (D18)

- Carrier: every `POST` and `PATCH` with a JSON body carries `operationId` (UUID v4) as a body field of the typed DTO; the `Idempotency-Key` header is not used. Exceptions: `/v1/auth/*`, `/v1/account/delete-challenge`, `/v1/account/delete` (authenticated by the challenge), `/v1/invitations/preview` and `/v1/invitations/decline` (D04 shapes). `DELETE` carries no body and is a strict state no-op when the target is already in the requested state (same status, no new event). `PUT` (attachment content, device registration, notification settings) is a full replacement, idempotent by state.
- Scope: per user. The same UUID from another user is a different operation. The client mints the id when it creates the operation, not when it sends it (D08), and never reuses an id after changing the payload.
- Record: `(userId, operationId)` with a `requestHash` of method + path + canonical JSON (sorted keys, explicit nulls kept, no floats), inserted inside the mutation transaction before the mutation runs. A concurrent duplicate blocks on the primary key until the first attempt finishes.
- Stored outcomes: every 2xx, 409 and 422 (except `idempotency_mismatch`) is stored with its status and body. 400, 401, 403, 404, 410, 413, 415, 426, 429, 5xx and transport failures store nothing; a retry is a fresh evaluation.
- Replay: same `operationId` and same hash returns the stored status and body verbatim with `Idempotency-Replayed: true`, re-emits no events and no pushes, and does not reproduce `X-Project-Sequence`. A different hash is 422 `idempotency_mismatch`.
- Interaction with 409: a replayed 409 stays a 409. After the user resolves a conflict the client submits a new `operationId` with the corrected input, so a retry can never double-apply or turn a conflict into a silent success.
- TTL: 30 days, purged daily, matching the offline queue age-out (D08). Events carry `operationId` so a client recognises its own echoes during catch-up.
- MCP confirmations execute the operation ids generated at proposal time, so a double confirmation cannot double-apply (D15).

## 3. Optimistic concurrency (D05)

Every mutable record (project, column, task, comment, member) carries `version` (starts at 1, +1 per committed change), `updatedAt` and `updatedBy`. Every PATCH requires `operationId` and `expectedVersion` (400 `invalid_request` if either is missing). PATCH bodies use `Patch<T>` semantics: an absent field is unchanged, an explicit `null` clears; the fields present are "the fields in this patch" and are the same identifiers used as keys in event `changes`.

Rule, inside one transaction under the project row lock (D06):

1. `expectedVersion == version`: apply, bump, write one event.
2. `expectedVersion < version`: union the `changes` keys of this entity's events with `entityVersion > expectedVersion`. No intersection with the patch fields: apply as a merge, bump, respond 200 with `X-Merged-From-Version: <expectedVersion>`. Intersecting fields whose incoming value equals the server's current value are dropped and the rest merges; nothing left to change: 200 with the current record, no bump, no event. Otherwise 409 `version_conflict`. Missing events for the window: every field counts as changed (fail closed).
3. `expectedVersion > version`: 400 `invalid_request` with `details.reason = "expectedVersionAhead"` and `details.current`.

| Entity | Mergeable PATCH fields | Never PATCHed |
|---|---|---|
| Task | `title`, `notes`, `dueAt`, `assigneeId` (each independent) | `columnId`, `rank` (move), `completedAt` (derived from the done column), `archivedAt` (archive/restore endpoints) |
| Column | `title`, `color`, `wipLimit`, `isDone` | `rank` (`/columns/reorder` with `expectedProjectVersion`), `archivedAt` |
| Project | `name` | `archivedAt`, `deletedAt`, `ownerId` (own endpoints) |
| Comment | `body` (concurrent edits always conflict) | — |
| Member | `role` | `notificationSettings` (PUT) |

The 409 body (the `current` DTO is returned only after normal read authorization):

```json
{
  "error": {
    "code": "version_conflict",
    "message": "Sam changed the notes while you were editing.",
    "details": {
      "entityType": "task",
      "entityId": "0d000000-0000-4000-8000-000000000001",
      "expectedVersion": 12,
      "currentVersion": 14,
      "conflictingFields": ["notes"],
      "current": { "id": "0d000000-0000-4000-8000-000000000001", "version": 14, "notes": "Call the plumber first", "...": "full TaskDTO" }
    },
    "requestId": "7c1f3e52-5a0b-4f1e-9c3d-2b8e6f4a1d90"
  }
}
```

Moves apply the same rule with the field set `{columnId, rank}`: a stale `expectedTaskVersion` is tolerated when no `task.moved` event exists for the task after it, otherwise 409 `position_conflict` with the same body shape. Rank renormalization never bumps versions and is not a position change. On every write, merged or not: `assigneeId` must be a current member (422), a PATCH on an archived task is 409 `task_archived` with `details.current`, a mutation on an archived project is 409 `project_archived`, and `updatedBy` is the actor of the committed write.

## 4. Resource shapes

Field lists are the SharedDTOs types. `updatedBy`, `actorUserId`, `ownerId` and `assigneeId` are user ids; display names are resolved client-side from the member list (D17), so a deleted member renders as "Deleted member".

| DTO | Fields |
|---|---|
| `ProjectDTO` | `id, name, ownerId, archivedAt, deletedAt, version, createdAt, updatedAt, updatedBy, myRole` |
| `ProjectSnapshotDTO` | `project: ProjectDTO, columns: [ColumnDTO]` (active and archived), `tasks: [TaskDTO]` (active only, unpaginated), `members: [MemberDTO], sequence` |
| `ColumnDTO` | `id, projectId, title (1–40), rank, color ∈ gray\|red\|orange\|yellow\|green\|teal\|blue\|indigo\|purple\|pink, wipLimit (null or 1–99), isDone, archivedAt, version, createdAt, updatedAt, updatedBy` |
| `TaskDTO` | see below |
| `MemberDTO` | `projectId, userId, displayName, role ∈ owner\|admin\|editor\|viewer, version, joinedAt, notificationSettings` (non-null only on the caller's own row: `{ muted, overrides: { <category>: Bool } }`) |
| `InvitationDTO` | `id, projectId, role, inviterId, expiresAt, createdAt, acceptedAt, declinedAt, revokedAt` (never the token) |
| `CommentDTO` | `id, taskId, projectId, authorId, body (1–5,000), mentionedUserIds, editedAt, deletedAt, version, createdAt, updatedAt, updatedBy` |
| `AttachmentDTO` | `id, taskId, projectId, uploaderId, originalName, mimeType` (sniffed), `byteSize, sha256, state ∈ pending\|complete, createdAt, completedAt, deletedAt` |
| `ActivityEventDTO` | `id, projectId, sequence, actorUserId` (null for system), `via ∈ app\|mcp\|system, action, entityType, entityId, entityVersion, changes, extra, operationId, createdAt` |
| `SessionDTO` | `id, deviceName, deviceModel, osVersion, appVersion, createdAt, lastUsedAt, current` |
| `DeviceDTO` | `installationId, environment ∈ sandbox\|production, timeZone, active, lastSeenAt` |
| `AccountDTO` | `user: { id, displayName, email, createdAt }, notificationSettings: { categories: { invitation, assignment, comment, mention, membership, activity }, dueLeadTime ∈ none\|1h\|1d, quietHours: { start: "HH:mm", end: "HH:mm" } \| null }` |
| `GrantDTO` | `id, clientName, scopes, createdAt, expiresAt, lastUsedAt, revokedAt` |

```json
{
  "id": "0d000000-0000-4000-8000-000000000001",
  "projectId": "0a000000-0000-4000-8000-000000000001",
  "columnId": "0c000000-0000-4000-8000-000000000002",
  "title": "Fix the kitchen tap",
  "notes": "Washer, then check the valve.",
  "rank": "a1",
  "assigneeId": "0e000000-0000-4000-8000-000000000002",
  "dueAt": "2026-10-04T09:00:00.000Z",
  "completedAt": null,
  "archivedAt": null,
  "version": 12,
  "createdBy": "0e000000-0000-4000-8000-000000000001",
  "createdAt": "2026-10-01T08:00:00.000Z",
  "updatedAt": "2026-10-01T12:34:56.789Z",
  "updatedBy": "0e000000-0000-4000-8000-000000000002",
  "attachmentCount": 1,
  "commentCount": 2
}
```

String caps shared by REST and MCP: task title 1–200, notes 0–20,000, comment body 1–5,000, project name 1–80, display name 1–80, column title 1–40. `dueAt` is an instant; there is no all-day concept (D07). `attachmentCount` and `commentCount` are computed on reads and maintained client-side from events.

## 5. Endpoints

Reading the tables: role `any` means any signed-in user; `member` any role in the project; `editor+`, `admin+` and `owner` are inclusive upward (D03). Bodies are shown compactly; `"…"` stands for a UUID or token. Creates return 201, other successes 200, `DELETE` 204. Mutations that write an event also return `X-Project-Sequence`.

### 5.1 Auth (D16)

No `operationId` on these routes.

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `POST /v1/auth/apple` | any | `{"identityToken":"<apple jwt>","authorizationCode":"<code>","nonce":"<hex>","installationId":"…","device":{"name":"Sam's iPhone","model":"iPhone16,1","osVersion":"18.0","appVersion":"1.0.0"},"profile":{"displayName":"Sam","email":"sam@example.invalid"}}` (`profile` is non-null only on Apple's first authorization) | 200 `SessionTokensDTO` | 401 `unauthenticated` (signature, issuer, audience, nonce, expiry or code exchange failed) |
| `POST /v1/auth/refresh` | any | `{"refreshToken":"skrt_…"}` | 200 `SessionTokensDTO`; the previous refresh token stays valid for 60 s for a lost-response retry | 401 `token_expired` (window lapsed), `session_revoked` (reuse detected; the session is revoked) |
| `POST /v1/auth/logout` | session | none | 204; revokes the session, deletes its device row, invalidates its upload tokens, closes its sockets | — |
| `GET /v1/auth/sessions` | session | — | `{ "items": [SessionDTO], "nextCursor": null }` | — |
| `DELETE /v1/auth/sessions/{sessionId}` | session | — | 204 (no-op when already revoked) | — |
| `POST /v1/auth/sessions/revoke-all` | session | `{"keepCurrent":true}` | 204 | — |
| `POST /v1/auth/apple/notifications` | Apple | Apple's signed server-to-server event (D09) | 200 | — (registered only once a public hostname exists) |

```json
{
  "accessToken": "skat_0b000000-0000-4000-8000-000000000001.EXAMPLEONLYNOTATOKEN0000000000000000000000",
  "accessExpiresAt": "2026-10-01T13:34:56.789Z",
  "refreshToken": "skrt_EXAMPLEONLYNOTATOKEN0000000000000000000000",
  "refreshExpiresAt": "2026-12-30T12:34:56.789Z",
  "sessionId": "0b000000-0000-4000-8000-000000000001",
  "user": { "id": "0e000000-0000-4000-8000-000000000001", "displayName": "Sam", "email": "sam@example.invalid", "createdAt": "2026-10-01T12:34:56.789Z" }
}
```

### 5.2 Account (D07, D09)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `GET /v1/account` | session | — | 200 `AccountDTO` | — |
| `PATCH /v1/account` | session | `{"operationId":"…","displayName":"Sam"}` (the user row has no version, D05) | 200 `AccountDTO` | — |
| `PUT /v1/account/notification-settings` | session | `{"categories":{"invitation":true,"assignment":true,"comment":true,"mention":true,"membership":true,"activity":false},"dueLeadTime":"1d","quietHours":{"start":"22:00","end":"07:00"}}` (full replacement; time zone comes from the device record) | 200 `AccountDTO` | — |
| `POST /v1/account/delete-challenge` | session | none | 200 `{"nonce":"<hex>","expiresAt":"…"}` (+5 minutes, single use) | — |
| `POST /v1/account/delete` | session | section 11 | 204 | 401 `reauth_required`; 409 `owned_projects_require_transfer` |

### 5.3 Projects (D03, D19)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `GET /v1/projects?scope=` | any | `scope` omitted: active and archived memberships; `scope=deleted`: soft-deleted projects the caller owns, restorable for 30 days | `{ "items": [ProjectDTO], "nextCursor": null }` | — |
| `POST /v1/projects` | any | `{"operationId":"…","id":"…","name":"Home Projects","template":{"id":"home-projects","titles":null}}`; `template.id ∈ home-projects\|restaurants-to-try\|custom`; `titles` (3 or 4 names) overrides the template names and is required for `custom`; the last column is the done column | 201 `ProjectSnapshotDTO` (caller is owner; atomic with its columns) | 409 `duplicate_id` |
| `GET /v1/projects/{projectId}` | member | — | 200 `ProjectSnapshotDTO` with the current `sequence` | — |
| `PATCH /v1/projects/{projectId}` | admin+ | `{"operationId":"…","expectedVersion":3,"name":"Home"}` | 200 `ProjectDTO` | 409 `version_conflict` |
| `POST /v1/projects/{projectId}/archive` | admin+ | `{"operationId":"…"}` | 200 `ProjectDTO`; evicts realtime subscriptions | 409 `project_archived` when already archived |
| `POST /v1/projects/{projectId}/unarchive` | admin+ | `{"operationId":"…"}` | 200 `ProjectDTO` (no-op when active) | — |
| `DELETE /v1/projects/{projectId}` | owner | — | 204; soft delete, hidden for everyone, pending invitations revoked, subscriptions dropped; purged after 30 days | — |
| `POST /v1/projects/{projectId}/restore` | owner | `{"operationId":"…"}` | 200 `ProjectDTO` | 404 once purged |
| `POST /v1/projects/{projectId}/transfer-ownership` | owner | `{"operationId":"…","newOwnerUserId":"…","expectedVersion":3}` | 200 `ProjectDTO`; previous owner becomes admin | 409 `version_conflict`; 422 (target not a member) |
| `POST /v1/projects/{projectId}/tasks/archive-completed` | editor+ | `{"operationId":"…"}` | 200 `{"archivedTaskIds":["…"]}`; one `task.archived` event per task | — |

### 5.4 Invitations and members (D04, D03)

Preview, accept and decline require a signed-in user; the token travels only in the body. The share link is `https://kanban.example.invalid/invite#<token>`; the fragment never reaches the server.

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `POST /v1/projects/{projectId}/invitations` | admin+; `role: "admin"` owner only | `{"operationId":"…","role":"editor","expiresInDays":7}` (`role ∈ admin\|editor\|viewer`; `expiresInDays ∈ 1\|7\|30`) | 201 below | 403 (admin inviting an admin) |
| `GET /v1/projects/{projectId}/invitations` | admin+ | — | `{ "items": [InvitationDTO] }` pending only, no tokens | — |
| `DELETE /v1/invitations/{invitationId}` | admin+ or inviter | — | 204 (no-op when revoked) | — |
| `POST /v1/invitations/preview` | any | `{"token":"<43 chars>"}` | 200 `{"projectName":"Home Projects","inviterDisplayName":"Sam","role":"editor","expiresAt":"…"}` | 404; 410 `invitation_unavailable` |
| `POST /v1/invitations/accept` | any | `{"operationId":"…","token":"<43 chars>"}` | 201 `{"membership": MemberDTO, "project": ProjectDTO, "alreadyMember": false}`; an existing member gets 200 with `alreadyMember: true`, the role unchanged and the invitation consumed | 404; 410 `invitation_unavailable` |
| `POST /v1/invitations/decline` | any | `{"token":"<43 chars>"}` | 204 (no-op when already declined) | 404; 410 |
| `GET /v1/projects/{projectId}/members` | member | — | `{ "items": [MemberDTO] }` | — |
| `PATCH /v1/projects/{projectId}/members/{userId}` | admin+ between editor and viewer; owner for admin promotion and demotion; an admin may demote self to editor | `{"operationId":"…","role":"viewer","expectedVersion":2}` (member row version) | 200 `MemberDTO`; target receives `member.role_changed` | 403; 409 `version_conflict`; 422 (`owner` role, or the owner as target) |
| `DELETE /v1/projects/{projectId}/members/{userId}` | admin+ for editors and viewers; owner for admins; self = leave | — | 204; assignee cleared on the member's active tasks with `task.updated` (`via = system`); from the next request the user gets 404 for the project | 403; 409 `owner_must_transfer` (owner leaving with other members; a sole-member owner leaving performs project deletion) |
| `PUT /v1/projects/{projectId}/members/{userId}/notification-settings` | self only | `{"muted":false,"overrides":{"activity":true}}` | 200 `MemberDTO` | 403 when `userId` is not the caller |

Invitation creation response; the plaintext token exists only here and in the share sheet:

```json
{
  "invitation": { "id": "0a100000-0000-4000-8000-000000000001", "projectId": "0a000000-0000-4000-8000-000000000001", "role": "editor", "inviterId": "0e000000-0000-4000-8000-000000000001", "expiresAt": "2026-10-08T12:34:56.789Z", "createdAt": "2026-10-01T12:34:56.789Z", "acceptedAt": null, "declinedAt": null, "revokedAt": null },
  "url": "https://kanban.example.invalid/invite#EXAMPLEONLYNOTATOKEN0000000000000000000000",
  "token": "EXAMPLEONLYNOTATOKEN0000000000000000000000"
}
```

### 5.5 Columns (D19, D06)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `POST /v1/projects/{projectId}/columns` | admin+ | `{"operationId":"…","id":"…","title":"Waiting","color":"orange","wipLimit":3,"isDone":false}` | 201 `ColumnDTO` appended last | 422 `column_count_invalid`; 409 `duplicate_id` |
| `PATCH /v1/columns/{columnId}` | admin+ | `{"operationId":"…","expectedVersion":1,"title":"Doing","wipLimit":null,"isDone":true}` | 200 `ColumnDTO`; `isDone: true` clears the flag on the previous done column | 409 `version_conflict` |
| `POST /v1/projects/{projectId}/columns/reorder` | admin+ | `{"operationId":"…","orderedColumnIds":["…","…","…"],"expectedProjectVersion":3}` (exactly the active column ids) | 200 `{"columns":[ColumnDTO],"sequence":1190}` | 409 `version_conflict` (strict, no merge) |
| `DELETE /v1/columns/{columnId}?moveTasksTo=` | admin+ | — | 204; archives; tasks appended to the target's bottom with `task.moved` events | 422 `column_count_invalid`, `column_not_empty` |
| `POST /v1/columns/{columnId}/restore` | admin+ | `{"operationId":"…"}` | 200 `ColumnDTO` (no-op when active) | 422 `column_count_invalid` at 4 |

### 5.6 Tasks (D05, D06, D19)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `POST /v1/projects/{projectId}/tasks` | editor+ | `{"operationId":"…","id":"…","columnId":null,"title":"Fix the kitchen tap","notes":null,"assigneeId":null,"dueAt":null}`; null `columnId` = first active column; appended to the bottom | 201 `TaskDTO` | 404 (archived or foreign column); 409 `duplicate_id` |
| `GET /v1/projects/{projectId}/tasks?state=&q=&cursor=&limit=` | member | `state ∈ active\|archived\|all` (default `active`); `q` matches title and notes case-insensitively | `{ "items": [TaskDTO], "nextCursor": … }` ordered by `(rank, id)` | — |
| `GET /v1/tasks/{taskId}` | member | — | 200 `TaskDTO` | — |
| `PATCH /v1/tasks/{taskId}` | editor+ | `{"operationId":"…","expectedVersion":12,"notes":"Washer, then valve.","dueAt":null}` | 200 `TaskDTO` | 409 `version_conflict`, `task_archived`; 422 (assignee not a member) |
| `POST /v1/tasks/{taskId}/move` | editor+ | section 6 | 200 move result | section 6 |
| `DELETE /v1/tasks/{taskId}` | editor+ | — | 204; archives (no-op when archived) | — |
| `DELETE /v1/tasks/{taskId}?permanent=true` | admin+ | — | 204; removes comments and attachment rows, files deleted by the worker | 422 (task not archived) |
| `POST /v1/tasks/{taskId}/restore` | editor+ | `{"operationId":"…"}` | 200 `TaskDTO` at the bottom of its column, or of the first column if its column is archived | — |

### 5.7 Attachments (D14)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `POST /v1/tasks/{taskId}/attachments/initiate` | editor+ | `{"operationId":"…","originalName":"tap.heic","declaredMime":"image/heic","byteSize":2048000,"sha256":"<64 hex>"}` | 201 upload instructions (section 10) | 413; 415; 422 (20 per task, 2 GB per project); 429 (`pendingUploads`); 507 |
| `PUT /v1/attachments/{attachmentId}/content` | upload token | raw bytes, `Content-Type: <declaredMime>`, `Content-Length` | 200 `AttachmentDTO` (`state: "complete"`); `attachment.added` | 401 `unauthenticated`; 409 `version_conflict` (re-PUT with a different hash); 413; 415; 422 (hash mismatch); 507 |
| `GET /v1/tasks/{taskId}/attachments` | member | — | `{ "items": [AttachmentDTO] }` complete, undeleted | — |
| `GET /v1/attachments/{attachmentId}/content` | member | optional `Range` | 200 or 206 bytes with the section 10 headers | 404 (pending or deleted) |
| `DELETE /v1/attachments/{attachmentId}` | uploader or admin+ | — | 204 (no-op when deleted); file unlinked 24 h later | 403 |

### 5.8 Comments (D03, D07)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `POST /v1/tasks/{taskId}/comments` | editor+ | `{"operationId":"…","id":"…","body":"Done, but the valve leaks.","mentionedUserIds":["…"]}` (mentions must be current members) | 201 `CommentDTO` | 409 `duplicate_id`; 422 |
| `GET /v1/tasks/{taskId}/comments?cursor=&limit=` | member | — | `{ "items": [CommentDTO], "nextCursor": … }` oldest first, deleted omitted | — |
| `PATCH /v1/comments/{commentId}` | author | `{"operationId":"…","expectedVersion":1,"body":"Done."}` | 200 `CommentDTO` with `editedAt` | 403; 409 `version_conflict` |
| `DELETE /v1/comments/{commentId}` | author or admin+ | — | 204; soft delete (no-op when deleted) | 403 |

### 5.9 Devices (D07)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `PUT /v1/devices/{installationId}` | session | `{"apnsToken":"<hex>","environment":"sandbox","timeZone":"Europe/Rome"}`; `installationId` must be the session's own | 200 `DeviceDTO`; bound to the session, deleted with it | 422 |
| `DELETE /v1/devices/{installationId}` | session | — | 204 (no-op) | — |

### 5.10 Events and activity (D13, D17)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `GET /v1/projects/{projectId}/events?after=&limit=` | member | `after` required; `limit` max 500 | 200 `{"items":[ActivityEventDTO],"latestSequence":1190}` ascending; continue with `after=<last sequence>` until the last item equals `latestSequence` | 410 `sequence_too_old` |
| `GET /v1/projects/{projectId}/activity?before=&limit=` | member | `before` omitted = newest | 200 `{"items":[ActivityEventDTO]}` descending; page with `before=<smallest sequence received>`; an empty page ends | — |
| `WS /v1/realtime` | session | section 8 | — | 401 before upgrade |

### 5.11 Claude connection and audit (D15)

| Endpoint | Role | Request | Success | Distinct errors |
|---|---|---|---|---|
| `POST /v1/mcp/grants` | session | `{"operationId":"…","clientName":"Claude Desktop on the Mac mini","scopes":["projects:read","tasks:read","tasks:write"]}`; `members:manage` only when listed explicitly | 201 `{"grant": GrantDTO, "token": "skct_…"}` shown once; 90-day expiry | 422 (unknown scope) |
| `GET /v1/mcp/grants` | session | — | `{ "items": [GrantDTO] }` | — |
| `DELETE /v1/mcp/grants/{grantId}` | session | — | 204 (no-op when revoked) | — |
| `GET /v1/mcp/audit?cursor=&limit=` | session | — | `{ "items": [ { "id", "at", "clientName", "tool", "argsRedacted", "projectId", "changeId", "outcome", "errorCode", "durationMs" } ], "nextCursor": … }` newest first, own calls only | — |

## 6. Move contract (D06)

`POST /v1/tasks/{taskId}/move`, role `editor+`. **`afterTaskId` names the card that will sit immediately above (precede) the moved card; `beforeTaskId` names the card immediately below (follow) it.** Both `null` appends to the bottom; that is also the only valid pair for an empty column. Move to top: `afterTaskId: null`, `beforeTaskId: <first active task>`. The client computes the pair from its cached `(rank, id)` order and places the card optimistically with the same `RankKey` generator the server uses.

```json
{
  "operationId": "0f000000-0000-4000-8000-000000000042",
  "destinationColumnId": "0c000000-0000-4000-8000-000000000003",
  "beforeTaskId": "0d000000-0000-4000-8000-000000000007",
  "afterTaskId": "0d000000-0000-4000-8000-000000000005",
  "expectedTaskVersion": 12
}
```

Server algorithm, under `SELECT ... FOR UPDATE` on the project row, which serializes every write in the project:

1. Authorize `moveTask`; the project must not be archived (409 `project_archived`).
2. The task must be active (409 `task_archived`). The destination must be an active column of the task's own project; an archived column or any column id from another project is 404 `not_found`. Neither neighbor may be the moving task (422 `validation_failed`).
3. Resolve neighbors tolerantly; adjacency is not required, so two concurrent inserts into one gap both succeed in a deterministic order. A neighbor id from another project is 404. Both neighbors present, active and in the destination column but no longer adjacent: place immediately after `afterTaskId` (or immediately before `beforeTaskId` when `afterTaskId` is null). Exactly one neighbor missing (archived, moved elsewhere, deleted): use the other. Both missing: 409 `neighbors_changed` with `details.currentOrder` (the destination column's active task ids in order).
4. Apply the section 3 version rule for `{columnId, rank}`: a stale `expectedTaskVersion` is tolerated unless a `task.moved` event exists after it, in which case 409 `position_conflict`.
5. Generate the key between the resolved neighbors. If it exceeds 32 characters, renormalize the destination column only: every active task, in its new order, receives fresh keys (two-pass, versions untouched). Set `completedAt = now()` when entering the done column and `null` when leaving it (D19). Bump `task.version`, write `task.moved` with `changes {columnId, rank, completedAt?}` and `extra {fromColumnId, toColumnId, wipExceeded, completionChange ∈ completed|reopened|null}`, then `column.ranks_normalized` if needed, then the idempotency record.

Response 200, with `X-Project-Sequence` equal to `sequence`:

```json
{
  "task": { "id": "0d000000-0000-4000-8000-000000000001", "columnId": "0c000000-0000-4000-8000-000000000003", "rank": "a2V", "version": 13, "completedAt": null, "...": "full TaskDTO" },
  "renormalizedRanks": null,
  "sequence": 1187
}
```

When renormalization occurred, `renormalizedRanks` lists every active task in the destination column (`[{ "taskId": "…", "rank": "a0" }, { "taskId": "…", "rank": "a1" }]`), `task.rank` is the moved task's final key, and the `column.ranks_normalized` event carries `sequence + 1`. WIP is never enforced here: `extra.wipExceeded` is true when the destination's active count after the move exceeds its `wipLimit`, and the client shows the warning.

Retry. Sending the identical body with the same `operationId` (after a timeout, an app kill or a reconnect) returns the identical status and body:

```
HTTP/1.1 200 OK
Idempotency-Replayed: true

{ "task": { "id": "0d000000-0000-4000-8000-000000000001", "version": 13, "rank": "a2V", "...": "unchanged" }, "renormalizedRanks": null, "sequence": 1187 }
```

No second event, no push, and the `sequence` in the replayed body may be older than the project's current sequence (harmless). A retry of a stored 409 replays the 409.

| Error | When | Client |
|---|---|---|
| 400 `invalid_request` | Missing field; `expectedTaskVersion` ahead of the server | Reload the snapshot |
| 403 `forbidden` | Viewer | Hide edit affordances |
| 404 `not_found` | Task, destination column or neighbor not in the caller's project; archived destination | Refresh the column |
| 409 `task_archived` | The moved task is archived | Offer Restore |
| 409 `project_archived` | Project archived | Read-only |
| 409 `neighbors_changed` | Both neighbors gone | Retry once from `details.currentOrder`, then snap |
| 409 `position_conflict` | Someone else moved this card after `expectedTaskVersion` | Snap to `details.current`, show "Moved by Sam a moment ago" |
| 422 `validation_failed` | Self as neighbor; non-UUID ids | Client bug |
| 422 `idempotency_mismatch` | Same `operationId`, different body | Client bug |

## 7. Column constraints (D19)

- A project has 3 or 4 active columns at every commit, enforced in the service layer under the project lock (422 `column_count_invalid`) and by a deferrable constraint trigger. Project creation is atomic with its template columns; `custom` requires 3 or 4 titles.
- Create appends after the last active column. Archive (`DELETE /v1/columns/{id}`) refuses when 2 would remain, and when active tasks remain unless `moveTasksTo` names another active column of the same project (422 `column_not_empty`); moved tasks are appended to that column's bottom with `task.moved` events. Archiving the done column clears `isDone`. Restore refuses at 4. Columns are never deleted, so archived tasks always point at an existing column.
- Reorder takes the full ordered list of active column ids with `expectedProjectVersion` (strict equality), reassigns every column rank with `generateNKeysBetween`, bumps the project version and writes one `column.reordered` event with `extra.orderedColumnIds`.
- `wipLimit` is `null` or 1–99. It is advisory only: the server never rejects a create, move or restore on WIP, `task.moved.extra.wipExceeded` records the state, and the client renders `count/limit` with the warning treatment. Completed tasks count toward WIP like any active task.
- `isDone`: at most one active column; moving into it sets `completedAt`, moving out clears it; changing the flag never rewrites existing `completedAt` values; `isDone: null` in a PATCH is 422 (use `false`).

## 8. Realtime WebSocket (D13)

Connect: `GET /v1/realtime` with `Upgrade: websocket` and `Authorization: Bearer skat_…` on the upgrade request; an invalid or expired token is HTTP 401 before the upgrade; a fourth socket on one session is HTTP 429. Messages are JSON, one object per text frame, discriminated by `type`; a binary or unparsable frame draws `error {code: "invalid_request"}`. At most 20 subscriptions per socket; the 21st draws `error {code: "rate_limited"}`.

| Direction | Message | Example |
|---|---|---|
| client → server | `subscribe` | `{"type":"subscribe","projectId":"0a000000-0000-4000-8000-000000000001"}` |
| client → server | `unsubscribe` | `{"type":"unsubscribe","projectId":"…"}` |
| client → server | `ping` | `{"type":"ping"}` |
| server → client | `subscribed` | `{"type":"subscribed","projectId":"…","sequence":1190}` after the membership check |
| server → client | `event` | below |
| server → client | `unsubscribed` | `{"type":"unsubscribed","projectId":"…","reason":"membership_revoked"}`; `reason ∈ membership_revoked\|project_unavailable` |
| server → client | `error` | `{"type":"error","code":"not_found","projectId":"…"}`; a non-member subscribe is `not_found`, never `forbidden` |
| server → client | `pong` | `{"type":"pong"}` |

Event envelope. The spec concepts map onto the register's shape: the event type is `event.action`, the payload is `event.changes` plus `event.extra`, `occurredAt` is `event.createdAt` and `actorId` is `event.actorUserId`.

```json
{
  "type": "event",
  "projectId": "0a000000-0000-4000-8000-000000000001",
  "sequence": 1187,
  "event": {
    "id": "0f100000-0000-4000-8000-000000000001",
    "projectId": "0a000000-0000-4000-8000-000000000001",
    "sequence": 1187,
    "actorUserId": "0e000000-0000-4000-8000-000000000002",
    "via": "app",
    "action": "task.moved",
    "entityType": "task",
    "entityId": "0d000000-0000-4000-8000-000000000001",
    "entityVersion": 13,
    "changes": { "columnId": { "from": "0c000000-0000-4000-8000-000000000002", "to": "0c000000-0000-4000-8000-000000000003" }, "rank": { "from": "a1", "to": "a2V" } },
    "extra": { "fromColumnId": "0c000000-0000-4000-8000-000000000002", "toColumnId": "0c000000-0000-4000-8000-000000000003", "wipExceeded": false, "completionChange": null },
    "operationId": "0f000000-0000-4000-8000-000000000042",
    "createdAt": "2026-10-01T12:34:56.789Z"
  }
}
```

Rules:

- Publication is post-commit only; nothing uncommitted is ever broadcast. Every event of a subscribed project is delivered, including the subscriber's own (recognised by `operationId`).
- Ordering and catch-up: the client applies a live event only when `sequence == lastSequence + 1`. After `subscribed`, or on any gap, it calls `GET /v1/projects/{id}/events?after=<lastSequence>&limit=500` until the last item equals `latestSequence`. Duplicate delivery is harmless because application is by sequence. 410 `sequence_too_old` (gap over 2,000 events) means reload `GET /v1/projects/{id}` and resume from its `sequence`. Events are retained forever (D17), so the cap is efficiency, not correctness.
- Heartbeat: the client sends `ping` every 30 s in the foreground; the server closes after 90 s of silence with 4408. The client reconnects with backoff 1, 2, 4, 8, 16, 30 s ± 20 % jitter, immediately on network path changes and on return to the foreground, re-subscribes every cached project and runs catch-up; it closes the socket 5 s after entering the background and polls `GET events` every 60 s while disconnected in the foreground.
- Membership revocation: removal, a role change to self, project archive and project deletion send `unsubscribed` and drop the subscription within one second; on `membership_revoked` the client removes the project subtree (D12); on `project_unavailable` it refreshes the project list and re-subscribes only if the project is still listed.
- Sockets outlive access tokens. Every revocation path (logout, per-session revoke, revoke-all, refresh reuse, account deletion, absolute expiry) closes that session's sockets, and the hub re-validates every open session every 60 s.

| Close code | Meaning | Client |
|---|---|---|
| 1001 | Server restart | Reconnect with backoff |
| 4401 | `session_revoked` | Forced sign-out (D08) |
| 4408 | No ping for 90 s | Reconnect |
| 4429 | Inbound limit exceeded | Reconnect with backoff |

## 9. Activity event catalog (D17)

`changes` is `{ "<dtoField>": { "from": …, "to": … } }`, keyed by DTO field names, string values truncated to 200 characters with `"truncated": true`. `extra` is action-specific. Never stored: emails, tokens, attachment names beyond 100 characters, full notes. Not recorded: reads, sign-ins, refreshes. Unknown actions decode as `.unknown` and still advance the sequence. `entityVersion` is null where no versioned record changed.

| Action | entityType | `changes` keys | `extra` | Push (D07) |
|---|---|---|---|---|
| `project.created` | project | `name` | `{templateId}` | — |
| `project.renamed` | project | `name` | — | activity |
| `project.archived`, `project.unarchived` | project | `archivedAt` | — | activity |
| `project.deleted`, `project.restored` | project | `deletedAt` | — | — |
| `project.ownership_transferred` | project | `ownerId` | `{fromUserId, toUserId}` | membership |
| `column.created` | column | `title, color, wipLimit, isDone` | — | activity |
| `column.updated` | column | any of `title, color, wipLimit, isDone` | — | activity |
| `column.reordered` | project | — | `{orderedColumnIds}` | activity |
| `column.archived` | column | `archivedAt`, `isDone` when cleared | `{moveTasksTo, movedTaskIds}` | activity |
| `column.restored` | column | `archivedAt` | — | activity |
| `column.ranks_normalized` | column (version null) | — | `{ranks: [{taskId, rank}]}` | — |
| `task.created` | task | `title, notes, assigneeId, dueAt, columnId, rank` (from null) | — | activity; assignment |
| `task.updated` | task | any of `title, notes, dueAt, assigneeId` | — | assignment when `assigneeId` changed, else activity |
| `task.moved` | task | `columnId, rank, completedAt?` | `{fromColumnId, toColumnId, wipExceeded, completionChange}` | activity |
| `task.archived` | task | `archivedAt` | — | activity |
| `task.restored` | task | `archivedAt, columnId?, rank` | — | activity |
| `task.deleted` | task (version null) | — | — | — |
| `comment.added` | comment | — | `{commentId, preview, mentionedUserIds}` | comment, mention |
| `comment.edited` | comment | `body` | `{commentId, preview}` | — |
| `comment.deleted` | comment | `deletedAt` | `{commentId}` | — |
| `attachment.added` | attachment (version null) | — | `{attachmentId, originalName ≤ 100, byteSize, mimeType}` | activity |
| `attachment.deleted` | attachment (version null) | — | `{attachmentId}` | activity |
| `member.joined` | member | — | `{userId, role, invitationId, inviterUserId}` | invitation (inviter, owner, admins) |
| `member.left`, `member.removed` | member (version null) | — | `{userId, role}` | membership |
| `member.role_changed` | member | `role` | `{userId, role, fromRole}` | membership |
| `invitation.created`, `invitation.revoked` | invitation (version null) | — | `{invitationId, role, expiresAt}` | — |
| `invitation.declined` | invitation (version null) | — | `{invitationId}` | invitation (inviter) |

`via` is `app`, `mcp` (actor is the confirming user; the feed renders "Sam via Claude") or `system` (assignee clearing on removal and deletion, actor null). Every version increment of an entity is recorded by exactly one event carrying that `entityVersion`; a PATCH changing three fields is one event with three `changes` keys; a renormalizing move is two events.

## 10. Attachment flow (D14)

1. Initiate. `POST /v1/tasks/{taskId}/attachments/initiate` checks role, the per-task count (20), project quota (2 GB), the global cap, free space and the allowlist (`image/jpeg`, `image/png`, `image/heic`, `image/heif`, `image/gif`, `image/webp`, `application/pdf`, `text/plain`, `text/markdown`, `video/mp4`, `video/quicktime`), then creates a `pending` row and answers:

```json
{
  "attachmentId": "0a200000-0000-4000-8000-000000000001",
  "uploadPath": "/v1/attachments/0a200000-0000-4000-8000-000000000001/content",
  "uploadToken": "skut_EXAMPLEONLYNOTATOKEN0000000000000000000000",
  "uploadExpiresAt": "2026-10-02T12:34:56.789Z"
}
```

The upload token is single-purpose, hashed at rest, bound to the attachment id, declared size, SHA-256, user and issuing session, valid 24 hours, and invalidated when that session logs out or is revoked or the account is deleted.

2. Upload. The app stages the file under `Application Support/uploads/<attachmentId>`, records a `StagedUpload` in the outbox store, and runs a background `URLSession` upload task from that file:

```
PUT /v1/attachments/0a200000-0000-4000-8000-000000000001/content HTTP/1.1
Authorization: Bearer skut_EXAMPLEONLYNOTATOKEN0000000000000000000000
Content-Type: image/heic
Content-Length: 2048000

<raw bytes>
```

No `operationId`, no JSON wrapper, no multipart. The server streams to `<key>.part`, enforces the declared byte size while streaming (413), sniffs the first 512 bytes and requires the sniffed family to match `declaredMime` (415), computes SHA-256 and compares it with the initiate value (422 `validation_failed`, `details.fields[0].path = "sha256"`), renames atomically, marks `complete`, emits `attachment.added` and returns 200 `AttachmentDTO`. The PUT is the completion step; there is no separate complete call.

3. Completion semantics. The PUT is idempotent by state: a re-PUT on a complete row with a matching SHA-256 returns 200 with the DTO and emits nothing; a different hash is 409 `version_conflict`. Pending rows older than 48 hours are deleted with their `.part` files. The attachment is visible to other members only after `attachment.added`.

4. Download. `GET /v1/attachments/{id}/content` with the normal access token requires membership, `state = complete` and no `deletedAt` (otherwise 404). Response headers: `Content-Type: <sniffed mime>`, `Content-Disposition: attachment; filename="<ascii-safe>"; filename*=UTF-8''<percent-encoded>`, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none'; sandbox`, `Cache-Control: private, no-store`; `Range` requests answer 206.

5. Delete. `DELETE /v1/attachments/{id}` by the uploader or admin+ sets `deletedAt`, emits `attachment.deleted` and returns 204; the worker unlinks the file 24 hours later. Permanent task deletion and project purge delete rows and files through the same worker path.

6. Limits. 25 MB per file, 20 per task, 2 GB per project, 10 pending uploads per user, uploads refused at 507 below 10 % or 10 GB free. Thumbnails are generated on the client only.

## 11. Account deletion (D09)

1. The app shows the consequences sheet and, for every owned project with other members, collects Transfer to a member or Delete project; solo-owned projects are deleted permanently with no restore.
2. `POST /v1/account/delete-challenge` returns `{nonce, expiresAt}` (+5 minutes, single use).
3. The app runs a fresh Sign in with Apple with that nonce and calls:

```json
{
  "identityToken": "<apple jwt>",
  "authorizationCode": "<code>",
  "nonce": "<hex>",
  "transfers": [ { "projectId": "0a000000-0000-4000-8000-000000000001", "newOwnerUserId": "0e000000-0000-4000-8000-000000000002" } ],
  "deletions": [ "0a000000-0000-4000-8000-000000000002" ]
}
```

The server verifies the identity token, requires the nonce to match an unconsumed challenge for this user, `sub` to equal the user's Apple subject and `iat` within 5 minutes (otherwise 401 `reauth_required`), and that every owned project with other members appears in `transfers` or `deletions` with a current member as target (otherwise 409 `owned_projects_require_transfer` with `details.projects`). Then, in one transaction: transfers through the ownership-transfer path, soft deletion of the chosen and solo projects (purged after 30 days, no restore), revocation of every session, device row, upload token, Claude grant and pending AI change, deletion of the user's pending invitations and memberships with `member.left` events and assignee clearing (`task.updated`, `via = system`), anonymization of the user row to "Deleted member", and an audit row. Response 204. Apple token revocation is best effort with worker retries and never blocks the response.

Client afterwards: wipe Keychain, both SwiftData stores, the attachment cache, pending local notifications and the APNs registration; show the sign-in screen with a one-time confirmation. Any later request with an old token is 401 `session_revoked`. A later Sign in with Apple with the same subject creates a brand-new user. Other members see `member.left` and the display name change; comments, attachments, tasks and event metadata remain attributed to "Deleted member".

## 12. MCP tool contracts (D15)

The MCP server runs inside the `api` process over the same services and `ProjectAuthorizer` as REST; it has no queries and no filesystem access of its own. Every call is bound to the grant's user and re-applies the project role checks, so a tool can never do what that user cannot do in the app. All ids are UUID strings validated against the caller's projects; cross-project ids are `not_found`. Schemas are strict (`additionalProperties: false`, undeclared parameters rejected), with the section 4 string caps and array caps of 50. Tool failures are returned as tool results with `isError: true` whose text content is the section 1.4 envelope; protocol-level failures (unknown tool, schema violation) are JSON-RPC errors (verify the exact mapping against the MCP specification version recorded in ADR-0005 at Phase 9).

| Tool | Scope | Risk | Confirmation | Input (`field: type`, `?` = optional) | Output |
|---|---|---|---|---|---|
| `list_projects` | `projects:read` | — | no | `includeArchived?: bool` (default false) | `{ projects: [{ id, name, myRole, archivedAt, activeTaskCount }] }` |
| `get_project` | `projects:read` | — | no | `projectId: uuid` | `{ project: ProjectDTO, columns: [ColumnDTO], members: [{ userId, displayName, role }] }` |
| `list_tasks` | `tasks:read` | — | no | `projectId: uuid; columnId?: uuid; state?: active\|archived\|all; assigneeId?: uuid; dueBefore?: timestamp; cursor?: string; limit?: int 1–50` | `{ tasks: [TaskDTO], nextCursor }` |
| `get_task` | `tasks:read` | — | no | `taskId: uuid` | `{ task: TaskDTO, comments: [CommentDTO], attachments: [AttachmentDTO] }` (metadata only, never bytes) |
| `search_tasks` | `tasks:read` | — | no | `query: string 1–200; projectId?: uuid; includeArchived?: bool; limit?: int 1–50` | `{ tasks: [TaskDTO], matchedIn: { <taskId>: [title\|notes\|comments] } }`; case-insensitive over title, notes and comment bodies |
| `list_project_members` | `members:read` | — | no | `projectId: uuid` | `{ members: [MemberDTO without notificationSettings] }` |
| `propose_create_task` | `tasks:write` | low; high if more than one | yes | `projectId: uuid; tasks: [{ columnId?: uuid; title: string 1–200; notes?: string ≤ 20,000; assigneeId?: uuid; dueAt?: timestamp }] 1–50` | PendingAIChange response |
| `propose_update_task` | `tasks:write` | low; medium if `assigneeId` changes or `dueAt` is cleared; high if more than one | yes | `updates: [{ taskId: uuid; title?: string; notes?: string\|null; assigneeId?: uuid\|null; dueAt?: timestamp\|null }] 1–50` (null clears, as `Patch<T>`) | PendingAIChange response |
| `propose_move_task` | `tasks:write` | low; high if more than one | yes | `moves: [{ taskId: uuid; destinationColumnId: uuid; placement?: top\|bottom }] 1–50` (default bottom; neighbors are resolved at confirmation through the section 6 contract) | PendingAIChange response |
| `propose_comment` | `comments:write` | low | yes | `taskId: uuid; body: string 1–5,000; mentionedUserIds?: [uuid] ≤ 50` | PendingAIChange response |
| `propose_create_project` | `projects:write` | medium | yes | `name: string 1–80; templateId: home-projects\|restaurants-to-try\|custom; titles?: [string 1–40] 3–4` | PendingAIChange response |
| `propose_invite_member` | `members:manage` | high | yes | `projectId: uuid; role: editor\|viewer\|admin; expiresInDays?: 1\|7\|30` | PendingAIChange response; the invitation link is returned only in the `confirm_change` result |
| `propose_remove_member` | `members:manage` | high | yes | `projectId: uuid; userId: uuid` | PendingAIChange response |
| `confirm_change` | `projects:read` plus the change's `requiredScope` | — | is the confirmation | `changeId: uuid; confirmationToken: string` | `{ changeId, state: "confirmed", sequence, results: [{ operationId, entityType, entityId, version }], invitationUrl? }` |
| `cancel_change` | `projects:read` | — | no | `changeId: uuid` | `{ changeId, state: "cancelled" }` |

Risk levels are display and audit information only; every proposal requires confirmation in release 1 and nothing auto-applies. Tools carry MCP annotations (`readOnlyHint` for reads, `destructiveHint` for `propose_remove_member`) so clients can prompt appropriately. There is no tool for ownership transfer, archive, deletion, column changes or role changes. Content returned by read tools has control characters stripped and is untrusted input to the model.

Proposal response (every `propose_*` tool), stored as a `PendingAIChange` for 10 minutes:

```json
{
  "changeId": "0f200000-0000-4000-8000-000000000001",
  "state": "pending",
  "tool": "propose_move_task",
  "projectId": "0a000000-0000-4000-8000-000000000001",
  "riskLevel": "low",
  "requiredScope": "tasks:write",
  "createdAt": "2026-10-01T12:34:56.789Z",
  "expiresAt": "2026-10-01T12:44:56.789Z",
  "confirmationToken": "EXAMPLEONLYNOTATOKEN0000000000000000000000",
  "operations": [
    { "operationId": "0f000000-0000-4000-8000-000000000043", "kind": "move_task", "taskId": "0d000000-0000-4000-8000-000000000001", "destinationColumnId": "0c000000-0000-4000-8000-000000000004", "placement": "bottom" }
  ],
  "diff": [
    { "entity": "task", "entityId": "0d000000-0000-4000-8000-000000000001", "title": "Fix the kitchen tap", "field": "columnId", "old": "In Progress", "new": "Complete" },
    { "entity": "task", "entityId": "0d000000-0000-4000-8000-000000000001", "title": "Fix the kitchen tap", "field": "completedAt", "old": null, "new": "<now at confirmation>" }
  ],
  "capturedVersions": { "0d000000-0000-4000-8000-000000000001": 13 }
}
```

Confirmation token rules:

- 32 CSPRNG bytes, base64url, returned once, stored only as a hash; bound to the pending change, the grant's user, the grant (`clientId`) and the 10-minute expiry.
- `confirm_change` atomically moves `pending → confirmed` (single use), checks the hash, user, client and expiry, re-runs full authorization at confirmation time (membership or role lost in between → `failed`, 403 `forbidden`), verifies that every captured version is unchanged (otherwise 409 `stale_proposal`, `details.reason = "versions_changed"`; the model must re-propose so the diff shown is exactly what is applied), then executes the operations in order through the normal services with the stored operation ids and `via = mcp`, and returns the committed `sequence`.
- A second confirmation is 409 `change_already_applied`; a wrong token is 403 `forbidden` and leaves the change pending; an expired or cancelled change is 409 `stale_proposal` with `details.reason`. `cancel_change` marks it `cancelled`. Pending proposals are not project events; only confirmed changes produce activity events, rendered as "Sam via Claude".
- Rate limits: 60 reads and 10 proposals per minute per grant; confirm and cancel count as proposals. Every call, including denials, is written to the audit log (`GET /v1/mcp/audit`) with arguments redacted and truncated to 4 KB, retained one year.
- Human veto: in release 1 the model receives the confirmation token and can call `confirm_change` itself, so the human opportunity to deny rests on the MCP client's per-call approval; the connector setup instructions say so. An in-app approval mode is a release-2 candidate.

Transport (ADR-0005). Release 1 serves Streamable HTTP at `http://127.0.0.1:8080/mcp`, loopback-only and excluded from every Funnel or tunnel path set, accepting only `Authorization: Bearer skct_…` connection tokens created in Settings → Connect Claude or with `Run mcp-token create`. `Run mcp-stdio` is a credential-free stdio bridge launched on the Mac mini by Claude Desktop or Claude Code that forwards messages to that endpoint with the token read from its environment variable (`SHAREDKANBAN_MCP_TOKEN`; name verified at Phase 9); it holds no database credentials and no filesystem access. Remote access (`https://kanban.example.invalid/mcp` on the public ingress profile with the MCP OAuth authorization profile, PKCE, audience-bound tokens and an in-app pairing code) is Phase 9b, built only if the owner wants claude.ai or phone access, against the specification version current then.
