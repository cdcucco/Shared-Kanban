# SharedKanban architecture

This is the Phase 0 technical architecture for SharedKanban. It applies the decisions in docs/decisions/decision-register.md (D01 to D22), cites them inline and never overrides them: where the two disagree, the register wins and this document has a bug. Wire contracts live in docs/api.md and threats in docs/threat-model.md. It is written for the owner, who approves Phase 0 by reading it, and for the future sessions (cloud Linux and Mac mini Cowork) that implement Phases 1 to 11 from it without guessing. No production code exists yet; the JSON, shell and Mermaid below are specification.

## 1. Overview and constraints

SharedKanban is a native SwiftUI iPhone and iPad app (iOS and iPadOS 18.0 minimum, Swift 6 strict concurrency, D20) backed by one Swift 6, Vapor 4, Fluent 4 binary and a PostgreSQL 17 database that run on a household Mac mini under Docker Compose with a launchd escape hatch (D11). The server is authoritative: the app keeps a disposable SwiftData cache and a durable outbox of queued mutations (D08, D12); every mutation is idempotent through a body `operationId` (D18); optimistic concurrency with field-level merge replaces any CRDT (D05); card order is a fractional-indexing string rank moved through a neighbor-based contract (D06). Every project-scoped write is serialized on the project row and allocates a gap-free per-project sequence, and one `activity_events` table feeds realtime delivery, catch-up, the activity feed, notifications and conflict detection (D13, D17). Apple is used only for Sign in with Apple and APNs (D10). Devices reach the Mac mini through one of three ingress profiles in order: LAN for development, Tailscale for the household beta and release-1 production, and a public hostname (Tailscale Funnel or Cloudflare Tunnel) only when a concrete need appears; a tunnel is transport, never authorization (D10). Claude reaches the same application service layer through an MCP adapter that lives inside the api process, has no SQL and no filesystem access, and can only propose changes that a separate confirmation applies (D15). Hard product constraints: exactly three or four active columns per project (D19), a non-drag alternative for every drag (D01, D02), explicit confirmation for destructive and membership-changing AI actions (D15), and no secrets in the repository (D11, D21).

```mermaid
flowchart TB
    subgraph app["iPhone and iPad SwiftUI app, iOS 18 minimum"]
        SIWA["AuthenticationServices sign-in plus Keychain session store"]
        DTO["SharedDTOs package, Codable types, RankKey, templates, error codes"]
        SYNC["SyncEngine, the one ModelActor that writes"]
        CACHE[("SwiftData cache.store, disposable")]
        OUTBOX[("SwiftData outbox.store, pending operations and staged uploads")]
        REST["URLSession REST client"]
        WSC["URLSessionWebSocketTask realtime client"]
        NOTIF["UserNotifications, local reminders, APNs registration"]
        BGU["Background URLSession attachment uploads"]
    end
    subgraph ingress["Ingress, one profile active at a time, transport only"]
        LAN["Profile 1 LAN dev, plain http on port 8080, Debug builds only"]
        TS["Profile 2 Tailscale serve, TLS on the tailnet, release-1 production"]
        PUB["Profile 3 Tailscale Funnel or Cloudflare Tunnel, public https, only when needed"]
    end
    subgraph mini["Mac mini"]
        subgraph compose["Docker Compose"]
            API["api, Vapor REST, WebSocket upgrade and loopback MCP endpoint"]
            SVC["Application service layer, ProjectAuthorizer, ConcurrencyGate, Mutation.commit"]
            HUB["RealtimeHub actor, post-commit fan-out"]
            MCP["MCP adapter, same services, no SQL, no files"]
            WRK["worker, APNs outbox and maintenance jobs"]
            MIG["migrate, explicit one-shot"]
            PG[("PostgreSQL 17, internal network only")]
        end
        VOL[("Attachment volume, bind-mounted host directory")]
        BK["Backup job, host launchd, pg_dump plus restic"]
        BR["mcp-stdio bridge, host process, no credentials"]
        CL["Claude Desktop or Claude Code"]
    end
    subgraph apple["Apple, reached outbound over HTTPS"]
        AID["Sign in with Apple endpoints, JWKS, token, revoke"]
        APNS["APNs"]
    end
    SIWA -.->|"system sign-in sheet"| AID
    SIWA --> REST
    SYNC --> CACHE
    SYNC --> OUTBOX
    SYNC --> REST
    SYNC --> WSC
    DTO -.- REST
    DTO -.- API
    REST -->|"HTTPS JSON under v1"| ingress
    WSC -->|"WSS v1 realtime"| ingress
    BGU -->|"PUT attachment content with upload token"| ingress
    ingress -->|"loopback port 8080"| API
    API --> SVC
    API -->|"socket upgrade and subscribe"| HUB
    API --> MCP
    MCP --> SVC
    SVC --> PG
    SVC --> VOL
    SVC -->|"after commit"| HUB
    HUB -.->|"event frames"| WSC
    WRK --> PG
    WRK --> VOL
    WRK -->|"HTTP/2 token auth"| APNS
    APNS -.->|"push"| NOTIF
    NOTIF -->|"device token"| REST
    MIG --> PG
    API -->|"JWKS, code exchange, revoke"| AID
    BK --> PG
    BK --> VOL
    CL -->|"stdio"| BR
    BR -->|"Streamable HTTP on loopback with skct token"| MCP
```

The app talks to one origin over HTTPS and WSS; the active ingress profile forwards to the loopback-only `api` container, the only process that serves clients. `api` hosts the Vapor routes, the realtime hub and the MCP endpoint in one process, all calling the same service layer; `worker` shares the binary and database but never serves clients or publishes events (D11, D13). Claude connects through a credential-free stdio bridge to the loopback MCP endpoint (D15).

## 2. Component responsibilities

Each component lists what it owns, what it must never do, and its interfaces. "Service layer" means 2.10, the only code that reads or writes domain state, shared by REST and MCP (D15).

### 2.1 SharedDTOs package

- Owns: every request, response, event and WebSocket frame type as plain `Codable` structs; `Patch<T>`, `Page<T>`, `APIError` and the error-code enum; the action enum with `.unknown`; `ProjectTemplate`; `RankKey`; `JSONCoding`; `LenientEnum` (D06, D19, D22). Declares iOS 18, macOS 15 and Linux (D20).
- Must never: import SwiftData, Vapor or Fluent; contain floats (D18 hashing); hold business logic.
- Interfaces: compiled into app, server and tests; fixtures under `shared/Fixtures/` are decoded by contract tests on both platforms (D22).

### 2.2 Sign-in and session store (AuthenticationServices and Keychain)

- Owns: the native Sign in with Apple request with a nonce; the fresh re-authentication for account deletion (D09); one non-synchronizable Keychain blob (`installationId`, tokens, expiries) under `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly`; single-flight proactive refresh under 5 minutes remaining, one retry on `token_expired`, forced sign-out handling (D16); the launch-time `getCredentialState` check (D09); the first-launch wipe of Keychain items left by an uninstall (D12).
- Must never: store Apple's identity or refresh token on the device; put tokens in URLs or logs; sync Keychain through iCloud; wipe the outbox on a forced sign-out (D08).
- Interfaces: `/v1/auth/*`; supplies the bearer header to REST and the WebSocket upgrade; uploads use upload tokens instead (D14).

### 2.3 SwiftData cache, outbox and SyncEngine

- Owns: the disposable `cache.store` (`Cached*` models, `SyncState`) and the durable `outbox.store` (`PendingOperation`, `StagedUpload`); `SyncEngine`, the single `@ModelActor` that performs every write; the draft overlay; snapshot and event application in sequence order; FIFO replay; eviction after 30 days unopened; the `CacheSchema.version` check that deletes a mismatched cache; `ProjectRepository` with an in-memory fake (D08, D12).
- Must never: be the source of truth; expose `@Model` objects to the APIClient or DTOs; migrate the cache (wipe instead); gate events on versions; put the outbox container in the SwiftUI environment; keep attachment bytes in SwiftData (a 200 MB LRU under Caches does).
- Interfaces: `@Query` sorted by `(rank, id)` with `.lexical`; `@Observable SyncStatus`; one REST call in flight from the outbox.

### 2.4 URLSession REST client

- Owns: the typed `APIClient`; the `Authorization`, `X-Request-Id` and `X-Client-Version` headers; decoding the error envelope into typed errors; backoff for 5xx, network errors and 429 (1 s doubling to 60 s, 20 % jitter); polling `GET events` every 60 s while the socket is down (D08, D13, D22).
- Must never: mint a new `operationId` for a retry; reorder outbox operations; trust push or deep-link content without re-fetching (D07).
- Interfaces: the `/v1` surface in docs/api.md; a stub `URLProtocol` in contract tests.

### 2.5 WebSocket client

- Owns: one `URLSessionWebSocketTask` to `WSS /v1/realtime` with the access token in the upgrade header; per-project `subscribe`; a `ping` every 30 s; gap detection with handoff to REST catch-up; reconnect backoff 1 to 30 s with jitter, immediate reconnect on `NWPathMonitor` and `scenePhase` changes; close 5 s after backgrounding (D13).
- Must never: put the token in the query string; apply an event out of order; replay history over the socket; open in the background.
- Interfaces: the frame protocol in docs/api.md (`subscribe`, `unsubscribe`, `ping`; `subscribed`, `event`, `unsubscribed`, `error`, `pong`) and close codes 4401, 4408, 4429.

### 2.6 Notifications (UserNotifications, local reminders, APNs registration)

- Owns: local `UNCalendarNotificationTrigger` reminders `due:<taskId>:soon` and `due:<taskId>:now` for the user's assigned tasks and unassigned tasks they created, at most 50 pending, rescheduled after sync, foreground, time-zone change, push receipt and `BGAppRefreshTask`, shifted past quiet hours; the permission prompt at the first useful moment; `PUT /v1/devices/{installationId}` with APNs token, environment and time zone; deep links that re-check authorization; the 24-hour watchdog notification (D07, D21).
- Must never: schedule server-driven due reminders; show payload content without re-fetching; prompt on first launch; use provisional authorization; show a badge.
- Interfaces: system APNs registration; `/v1/devices`; the payload `sk` object (D07).

### 2.7 Background URLSession transfers

- Owns: the background session `app.sharedkanban.uploads`; staging under `Application Support/uploads/<attachmentId>` with location EXIF stripped; one file-based `PUT /v1/attachments/{id}/content` with the upload token as bearer; `handleEventsForBackgroundURLSession`; Progress, Retry and Cancel; the download cache (D14).
- Must never: use the access token; load a 25 MB file into memory; start a PUT offline; call a completion endpoint (the PUT completes).
- Interfaces: initiate, content PUT, content GET with `Range`, DELETE (D14).

### 2.8 Ingress (LAN, Tailscale serve, Funnel or Cloudflare Tunnel)

- Owns: transport to the loopback-only api: plain LAN http for Debug builds (profile 1); `tailscale serve` with a Let's Encrypt certificate for the tailnet hostname (profile 2); Tailscale Funnel scoped to `/v1/*`, `/invite`, `/.well-known/*`, `/health` and, after Phase 9b, `/mcp`, or `cloudflared` with an owned domain (profile 3) (D10).
- Must never: be relied on for authorization or IP allowlisting; expose PostgreSQL; expose `/mcp` before Phase 9b; see the invitation token (URL fragment, D04).
- Interfaces: 443 forwarded to `127.0.0.1:8080`; WebSocket upgrades and 25 MB bodies must pass (verify at Phase 10).

### 2.9 Vapor API

- Owns: routing under `/v1`; the content-free `/invite` page and the AASA (D04); `/health` and `/ready` (D11); the loopback `/mcp` endpoint (D15); the `/v1/realtime` upgrade (D13); bearer verification as one indexed session lookup plus the user row (D16); request canonicalization and idempotency middleware (D18); rate limits, the error envelope middleware, security headers and the 426 minimum-version gate (D22); JSON logs with redaction (D11).
- Must never: contain domain rules; auto-run migrations on `serve`; use `FileMiddleware` (D14); answer anything but `not_found` for ids outside the caller's projects (D22); log raw tokens.
- Interfaces: HTTPS JSON, WSS and MCP Streamable HTTP inbound; outbound HTTPS to Apple for JWKS, `/auth/token` and `/auth/revoke` (D10).

### 2.10 Application service layer

- Owns: every domain operation as a method taking an actor; `ProjectAuthorizer.require(capability, project:, user:)` over the `Capability` enum (D03); the `ConcurrencyGate` shared by PATCH and `/move` (D05, D06); `Mutation.commit`, which inside one transaction takes the project lock, writes the entity change, allocates the sequence, writes the events and the operation record and registers post-commit publication (D17, D18); the column invariant, templates, done-column and WIP rules (D19); invitations, roles, removal, transfer and account deletion (D03, D04, D09); attachment metadata and quotas (D14); the PendingAIChange engine (D15); the `AppleIdentityProvider` and `PushProvider` protocols with fakes (D07, D09).
- Must never: publish before commit; write a mutation without an event; hold the project lock during a file transfer (D17); acknowledge an id from another project.
- Interfaces: called by the Vapor controllers and the MCP adapter; Fluent and SQLKit; the attachment volume; the after-commit hook.

### 2.11 Realtime hub

- Owns: the in-process `RealtimeHub` actor keyed by project and by session; the membership check on `subscribe`; post-commit fan-out; `evict(userId:projectId:)` within one second on removal, self role change, archive and deletion; closing a session's sockets with 4401 on every revocation; re-validating open sessions every 60 s; the limits of 3 sockets per session, 20 subscriptions per socket, 10 inbound messages per second and 64 KB per frame (D13).
- Must never: publish uncommitted data; serve catch-up; answer `forbidden` to a non-member subscribe (always `not_found`); accept a query-string token; run as more than one instance.
- Interfaces: the after-commit hook; revocation callbacks (D16); the frame protocol in 2.5.

### 2.12 Worker

- Owns: the APNs outbox `notification_deliveries` (unique on `(event_id, user_id)`, drained with `FOR UPDATE SKIP LOCKED`); per-project cursors in `worker_cursors` polled every 2 s; collapse ids, passive delivery in quiet hours, the activity throttle, the actor exclusion; device inactivation on APNs 410 or `BadDeviceToken`; the hourly and daily maintenance jobs (attachment cleanup, project purge, Apple revoke retries, session and operation purges); backup and disk status for `/ready` and owner alerts (D07, D09, D11, D14, D21).
- Must never: write project events; push for due dates; push to an event's actor; auto-archive tasks (D19).
- Interfaces: PostgreSQL; HTTP/2 to APNs through `PushProvider`; the attachment volume.

### 2.13 MCP adapter and stdio bridge

- Owns: the MCP server inside the api process at `http://127.0.0.1:8080/mcp`, accepting only `skct_` tokens; six read tools, seven `propose_*` tools, `confirm_change` and `cancel_change`; strict JSON Schemas (`additionalProperties: false`, title 200, notes 20,000, comment 5,000, arrays 50); one required scope per tool; 60 reads and 10 proposals per minute per grant; `mcp_audit_log` rows for every call including denials; the `Run mcp-stdio` bridge, a host process launched by Claude Desktop or Claude Code that forwards MCP messages to the loopback endpoint with the connection token from its environment (D15).
- Must never: query PostgreSQL or touch the filesystem (the bridge holds no credentials at all); apply anything without `confirm_change`; bypass `ProjectAuthorizer`; be reachable through Funnel or a tunnel before Phase 9b; offer ownership transfer or permanent deletion (D03).
- Interfaces: MCP Streamable HTTP on loopback; stdio to the client; `GET /v1/mcp/audit`; `Run mcp-token`.

### 2.14 PostgreSQL

- Owns: all durable state (3.1); the single-owner index and trigger (D03); the 3-to-4 column trigger and the single `is_done` index (D19); the `(column_id, rank)` partial unique index under `COLLATE "C"` (D06); `(project_id, sequence)` uniqueness (D17); the `(user_id, operation_id)` key (D18). It is the only queue (D11).
- Must never: publish a port outside the Compose network (`127.0.0.1:5432` under `dev` only); hold attachment bytes; be migrated by `serve`.
- Interfaces: Fluent and SQLKit from api, worker and migrate; `pg_dump -Fc` from the backup job.

### 2.15 Attachment volume

- Owns: `/Users/kanban/SharedKanban/data/attachments`, bind-mounted at `/data/attachments` (0700) into api and worker; files at `<k[0..2]>/<k[2..4]>/<k>` for a 64-hex key validated against `^[0-9a-f]{64}$` before any path is built; `.part` files; the orphan quarantine (D14).
- Must never: contain user-named files or extensions; be served by path; be written by anything but the upload handler and the worker.
- Interfaces: streamed writes, atomic rename and `Range` reads by the api; cleanup by the worker; restic snapshots.

### 2.16 Backup job

- Owns: `backup.sh` under host launchd every 6 hours (`pg_dump -Fc` via `docker compose exec`, last 8 dumps kept, `restic backup` of dumps and attachments to the external SSD); the nightly `restic copy` to iCloud Drive (B2 as the alternative); retention 14 daily, 8 weekly, 12 monthly; weekly `restic check`; `restore.sh` into the disposable `sharedkanban-drill` stack with blank Apple credentials; monthly automated and quarterly observed drills; `health.sh` every 5 minutes; `export-secrets.sh` (D21).
- Must never: include the secrets directory in a repository; run in a container; push or revoke anything during a drill; add a second encryption layer.
- Interfaces: the postgres service, the attachments directory, `/ready`, the System Status screen through the worker.

### 2.17 Sign in with Apple endpoints (Apple)

- Provides: identity tokens; JWKS at `appleid.apple.com` cached 24 h; `/auth/token` code exchange returning Apple's refresh token; `/auth/revoke`; server-to-server events to `POST /v1/auth/apple/notifications` once a public hostname exists (D09, D10).
- The system must never: expect name or email after the first authorization; store Apple's refresh token unencrypted (AES-GCM under `APPLE_TOKEN_KEY`); call live Apple endpoints from tests.
- Interfaces: `AppleIdentityProvider`; JWTKit verification of signature, issuer, audience, expiry and nonce; the `client_secret` JWT signed with the `.p8` key (verify the format at Phase 3).

### 2.18 APNs (Apple)

- Provides: push delivery in the sandbox and production environments.
- The system must never: put notes, tokens or emails in a payload; exceed the title plus 80 characters of comment text; rely on silent pushes; push every card move by default.
- Interfaces: token-based HTTP/2 with the APNs `.p8` key through `PushProvider`; `apns-collapse-id`, `interruption-level: passive` in quiet hours, `thread-id` equal to the project id (D07).

## 3. Data model

Ids are UUID v4, lowercase, compared case-insensitively (D22). Tables are plural snake_case; DTO keys are camelCase and the same identifiers key the `changes` object of events (D17). "Mutable record" means project, column, task, comment and member, the five records that carry `version`, `updated_at` and `updated_by` (D05). Timestamps are `timestamptz`, serialized as ISO-8601 UTC with milliseconds (D22).

### 3.1 Entities

| Entity | Table | Fields | Notes |
|---|---|---|---|
| User | `users` | `id`, `apple_subject` (unique, null after deletion), `display_name`, `email` (nullable), `apple_refresh_token_ciphertext` (AES-GCM under `APPLE_TOKEN_KEY`), `apple_revoked_at`, account notification preferences (category toggles, `dueLeadTime`, quiet hours), `created_at`, `deleted_at` | Name and email are captured at first sign-in only; deletion anonymizes in place and keeps the row forever (D09). The register fixes the account-level notification settings but not their column layout; Phase 2 decides (D07). |
| Session | `sessions` | `id`, `user_id`, `installation_id`, `access_token_hash`, `access_expires_at`, `refresh_token_hash`, `refresh_expires_at`, `absolute_expires_at`, `previous_refresh_token_hash`, `previous_valid_until`, `rotated_at`, `device_name`, `device_model`, `os_version`, `app_version`, `last_ip_prefix`, `created_at`, `last_used_at`, `revoked_at`, `revoke_reason` | One per installation; hashes only (D16). |
| Project | `projects` | `id`, `name`, `owner_id`, `last_sequence`, `archived_at`, `deleted_at`, `version`, `created_at`, `updated_at`, `updated_by` | `last_sequence` is the per-project event counter (D17); exactly one owner (D03). |
| ProjectMember | `project_members` | `project_id`, `user_id`, `role` (owner, admin, editor or viewer), `notification_settings` (`muted`, per-category `overrides`), `joined_at`, `version`, `updated_at`, `updated_by` | Composite key; versioned so concurrent role changes conflict (D03, D05, D07). |
| Invitation | `invitations` | `id`, `project_id`, `role` (admin, editor or viewer), `inviter_id`, `token_hash` (SHA-256 hex, unique), `expires_at`, `accepted_at`, `accepted_by_user_id`, `declined_at`, `revoked_at`, `created_at` | The plaintext token is never stored (D04). |
| Column | `columns` | `id`, `project_id`, `title` (1 to 40 chars), `rank` (`TEXT COLLATE "C"`), `color` (semantic token), `wip_limit` (null or 1 to 99), `is_done`, `version`, `archived_at`, `created_at`, `updated_at`, `updated_by` | Archived, never deleted; 3 to 4 active per project; at most one `is_done` (D06, D19). |
| Task | `tasks` | `id`, `project_id`, `column_id`, `title`, `notes`, `rank` (`TEXT COLLATE "C"`), `assignee_id`, `due_at`, `completed_at`, `created_by`, `version`, `archived_at`, `created_at`, `updated_at`, `updated_by` | Clients may supply `id` (D08); `completed_at` is derived from the done column (D19); `created_by` serves the local-reminder rule (D07). |
| Attachment | `attachments` | `id`, `task_id`, `project_id` (denormalized for authorization), `uploader_id`, `storage_key` (64 lowercase hex), `original_name` (sanitized, 255 max), `declared_mime`, `sniffed_mime`, `byte_size`, `sha256`, `state` (pending or complete), `upload_token_hash`, `upload_expires_at`, `session_id`, `created_at`, `completed_at`, `deleted_at` | Bytes live on the volume, never in PostgreSQL (D14). |
| Comment | `comments` | `id`, `task_id`, `project_id`, `author_id`, `body`, `mentioned_user_ids` (validated members), `version`, `edited_at`, `deleted_at`, `created_at`, `updated_at`, `updated_by` | `body` is the only patchable field, so concurrent edits always conflict (D05). |
| ActivityEvent | `activity_events` | `id`, `project_id`, `sequence`, `actor_user_id` (null for system), `via` (app, mcp or system), `action`, `entity_type`, `entity_id`, `entity_version` (nullable), `changes` (jsonb, field to `{from, to}`), `extra` (jsonb), `operation_id` (nullable), `created_at` | Unique `(project_id, sequence)`; index `(entity_type, entity_id, entity_version)`; kept forever (D17). |
| Device | `devices` | `id`, `installation_id`, `user_id`, `session_id`, `apns_token`, `environment` (sandbox or production), `time_zone` (IANA), `is_active`, `last_seen_at`, `created_at`, `updated_at` | Full replacement by `PUT /v1/devices/{installationId}`; inactive after APNs 410; deleted with its session (D07, D16). |
| PendingAIChange | `pending_ai_changes` | `id`, `user_id`, `client_id`, `project_id`, `tool`, `operations` (typed list, each with a pre-generated `operationId`), `diff` (list of `{entity, field, old, new}`), `captured_versions`, `risk_level`, `required_scope`, `confirmation_token_hash`, `state` (pending, confirmed, cancelled, expired or failed), `created_at`, `expires_at` (+10 minutes), `confirmed_at`, `result_sequence` | Never a project event until confirmed (D15). |
| Operation | `operations` | `user_id`, `operation_id`, `request_hash`, `status_code`, `response_body` (jsonb), `created_at`, `completed_at` | Primary key `(user_id, operation_id)`; 30-day TTL (D18); see 3.7. |
| Supporting rows | `notification_deliveries`, `worker_cursors`, `mcp_grants`, `mcp_audit_log`, delete challenges, Apple revoke retries, security audit rows | `notification_deliveries {event_id, user_id, category, state, attempts, created_at, sent_at}` unique on `(event_id, user_id)` (D07); `worker_cursors {job, project_id, last_sequence}` (D17); `mcp_grants {id, user_id, client_id, token_hash, scopes, created_at, expires_at, last_used_at, revoked_at}` and `mcp_audit_log {id, at, user_id, client_id, tool, args_redacted, project_id, change_id, outcome, error_code, duration_ms, remote_addr}` (D15); the single-use 5-minute deletion challenge, the hourly Apple revoke retry record and the `security.session_reuse` and deletion audit rows (D09, D16) | The register defines the behavior of the last three but not their layout; Phase 3 fixes it. |

### 3.2 Relationships

```mermaid
erDiagram
    USER ||--o{ SESSION : has
    SESSION ||--o{ DEVICE : registers
    USER ||--o{ PROJECT : owns
    PROJECT ||--o{ PROJECT_MEMBER : has
    USER ||--o{ PROJECT_MEMBER : holds
    PROJECT ||--o{ INVITATION : issues
    USER ||--o{ INVITATION : creates
    PROJECT ||--|{ COLUMN : orders
    PROJECT ||--o{ TASK : contains
    COLUMN ||--o{ TASK : holds
    USER |o--o{ TASK : assigned
    TASK ||--o{ ATTACHMENT : has
    TASK ||--o{ COMMENT : has
    USER ||--o{ COMMENT : writes
    PROJECT ||--o{ ACTIVITY_EVENT : logs
    USER ||--o{ OPERATION : sends
    OPERATION |o--o{ ACTIVITY_EVENT : produces
    USER ||--o{ PENDING_AI_CHANGE : proposes
    USER ||--o{ MCP_GRANT : pairs
    ACTIVITY_EVENT ||--o{ NOTIFICATION_DELIVERY : queues
    USER ||--o{ NOTIFICATION_DELIVERY : receives
    USER {
        uuid id PK
        text apple_subject UK "null after deletion"
        timestamptz deleted_at "tombstone kept forever"
    }
    SESSION {
        uuid id PK
        uuid user_id FK
        uuid installation_id "one session per installation"
        text previous_refresh_token_hash "60 s grace window"
        timestamptz revoked_at
    }
    DEVICE {
        uuid id PK
        uuid session_id FK
        text apns_token
        bool is_active
    }
    PROJECT {
        uuid id PK
        uuid owner_id FK
        bigint last_sequence "per-project counter"
        int version
        timestamptz deleted_at
    }
    PROJECT_MEMBER {
        uuid project_id PK "composite with user_id"
        uuid user_id PK
        text role
        int version
    }
    INVITATION {
        uuid id PK
        uuid project_id FK
        uuid inviter_id FK
        text token_hash UK "sha256 hex"
        timestamptz expires_at
    }
    COLUMN {
        uuid id PK
        uuid project_id FK
        text rank "COLLATE C"
        bool is_done "at most one per project"
        int version
        timestamptz archived_at
    }
    TASK {
        uuid id PK
        uuid column_id FK
        uuid assignee_id FK "nullable"
        text rank "COLLATE C"
        timestamptz completed_at "derived from the done column"
        int version
        timestamptz archived_at
    }
    ATTACHMENT {
        uuid id PK
        uuid task_id FK
        text storage_key UK "64 hex chars"
        text sha256
        text state "pending or complete"
    }
    COMMENT {
        uuid id PK
        uuid task_id FK
        uuid author_id FK
        int version
        timestamptz deleted_at
    }
    ACTIVITY_EVENT {
        uuid id PK
        uuid project_id FK
        bigint sequence "unique per project"
        text action
        int entity_version "nullable"
        jsonb changes
        uuid operation_id "nullable"
    }
    OPERATION {
        uuid user_id PK "composite with operation_id"
        uuid operation_id PK
        text request_hash
        int status_code
        jsonb response_body
    }
    PENDING_AI_CHANGE {
        uuid id PK
        uuid user_id FK
        uuid project_id FK
        jsonb captured_versions
        text state
        timestamptz expires_at
    }
    MCP_GRANT {
        uuid id PK
        uuid user_id FK
        text token_hash
        text scopes
    }
    NOTIFICATION_DELIVERY {
        uuid event_id PK "composite with user_id"
        uuid user_id PK
        text state "queued sent failed"
    }
```

The diagram shows keys and rule-carrying fields; 3.1 is the complete list.

### 3.3 Versioning fields

Every mutable record carries `version` (starts at 1, +1 per committed change), `updated_at` and `updated_by`, the actor of the committed write (D05). Every version increment is recorded by exactly one activity event carrying that `entity_version`; a mutation without an event, or an event with incomplete `changes` keys, is a test failure, because field-level merge (5.1) is computed from these events (D05, D17). Clients send `expectedVersion` on every PATCH, `expectedTaskVersion` on `/move`, `expectedProjectVersion` on `/columns/reorder` and the project's `expectedVersion` on `/transfer-ownership`, always beside an `operationId` (D03, D05, D06, D18). Renormalization rewrites `rank` without bumping any version and records a `column.ranks_normalized` event with a null `entity_version` (D06, D17). Users, sessions, invitations, attachments, devices and events carry no `version`; their changes are state transitions, not concurrent field edits.

### 3.4 Rank representation

`tasks.rank` and `columns.rank` are `TEXT NOT NULL COLLATE "C"` holding fractional-indexing keys over the 62-symbol alphabet `0-9A-Za-z`, generated by `RankKey.generateKeyBetween(a, b)` and `generateNKeysBetween(a, b, n)` in SharedDTOs and unit-tested on Linux and iOS (D06). Keys never end in the smallest symbol, so a key between any two neighbors always exists; repeated insertion at one spot grows the key by about one character each time. A partial unique index on `tasks (column_id, rank) WHERE archived_at IS NULL` guards active tasks. Every sort is byte-wise (`COLLATE "C"` on the server; `.lexical` or UTF-8 comparison on the client, never locale-aware) with `id` as the tiebreaker; Phase 2 tests include keys such as `a9` and `a10` on both platforms. The client runs the same generator, so an optimistic placement produces the key the server will accept. Column order uses the same keys: `/columns/reorder` reassigns every column rank with `generateNKeysBetween` and writes one `column.reordered` event with `extra.orderedColumnIds` (D06). Renormalization is in 5.4. Verify that `COLLATE "C"` is byte-wise on the pinned PostgreSQL major at Phase 2.

### 3.5 Sequence allocation

`projects.last_sequence` is the only counter. Every project-scoped mutation runs in one transaction that first executes `SELECT ... FOR UPDATE` on the project row, performs the domain writes, then runs `UPDATE projects SET last_sequence = last_sequence + 1 WHERE id = $1 RETURNING last_sequence` once per event; because the lock is held until commit, sequence order equals commit order and a rollback leaves no gap (D17). The result is a gap-free, strictly increasing per-project `bigint`, unique on `(project_id, sequence)`; PostgreSQL sequences (non-transactional, gappy) and global counters are not used. Consumers depend on gap-freeness: the client applies a live event only when `sequence == lastSequence + 1`, catch-up is `GET /v1/projects/{id}/events?after=`, and worker jobs find work by comparing `projects.last_sequence` with their `worker_cursors` row (D13, D17). The same lock serializes every write to a project, the deliberate single serialization point for moves (D06).

### 3.6 Soft deletion and retention

| Record | Marker and effect | Reversal | Purge |
|---|---|---|---|
| User | `deleted_at` plus anonymized fields; a non-member everywhere; comments, attachments, tasks and events stay attributed to "Deleted member" | None; the same Apple subject later creates a new user | Never (D09) |
| Project | `deleted_at`; hidden for everyone; pending invitations auto-revoked; subscriptions dropped | Owner restore within 30 days via `POST /v1/projects/{id}/restore`, except projects deleted by account deletion | Worker hard-deletes rows and files after 30 days (D03) |
| Project | `archived_at`; read-only, mutations answer 409 `project_archived` except unarchive, leave, member removal and notification settings | Unarchive by owner or admin | None (D03) |
| Column | `archived_at`; off the board, never deleted so archived tasks keep a column | `POST /v1/columns/{id}/restore` unless 4 are active | None (D19) |
| Task | `archived_at` via `DELETE /v1/tasks/{id}`; off the board, searchable under Archived | `POST /v1/tasks/{id}/restore` to the bottom of its column, or of the first column if its column is archived | Only `DELETE /v1/tasks/{id}?permanent=true` by owner or admin on an archived task, writing `task.deleted`; no automatic purge (D19) |
| Comment | `deleted_at`; body hidden, event preview fragments remain | None | None (D17) |
| Attachment | `deleted_at`; download 404, `attachment.deleted` event | None | File unlinked after 24 h; `pending` rows older than 48 h and `.part` files hourly; orphans quarantined weekly, deleted 7 days later (D14) |
| Invitation | `revoked_at`, `declined_at` or `accepted_at`; 410 `invitation_unavailable` with a reason | None | None (D04) |
| Session | `revoked_at`, `revoke_reason`; 401 `session_revoked`, sockets closed, upload tokens and device rows invalidated | None | Expired and revoked rows after 30 days (D16) |
| Device | `is_active = false`; no pushes | A later `PUT /v1/devices/{installationId}` replaces the row | Deleted with its session (D07) |
| Operation | none | none | 30 days, daily (D18) |
| PendingAIChange | `state`; cancelled, expired and failed proposals are inert | Re-propose | Not fixed by the register; audit rows are kept 1 year (D15) |
| ActivityEvent | kept forever | none | Only with the project at purge (D17) |

### 3.7 Idempotency records

Every mutating `POST` and `PATCH` carries `operationId` (UUID v4) in the body as part of the typed DTO; `DELETE` carries no body and is a strict state no-op when the target is already in the requested state (same status, no new event); `PUT` requests (attachment content, device registration, notification settings) are full replacements, idempotent by state; auth endpoints carry no `operationId`; the `Idempotency-Key` header is not used (D18). The `operations` row is keyed by `(user_id, operation_id)`, so the same UUID from another user is a different operation, and stores `request_hash` (SHA-256 of method, path and the canonical JSON body with sorted keys, explicit nulls preserved, no floats in any DTO), `status_code`, `response_body`, `created_at` and `completed_at`.

Inside the mutation transaction, after the project lock, the service runs `INSERT ... ON CONFLICT DO NOTHING RETURNING` on the operation row first; a concurrent duplicate blocks on the primary key until the first transaction finishes, so no in-progress state is needed. If the row was inserted, the mutation runs inside a `SAVEPOINT`: on success the row is completed with the status and body and the transaction commits; on a storable failure (409 or 422, except `idempotency_mismatch`) the savepoint is rolled back, the failure is stored and the transaction still commits; on any other failure (401, 403, 404, 429, 5xx, transport) everything rolls back and the retry is a fresh evaluation. If the row already existed, the same `request_hash` replays the stored status and body verbatim with `Idempotency-Replayed: true`, re-emitting no events and no pushes; a different hash returns 422 `idempotency_mismatch`. A replayed 409 stays a 409; after resolving a conflict the client mints a new `operationId`. Events carry `operation_id` so a client recognises its own echoes during catch-up. Rows are purged after 30 days, matching the outbox age-out (D08). MCP confirmations execute the per-operation ids generated at proposal time, so a double confirm cannot double-apply (D15). Whether Fluent plus SQLKit can express the savepoint pattern is verified at Phase 2.

## 4. Sequence diagrams

App is the iOS app, API the Vapor process, Svc the service layer, DB PostgreSQL, Hub the realtime hub; error codes are the D22 codes.

### 4a. Sign in with Apple and session issuance

```mermaid
sequenceDiagram
    participant App as iOS app
    participant ASA as AuthenticationServices
    participant API as Vapor API
    participant Apple as Apple ID servers
    participant DB as PostgreSQL
    App->>App: Generate a random nonce, keep it in memory
    App->>ASA: Authorization request with SHA-256 of the nonce
    ASA->>Apple: System sign-in sheet
    Apple-->>App: identityToken and authorizationCode, name and email on first authorization only
    App->>API: POST /v1/auth/apple with identityToken, authorizationCode, nonce, installationId and device info
    API->>Apple: Fetch JWKS, cached 24 h
    Apple-->>API: Signing keys
    API->>API: Verify signature, issuer, audience, expiry, and that the token nonce equals SHA-256 of the sent nonce
    API->>Apple: POST /auth/token with authorizationCode and the client_secret JWT
    Apple-->>API: Apple refresh token
    API->>DB: Upsert the user by apple_subject, store first-login name and email, store the Apple refresh token encrypted
    API->>DB: Insert the session with hashed skat and skrt tokens, replacing the session of this installationId
    API-->>App: 200 with access token, refresh token, expiries and the user
    App->>App: Store one Keychain blob, AfterFirstUnlockThisDeviceOnly, non-synchronizable
```

The server verifies the identity token with JWTKit against the cached JWKS, exchanges the short-lived code and keeps only Apple's refresh token, encrypted, for later revocation (D09, D11). Name and email are stored now or never (D09). One session exists per installation, so a sign-in on the same installation replaces the previous one (D16). `/v1/auth/apple` is limited to 10 per minute per IP and 5 per user; failed verifications log reason codes only; App Attest is not used; tests use an `AppleIdentityProvider` fake (D16). The `client_secret` JWT format is verified at Phase 3.

### 4b. Token refresh with rotation and reuse detection

```mermaid
sequenceDiagram
    participant App as iOS app
    participant API as Vapor API
    participant DB as PostgreSQL
    participant Hub as Realtime hub
    participant Push as PushProvider
    App->>API: POST /v1/auth/refresh with refresh token R1, single-flight, under 5 minutes left
    API->>DB: Load the session by hash of R1, check revoked_at and absolute_expires_at
    API->>DB: Issue A2 and R2, move hash of R1 to previous_refresh_token_hash with previous_valid_until = now + 60 s, extend the sliding window
    API-->>App: 200 with A2 and R2
    Note over App,API: The response is lost, the client retries with R1 inside the 60 s window
    App->>API: POST /v1/auth/refresh with R1
    API->>DB: R1 matches the previous hash within the grace window, issue A3 and R3 superseding A2 and R2
    API-->>App: 200 with A3 and R3
    Note over App,API: Later, a stale copy or an attacker presents R1 after the window
    App->>API: POST /v1/auth/refresh with R1
    API->>DB: No current or grace match, revoke the session with revoke_reason = refresh_reuse, delete device rows, invalidate upload tokens, write a security.session_reuse audit row
    API-->>App: 401 session_revoked
    API->>Hub: Close every socket of this session with 4401
    API->>Push: Signed out for your safety, to the other devices of the user
```

Access tokens live 60 minutes; refresh tokens rotate on every use with a 90-day sliding and 365-day absolute lifetime (D16). The grace window makes a retry after a lost response harmless, while any other presentation of a non-current token revokes the whole session, closes its sockets (D13) and warns the other devices. The client retries once on `token_expired` and on `session_revoked` shows the sign-in screen while keeping the outbox (D08). The security push has no project event behind it, so it cannot ride the worker's event-keyed outbox; the diagram hands it to `PushProvider` from the api, a placement Phase 3 confirms.

### 4c. Task move with optimistic UI, the move contract, commit, event sequence and the 409 path

```mermaid
sequenceDiagram
    participant U as User
    participant App as iOS app
    participant Cache as SwiftData cache and outbox
    participant API as Vapor API
    participant Svc as Service layer
    participant DB as PostgreSQL
    participant Hub as Realtime hub
    participant Peer as Other member device
    U->>App: Drop card T between A and B in column X, or choose Move to Column
    App->>App: afterTaskId = A, beforeTaskId = B, mint operationId, generate the rank with RankKey
    App->>Cache: Set draftColumnId and draftRank on T, enqueue the PendingOperation
    App->>API: POST /v1/tasks/T/move with operationId, destinationColumnId, beforeTaskId, afterTaskId, expectedTaskVersion
    API->>Svc: moveTask as the actor
    Svc->>DB: BEGIN, SELECT project FOR UPDATE, INSERT operation row ON CONFLICT DO NOTHING, SAVEPOINT
    Svc->>Svc: Authorize moveTask, project not archived, task active, column active in the same project, neighbors not the task itself
    Svc->>Svc: Resolve neighbors tolerantly, then apply the version rule to columnId and rank
    alt No task.moved event exists after expectedTaskVersion
        Svc->>DB: Compute the key, renormalize the column if the key exceeds 32 chars, set completedAt per is_done, bump task.version
        Svc->>DB: UPDATE projects SET last_sequence = last_sequence + 1 RETURNING, insert task.moved and the optional column.ranks_normalized
        Svc->>DB: Store the 200 body on the operation row, COMMIT
        Svc->>Hub: Post-commit publish
        API-->>App: 200 with task, renormalizedRanks and sequence
        App->>Cache: Clear the draft overlay, apply the server task, remove the operation
        Hub-->>Peer: event frames in sequence order
    else A task.moved event exists after expectedTaskVersion
        Svc->>DB: ROLLBACK TO SAVEPOINT, store the 409 body on the operation row, COMMIT
        API-->>App: 409 position_conflict with the current task
        App->>Cache: Drop the draft, snap T to the server position
        App->>U: Banner for 4 s, Moved by Sam a moment ago. Your move was not applied.
    end
```

Every drop and menu move computes `(destinationColumnId, afterTaskId, beforeTaskId)` from the cached column order and calls one `MoveCoordinator.move` (D01). `afterTaskId` names the card that will precede the moved card and `beforeTaskId` the card that will follow; both null means append to the bottom (D06). The server tolerates neighbors no longer adjacent, uses the surviving neighbor when one is gone, and answers 409 `neighbors_changed` with `details.currentOrder` when both are gone, which the client retries once with new neighbors and a new `operationId` (D06). A stale `expectedTaskVersion` is tolerated unless a `task.moved` event exists after it; the rejected card snaps back with a banner and an accessibility announcement, never a dialog (D01, D05, D08). WIP never rejects a move; the event records `wipExceeded` (D19).

### 4d. Realtime fan-out over WebSocket, disconnect, and catch-up

```mermaid
sequenceDiagram
    participant A as Device A
    participant API as Vapor API
    participant Hub as Realtime hub
    participant B as Device B
    B->>Hub: WSS upgrade of /v1/realtime with the access token in the Authorization header
    Hub-->>B: 101 switching protocols
    B->>Hub: subscribe projectId P
    Hub->>Hub: ProjectAuthorizer read check for P
    Hub-->>B: subscribed P with sequence 41
    A->>API: POST move, commits event 42
    API->>Hub: Publish 42 after commit
    Hub-->>B: event P 42
    B->>B: 42 equals lastSequence + 1, apply, lastSequence = 42
    Note over Hub,B: Network drops, pings go unanswered, the server closes after 90 s with 4408
    A->>API: Two more edits commit events 43 and 44
    B->>Hub: Reconnect with backoff 1 to 30 s plus jitter, re-subscribe to every cached project
    Hub-->>B: subscribed P with sequence 44
    B->>API: GET v1 projects P events after 42 limit 500
    API-->>B: items 43 and 44, nextCursor null
    B->>B: Apply in order, lastSequence = 44
    Hub-->>B: event P 45 arrives live later
    Note over B,API: If the gap exceeds 2000 events the API answers 410 sequence_too_old and B reloads the project snapshot
```

The upgrade is authenticated by the bearer header and rejected with HTTP 401 before upgrading when invalid; a non-member subscribe answers `not_found` (D13). Events are published only by the post-commit hook, never by the worker, and catch-up is REST-only: after `subscribed`, or whenever a live event is not `lastSequence + 1`, the client pages `GET events?after=` until caught up; a gap above 2,000 events or a `SyncState` older than 24 h triggers a snapshot reload (D08, D13). Duplicates are harmless because application is by sequence. Sockets outlive access tokens; every revocation closes them with 4401 and the hub re-validates sessions every 60 s (D13, D16). While disconnected in the foreground the client polls events every 60 s; it closes the socket 5 s after backgrounding (D13).

### 4e. Offline queue replay with idempotency keys

```mermaid
sequenceDiagram
    participant U as User
    participant App as iOS app
    participant Outbox as outbox.store
    participant API as Vapor API
    participant DB as PostgreSQL
    Note over App: Offline
    U->>App: Create task T1, then edit its notes, then move T2
    App->>Outbox: op1 create T1 with a client-generated id, op2 edit T1 dependsOn op1, op3 move T2, each with its own operationId
    Note over App,API: Reachability returns, the session is refreshed if needed, REST catch-up runs, then replay starts
    App->>API: op1 POST tasks with operationId op1
    API->>DB: Insert the operation row, run the mutation, store 201
    API-->>App: 201 task T1
    App->>API: op2 PATCH T1 with operationId op2 and expectedVersion
    Note over App,API: The response is lost and the app is killed while op2 is inFlight
    App->>API: On the next launch, op2 again with the same operationId
    API->>DB: Row exists and request_hash matches
    API-->>App: The stored 200 replayed with the header Idempotency-Replayed true
    App->>API: op3 move T2 with operationId op3
    API-->>App: 409 position_conflict
    App->>Outbox: op3 marked conflict, later operations on T2 are blocked, other entities continue
    App->>U: T2 snaps to the server position with the banner
```

The outbox replays in strict global FIFO by `createdAt`, one operation in flight, after the session is valid, reachability has returned and REST catch-up has run (D08). Client-generated ids let a create-then-edit chain replay without temporary-id mapping (a collision is 409 `duplicate_id`), and `dependsOn` makes a rejected create fail its dependents visibly (D08, D22). An operation left `inFlight` by a kill is retried with the same `operationId`, exactly the D18 replay case. Membership changes, invitations, project creation and attachment uploads are never queued (D08). The state machine below is the `PendingOperation` lifecycle.

```mermaid
stateDiagram-v2
    [*] --> pending : minted with the edit, draft overlay set
    pending --> inFlight : head of the FIFO, session valid, reachable
    inFlight --> [*] : 2xx, draft overlay cleared
    inFlight --> pending : 5xx, network error or 429, backoff, same operationId
    inFlight --> conflict : 409 version_conflict, position_conflict, neighbors_changed or task_archived
    inFlight --> failed : 400, 403, 404, 410 or 422, dependents fail too
    pending --> failed : older than 30 days, or its dependsOn failed
    conflict --> [*] : Keep Mine or Compare, resubmitted as a new operation
    conflict --> [*] : Use Server, draft discarded
```

### 4f. Invitation create, share, preview, accept

```mermaid
sequenceDiagram
    participant Inv as Inviter app
    participant API as Vapor API
    participant DB as PostgreSQL
    participant Msg as Messages or Mail
    participant Rec as Recipient app
    participant Hub as Realtime hub
    Inv->>API: POST /v1/projects/P/invitations with operationId, role, expiresInDays
    API->>API: Authorize, owner or admin, and the admin role only when the inviter is the owner
    API->>DB: Insert the invitation with the SHA-256 hash of a 256-bit token, write invitation.created
    API-->>Inv: 201 with the invitation, the https url and the one-time token
    Inv->>Msg: ShareLink with the token in the URL fragment
    Msg->>Rec: Recipient taps the link
    Rec->>Rec: Parse the fragment, keep the token in memory only, run Sign in with Apple if signed out
    Rec->>API: POST /v1/invitations/preview with the token
    API-->>Rec: projectName, inviterDisplayName, role, expiresAt
    Rec->>API: POST /v1/invitations/accept with operationId and the token
    API->>DB: Project FOR UPDATE, then invitation FOR UPDATE, check expiry, revoked, used, declined, project state and inviter standing
    API->>DB: Insert the membership, set accepted_at and accepted_by_user_id, write member.joined
    API-->>Rec: 201 with the membership
    API->>Hub: Publish member.joined after commit
    Note over API,DB: The worker later pushes the invitation category to the inviter
```

The token is 32 CSPRNG bytes as base64url, stored only as its SHA-256, single use, and travels in the fragment of `https://<host>/invite#<token>` so no proxy, tunnel or server log sees it; the content-free `/invite` page hands it to the app through the custom scheme or a "Copy code" button, and a "Join with code" field accepts the bare token (D04). Preview, accept and decline require a signed-in user, limited to 10 per minute per user and 30 per IP; creation is 5 per project per hour (D04, D22). Acceptance returns 404 for an unknown token, 410 `invitation_unavailable` with a reason for expired, revoked, used, declined, archived-project and inviter-unavailable cases, and 200 `{alreadyMember: true}` for an existing member while still consuming the token (D04). Nothing is pushed on creation because a link has no known recipient (D07).

### 4g. Attachment upload initiate, background upload, completion and authorized download

```mermaid
sequenceDiagram
    participant App as iOS app
    participant BG as Background URLSession
    participant API as Vapor API
    participant DB as PostgreSQL
    participant Vol as Attachment volume
    App->>App: Stage the picked file under Application Support uploads, strip location EXIF, compute SHA-256, record a StagedUpload
    App->>API: POST /v1/tasks/T/attachments/initiate with operationId, originalName, declaredMime, byteSize, sha256
    API->>DB: Check role, 20 per task, 2 GB per project, free space and allowlist, insert a pending row with the upload token hash
    API-->>App: 201 with attachmentId, uploadPath, uploadToken skut and uploadExpiresAt at plus 24 h
    App->>BG: uploadTask PUT uploadPath fromFile with Authorization Bearer uploadToken
    Note over App,BG: The app may be suspended, the transfer continues
    BG->>API: PUT /v1/attachments/A/content, streaming body
    API->>Vol: Stream to key.part, enforce the declared size, hash while streaming, sniff the first 512 bytes
    API->>API: Compare the hash and the sniffed family with the declared values
    API->>Vol: Atomic rename of key.part to key
    API->>DB: Mark complete, write attachment.added
    API-->>BG: 200 with the AttachmentDTO
    BG-->>App: handleEventsForBackgroundURLSession, StagedUpload cleared
    Note over App,API: Later, any member downloads
    App->>API: GET /v1/attachments/A/content with the access token and an optional Range
    API->>DB: Membership, state complete, deleted_at null
    API->>Vol: Open the file by its validated key only
    API-->>App: 200 with the sniffed Content-Type, attachment disposition, nosniff, CSP sandbox, no-store
```

The PUT is the completion step; there is no `/complete` endpoint (D14). The upload token is a single-purpose secret bound to the attachment, size, hash, user and issuing session, valid 24 hours, hashed at rest and invalidated when that session is revoked, so a background transfer can finish long after the access token expired (D14, D16). The server answers 413 for oversized bodies, 415 for a sniff mismatch, 409 `version_conflict` for a re-PUT with a different hash, 429 above 10 pending uploads per user and 507 when the volume has under 10 % or 10 GB free; a re-PUT with a matching hash returns 200 and emits nothing (D14). Downloads require membership and a complete, undeleted row, never derive a path from user input, and carry headers that keep polyglot files inert (D14).

### 4h. MCP propose, confirm, cancel and audit

```mermaid
sequenceDiagram
    participant C as Claude client
    participant Br as mcp-stdio bridge
    participant MCP as MCP adapter in api
    participant Svc as Service layer
    participant DB as PostgreSQL
    participant Hub as Realtime hub
    C->>Br: tools call propose_move_task
    Br->>MCP: Streamable HTTP to loopback /mcp with the skct connection token
    MCP->>DB: Validate the grant hash, scope tasks write, expiry and rate limit, audit the call
    MCP->>Svc: Read the task and the destination column as the bound user
    MCP->>DB: Insert PendingAIChange with operations, diff, captured_versions, risk_level, token hash, expires_at at plus 10 minutes
    MCP-->>C: diff, affected project and card, risk level, required scope, expiry, confirmation_token
    Note over C: The MCP client asks the human before the next call
    alt Human approves
        C->>Br: confirm_change with change_id and confirmation_token
        Br->>MCP: Forward
        MCP->>DB: pending to confirmed atomically, check token hash, user, client_id and expiry
        MCP->>Svc: Re-run ProjectAuthorizer, compare every captured version
        Svc->>DB: Execute the stored operations with their stored operationIds, via = mcp, allocate the sequence, COMMIT
        Svc->>Hub: Publish after commit
        MCP->>DB: Audit row with change_id and outcome ok
        MCP-->>C: Committed sequence
    else Human declines
        C->>Br: cancel_change with change_id
        Br->>MCP: Mark cancelled, audit row
        MCP-->>C: Cancelled
    end
    Note over MCP,DB: A second confirm answers 409 change_already_applied, a moved version answers 409 stale_proposal, lost membership answers failed with 403
```

Read tools execute immediately under the bound user's permissions; every mutation tool only writes a `PendingAIChange` and returns the exact diff (D15). Confirmation is single use, bound to the user, client, change and expiry, re-authorizes at confirmation time and refuses a diff whose captured versions moved, so what was shown is exactly what is applied; execution reuses the per-operation ids from the proposal, so a double confirm cannot double-apply (D15, D18). Confirmed changes appear in the feed as the actor name followed by "via Claude"; pending proposals are never project events. Scopes default to `projects:read` and `tasks:read`; writes need an explicit grant and `members:manage` is a separate opt-in. In release 1 the human veto rests on the MCP client's per-call approval because the model holds the confirmation token; an in-app approval mode is a release-2 candidate (D15).

### 4i. Account deletion with Apple token revocation and ownership handling

```mermaid
sequenceDiagram
    participant App as iOS app
    participant API as Vapor API
    participant Apple as Apple ID servers
    participant DB as PostgreSQL
    participant Hub as Realtime hub
    participant W as Worker
    App->>App: Consequences sheet, Transfer or Delete chosen for every owned project with other members, destructive confirmation
    App->>API: POST /v1/account/delete-challenge
    API->>DB: Store a single-use nonce that expires in 5 minutes
    API-->>App: nonce and expiresAt
    App->>Apple: Fresh Sign in with Apple with that nonce
    Apple-->>App: identityToken and authorizationCode
    App->>API: POST /v1/account/delete with identityToken, authorizationCode, nonce, transfers, deletions
    API->>API: Verify the token, nonce equals an unconsumed challenge, sub equals the user, iat within 5 minutes, consume the challenge
    API->>DB: Every owned project with other members must appear in transfers or deletions, else 409 owned_projects_require_transfer
    API->>DB: BEGIN, apply transfers through the D03 path, soft-delete chosen and solo-owned projects
    API->>DB: Revoke every session, delete device rows, invalidate upload tokens, revoke Claude grants and pending AI changes, delete pending invitations
    API->>DB: Delete every membership, clear assigneeId with task.updated via system, write member.left
    API->>DB: Anonymize the user row, apple_subject null, email null, display_name Deleted member, ciphertext null, deleted_at now, audit row, COMMIT
    API->>Hub: Close every socket of the user with 4401 and evict subscriptions
    API->>Apple: Exchange the fresh code, then POST /auth/revoke, falling back to the stored token
    alt Revocation fails
        API->>W: Retry record, hourly for 7 days, final outcome logged
    end
    API-->>App: 204
    App->>App: Wipe Keychain, both SwiftData stores, the attachment cache, local notifications and the APNs registration
    Note over W,DB: The worker purges the soft-deleted projects after 30 days, with no restore
```

Deletion requires a fresh Apple sign-in bound to a server-issued single-use challenge, so a stolen access token cannot delete the account; the brief's `DELETE /v1/account` becomes `POST /v1/account/delete` because DELETE requests carry no body (D09, D18). Every owned project is decided in the one request: transfers run through the same ownership-transfer path as the UI, projects without other members are deleted permanently, and projects soft-deleted this way cannot be restored (D03, D09). The user row is anonymized in place and kept forever, so the other household member keeps their history under "Deleted member"; MCP audit rows are kept 1 year (D09, D15). Apple revocation is best effort with worker retries, and `POST /v1/auth/apple/notifications` handles `consent-revoked` and `account-delete` once a public hostname exists (D09, D10). Whether the revoke call runs before or after the 204 is fixed at Phase 3; it never blocks deletion.

### 4j. Member removal while the member has an open socket

```mermaid
sequenceDiagram
    participant Adm as Admin app
    participant API as Vapor API
    participant DB as PostgreSQL
    participant Hub as Realtime hub
    participant Mem as Removed member app
    Mem->>Hub: Subscribed to project P, socket open
    Adm->>API: DELETE /v1/projects/P/members/M
    API->>API: Authorize, an admin may remove an editor or viewer, only the owner may remove an admin, nobody removes the owner
    API->>DB: BEGIN, project FOR UPDATE, delete the membership row
    API->>DB: Clear assigneeId on active tasks of M, one task.updated per task via system
    API->>DB: Write member.removed with userId and fromRole, COMMIT
    API-->>Adm: 204
    API->>Hub: Publish the events to the remaining members
    API->>Hub: evict userId M from project P
    Hub-->>Mem: unsubscribed P with reason membership_revoked
    Hub->>Hub: Drop the subscription within one second, the socket stays open for other projects
    Mem->>Mem: Remove the P subtree from the cache, fail queued operations for P with not_found
    Mem->>API: Any later REST call for P
    API-->>Mem: 404 not_found
    Note over Mem: The session is not revoked and other projects keep working
```

Removal deletes the membership row and leaves events, comments and attachments attributed; the removed user's assignments on active tasks are cleared with system events (D03). From the next request every REST call for that project answers 404 `not_found`, the hub sends `unsubscribed` and drops the subscription within one second, and the client removes the project subtree and fails its queued operations for that project (D03, D08, D12, D13). The session is deliberately not revoked. The worker also sends the removed user a `membership` push, which bypasses project mute (D07). A repeat `DELETE` is a strict no-op (D18). The same eviction path serves self role changes, project archive and project deletion (D13).

## 5. Concurrency and consistency model

### 5.1 Optimistic concurrency rules

1. The server is the authority; the client applies changes optimistically through the draft overlay and reconciles with the response or the event stream (D08, D12).
2. Every project-scoped mutation is one transaction that takes `SELECT ... FOR UPDATE` on the project row first, inserts the operation row, performs the domain writes, allocates the sequence once per event, writes the events, stores the operation outcome and commits; publication happens only after commit (D06, D17, D18). One lock per transaction makes deadlocks impossible; the only transaction touching several projects is account deletion (5.3).
3. PATCH on a task, column, project, comment or member requires `operationId` and `expectedVersion` (400 `invalid_request` if either is missing) and uses `Patch<T>`: an absent field is unchanged, an explicit `null` clears (D05, D22).
4. A stale `expectedVersion` is resolved from the activity log: the `changes` keys of the events for this entity with `entity_version > expectedVersion` are the fields changed since; the outcomes are in 5.2. If those events are missing for any reason, every field counts as changed (fail closed). An `expectedVersion` ahead of the current version is a client that needs a resync, not a conflict (D05).
5. Mergeable fields: task `title`, `notes`, `dueAt`, `assigneeId`; column `title`, `color`, `wipLimit`, `isDone`; project `name`; comment `body`; member `role`. `completedAt`, `columnId`, `rank` and `archivedAt` are never patched: position goes through `/move`, completion is derived from the done column, archive and restore have their own endpoints, and column order goes through `/columns/reorder` with `expectedProjectVersion` (D05, D06, D19).
6. Validation runs on every write, merged or not: `assigneeId` must be a current member (422 `validation_failed`); archived tasks and projects answer 409; unknown or permanently deleted entities answer 404 (D05).
7. A retry with the same `operationId` replays the stored outcome, including a stored 409; a resolved conflict is always a new `operationId` with the new `expectedVersion` (D05, D18).

### 5.2 What merges and what returns 409

| Situation | Outcome |
|---|---|
| `expectedVersion` equals current | Applied; version bumped; one event |
| Stale version, patched fields disjoint from the fields changed since | Merged; 200 with `X-Merged-From-Version` |
| Stale version, overlapping field with the same value | Field dropped, rest merged; 200 with the current record and no event if nothing remains |
| Stale version, overlapping field with a different value | 409 `version_conflict` with `conflictingFields` and `current` |
| Events for the window missing | Fail closed: 409 `version_conflict` |
| `expectedVersion` ahead of current | 400 `invalid_request`, reason `expectedVersionAhead`; client resyncs |
| Move with a stale `expectedTaskVersion` and no later `task.moved` | Applied |
| Move with a later `task.moved` | 409 `position_conflict` with the current task |
| Move whose neighbors are no longer adjacent, or one neighbor gone | Applied after `afterTaskId`, else the surviving neighbor |
| Move with both neighbors gone | 409 `neighbors_changed` with `details.currentOrder`; the client retries once with new neighbors |
| Any write to an archived task | 409 `task_archived`; the client offers Restore |
| Any write to an archived project except unarchive, leave, removal, notification settings | 409 `project_archived` |
| Concurrent edits of one comment body | Always 409 (single field) |
| Concurrent role changes of one member | 409 `version_conflict` on the member row version |
| Create with a client id that already exists | 409 `duplicate_id` |
| Owner tries to leave with other members present | 409 `owner_must_transfer` |
| Attachment re-PUT with a different hash | 409 `version_conflict` |
| MCP confirm after a captured version moved, or a second confirm | 409 `stale_proposal`, 409 `change_already_applied` |
| Retry with the same `operationId` and body | The stored status replayed verbatim, even a 409 |
| Retry with the same `operationId` and a different body | 422 `idempotency_mismatch` |

### 5.3 Transactions for moves and membership changes

A move (D06), under the project lock: authorize `moveTask` and refuse an archived project; require an active task (409 `task_archived`) and an active destination column of the task's own project (404 otherwise); refuse the task as its own neighbor (422); resolve neighbors tolerantly, any neighbor from another project being 404; apply the version rule to `{columnId, rank}`; compute the key, renormalize if needed, set `completedAt` when entering the `is_done` column and clear it when leaving, bump `task.version`; write `task.moved` with `changes {columnId, rank, completedAt?}` and `extra {fromColumnId, toColumnId, wipExceeded, completionChange}`, then the optional `column.ranks_normalized`, then the operation record; respond `{task, renormalizedRanks, sequence}`. Column archive with tasks requires `moveTasksTo` and appends the moved tasks to the target's bottom with `task.moved` events; column reorder reassigns every rank under `expectedProjectVersion` (D19).

Membership changes (D03, D04, D09): invitation acceptance takes the project lock, then `SELECT ... FOR UPDATE` on the invitation row, re-checks expiry, revocation, prior use, project state and the inviter's current standing, inserts the membership and writes `member.joined`. A role change carries the member row's `expectedVersion` and writes `member.role_changed`; the hub evicts the affected user's subscription so the client re-subscribes and drops edit affordances (D13). Removal deletes the membership row, clears `assigneeId` on that project's active tasks with one `task.updated` per task (`via = system`), writes `member.removed`, and after commit publishes and evicts; sessions are not revoked. Ownership transfer makes the target owner and the previous owner admin, increments `project.version` and records `project.ownership_transferred` in one transaction; it has no MCP tool. An owner leaving with other members present receives 409 `owner_must_transfer`; a solo owner's Leave is project deletion. Account deletion is the one transaction spanning several projects: it runs the transfer path per transferred project, soft-deletes the rest, deletes memberships with the same assignee-clearing events, revokes sessions and grants, and anonymizes the user; it takes the project locks in ascending project id order so two concurrent deletions cannot deadlock (verify at Phase 3).

### 5.4 Rank renormalization

Trigger: the generated key is longer than 32 characters. Scope: the destination column only, inside the same transaction. Every active task in the column, in current order with the moving task at its new place, receives keys from `generateNKeysBetween(nil, nil, n)`. Because the partial unique index cannot be deferred, the update is two-pass: pass one prefixes every rank in the column with `-` (outside the alphabet, so no final key collides); pass two assigns the final keys. Renormalization rewrites `rank` only, bumps no versions, and writes one `column.ranks_normalized` event with `extra.ranks = [{taskId, rank}]` after the `task.moved` event, which other clients apply in sequence order without version gating (D06, D08). Phase 2 inserts 1,000 times into one gap to exercise it and verifies the two passes are expressible through Fluent plus SQLKit.

### 5.5 Event ordering guarantees

- Per project, events form a total order equal to commit order, gap-free from 1, unique on `(project_id, sequence)` (D17).
- A client learns the sequence of its own mutation from the move response or the `X-Project-Sequence` header, and events carry `operation_id`, so an echo is recognisable (D18, D22).
- A mutation writing two events (`task.moved` then `column.ranks_normalized`) writes them in that order with consecutive sequences; bulk effects such as assignee clearing on removal are contiguous within one transaction (D06, D17).
- Nothing uncommitted is ever broadcast; delivery is at-least-once across socket plus REST catch-up, and application is idempotent by sequence (D13).
- Catch-up by `after=` returns events in sequence order, up to 500 per page; `before=` pages the feed backwards (D22).
- Worker consumption is exact because per-project cursors advance over a gap-free counter (D17).
- Eviction after removal, role change, archive or deletion reaches the socket within one second; every revocation closes sockets explicitly; the hub re-validates sessions every 60 s as a backstop (D13).

### 5.6 Explicitly not guaranteed

- No ordering across projects and no global sequence (D17).
- No exactly-once socket delivery; duplicates and gaps are expected and handled by sequence (D13).
- No read-your-writes across devices before the event or snapshot arrives; a read outside the lock may trail the latest sequence.
- No automatic text merge; same-field conflicts are surfaced, not resolved; comment edits always conflict (D05, D08).
- No conflict dialog for moves; a rejected move snaps back (D01, D08).
- No cross-device consistency of unsent drafts; no replay of operations older than 30 days (D08).
- No correctness with more than one api instance or with worker-published events; the hub is in-memory and LISTEN/NOTIFY is deliberately not built (D11, D13).
- No point-in-time consistency between the database dump and the attachment snapshot; the restore drill tolerates files without rows (D21).
- No stable `sequence` in a replayed response; it may be stale (D18).
- No WIP enforcement; limits warn only (D19).
- No due reminder on a device that has not synced the task; no server-sent due reminders (D07).
- No server-side defense against prompt injection in task content read by Claude; the confirmation step and the audit log are the mitigation (D15).

## 6. Runtime topology on the Mac mini

### 6.1 Processes, bindings and volumes

| Process | Runs as | Network binding | Storage |
|---|---|---|---|
| `api` (`Run serve`) | Compose service; non-root, read-only root filesystem, health check, restart policy, graceful shutdown | `127.0.0.1:8080` only; the `dev` profile publishes on the LAN interface; `/mcp` loopback-only until Phase 9b | `/data/attachments` bind mount (rw), `/tmp` |
| `worker` (`Run worker`) | Compose service, same image and hardening | None published | `/data/attachments` (rw) for cleanup |
| `postgres` (`postgres:17-alpine`) | Compose service | Internal Compose network only; `127.0.0.1:5432` under the `dev` profile only | Named volume |
| `migrate` (`Run migrate`) | One-shot Compose service run explicitly with `docker compose run --rm migrate`, never in `depends_on` | None | None |
| Tailscale (`tailscaled`, `tailscale serve`, optional `funnel`) | macOS host app | 443 on the tailnet hostname forwarded to `127.0.0.1:8080`; Funnel path-scoped when enabled | None |
| `cloudflared` | Compose service only in the Cloudflare alternative | Outbound-only tunnel to `api` | None |
| Backup (`backup.sh`, restic, `health.sh`, `export-secrets.sh`) | Host launchd jobs, not containers | `pg_dump` through `docker compose exec`; probes `/ready` | `/Users/kanban/SharedKanban/data/backups`, `/Volumes/KanbanBackup/restic`, the iCloud Drive repository |
| `Run mcp-stdio` | Host process launched by Claude Desktop or Claude Code | Client of `127.0.0.1:8080/mcp` | None; only the connection token in its environment |
| `Run mcp-token`, `Run admin` | Operator-invoked | None | None |

The register places Tailscale, the backup jobs and the stdio bridge on the macOS host rather than in Compose, so backups reach the SSD and iCloud Drive natively, no second TLS terminator runs, and the AI-launched process holds no credentials (D11, D15, D21). Writable host data is `/Users/kanban/SharedKanban/data/{attachments,backups}`, kept under `/Users` so Docker Desktop shares it by default (D11). Docker Desktop (OrbStack is the lighter documented alternative) runs under the dedicated auto-login service user `kanban`; api and worker get a best-effort egress allowlist of the Apple hosts only (D10, D11).

### 6.2 Environment and secrets

Configuration comes exclusively from environment variables loaded from `/Users/kanban/SharedKanban/secrets/.env`, outside the repository; `.env.example` is committed with placeholders and the exact variable names are fixed there at Phase 1 (D11). The register names `DATABASE_URL` for tests and `APPLE_TOKEN_KEY` (32 bytes) for encrypting Apple refresh tokens (D09, D20). The secrets directory also holds the Sign in with Apple `.p8` key, the APNs `.p8` key, the App Store Connect API key used by Cowork for TestFlight uploads, and `restic-password` (0600); it is never inside a restic repository, and `export-secrets.sh` writes it as an encrypted disk image to the SSD whose passphrase lives in the owner's password manager (D20, D21). The Claude connection token sits in the Claude client's configuration on the Mac mini, readable by that macOS user, which the threat model accepts for the trusted host (D15). On the client, the base URL and environment name come from a git-ignored `Config/Environment.xcconfig` with a committed example plus a Debug-only in-app picker, so the production hostname never enters source control (D10). Nothing secret is ever written to `CLAUDE.md`.

### 6.3 The launchd escape hatch

If Docker Desktop proves unstable or too heavy, `deploy/launchd/build.sh` builds the arm64 release binary with `swift build -c release` and `deploy/launchd/` plists run `Run serve` and `Run worker` as LaunchDaemons against Homebrew `postgresql@17` with identical environment variables; this path needs no GUI login and is exercised once in Phase 10 so it is known to work before it is needed (D11). Because worker, migrations and the MCP endpoint are subcommands of one binary and PostgreSQL is the only queue, the escape hatch changes nothing in the application.

### 6.4 Ingress profiles in order

| Profile | When | Requires | Changes |
|---|---|---|---|
| 1. LAN development | Phases 1 to 9 on the home network | Debug builds only; `http://<mac-mini>.local:8080`; `NSAllowsLocalNetworking = true` and `NSLocalNetworkUsageDescription` in the Debug Info.plist; the Compose `dev` profile publishing the api on the LAN | Release builds carry no ATS exception |
| 2. Tailscale household beta, the release-1 production profile | From Phase 10, by default | Tailscale on the Mac mini and both devices with on-demand VPN; HTTPS certificates enabled once in the admin console; key expiry disabled for the Mac mini node; `tailscale serve` terminating TLS with a Let's Encrypt certificate for `<mini>.<tailnet>.ts.net` and proxying to `127.0.0.1:8080`; a macOS Tailscale build whose CLI supports `serve`, `cert` and `funnel` (verify at Phase 10) | Nothing listens on the public internet; no domain, certificate work or ATS exception; invitations use the https link, the `/invite` page and the code field; the Associated Domains developer-mode experiment at Phase 7 may enable universal links on the household devices; first-run UX says the VPN must be on |
| 3. Public HTTPS, only when a concrete need appears | Remote MCP from claude.ai or a phone, universal links, a device that cannot run Tailscale | Default: Tailscale Funnel on the same hostname, one admin-console toggle, path-scoped to `/v1/*`, `/invite`, `/.well-known/*`, `/health` and, after Phase 9b, `/mcp`. Alternative: Cloudflare Tunnel with an owned domain and `cloudflared` in Compose, accepting edge decryption and trusting forwarded client-IP headers only for requests from the tunnel | Enables the AASA fetch by Apple's CDN, registration of `/v1/auth/apple/notifications`, and the Phase 9b OAuth server for `/mcp`; the Funnel endpoint counts as public internet in the threat model |

In every profile a tunnel is transport, not authorization: every REST request, WebSocket upgrade and MCP call carries a SharedKanban bearer token and passes project authorization; no IP- or tailnet-based allowlist is relied on; PostgreSQL has no published port; rate limits key by user first and by client IP only where the address reaches the origin; HSTS and `X-Content-Type-Options: nosniff` are on every response (D10, D22). Funnel WebSocket support, idle timeouts and bandwidth limits are verified at Phase 10, and the Phase 10 runbook holds the "switch to public" steps.

## 7. Build, test and delivery topology

### 7.1 Where work runs

| Work | Cloud Linux session | Mac mini Cowork session | GitHub Actions CI (ubuntu) |
|---|---|---|---|
| Writing code, docs, DTOs, migrations, deploy files | All of it | Fixes found while building | None |
| `swift build` and `swift test` for `shared/` and `server/` | Yes when a Swift 6 toolchain is present; integration tests read `DATABASE_URL` and `XCTSkip` without a database | Yes, natively, including the launchd binary | On every push, with a PostgreSQL service |
| Compiling and testing the iOS app | Never | `xcodebuild build` and `test` on simulators at the D02 widths and window modes; `xcrun simctl`; `xcodebuild -runFirstLaunch` and `-downloadPlatform iOS`; screenshots at Phases 4 and 6 | No macOS runners in release 1 |
| Real-device work | Never | `xcrun devicectl` cable installs; Sign in with Apple, APNs sandbox and background-upload checklists | None |
| Signing and distribution | Never | Automatic signing with the login keychain kept unlocked; App Store Connect API uploads to TestFlight; reports when the Apple ID session needs the human | None |
| Lint and format | Where the toolchain exists | SwiftLint and format | Same script |
| Operations | Writes `deploy/` | `docker compose`, Tailscale and `tailscale serve`, restic, launchd, restore drills, log inspection | None |

One CI script runs in all three places with "requires macOS" markers, so the Mac mini is the macOS CI (D20). Swift Testing is used for new unit tests and XCTest for UI tests; warnings are errors for `server/` and SharedDTOs from Phase 1 and for the iOS target once Phase 4 stabilizes (D20). The owner's part is the one-time nine-item checklist in D20 (Apple ID and Xcode sign-in, Developer Program enrollment, portal capabilities and the three keys, device Developer Mode and trust, the Tailscale account and console toggles, macOS admin prompts, the UPS and SSD, the password-manager entries, the optional B2 account); enrollment can take days and gates Phases 3 and 8, so it starts during Phase 1.

### 7.2 The cloud container today

As observed on 2026-10-01, the cloud Linux container has no Swift toolchain (`swift` and `swiftc` are not installed) and no Docker daemon (the Docker CLI is present but `/var/run/docker.sock` does not exist). PostgreSQL 16 client and server packages are installed but no server is running, and 16 is not the pinned major. Nothing in `server/` or `shared/` can be compiled or tested here until Phase 1 installs a Swift 6 toolchain in the container and starts a local PostgreSQL (apt, or Docker if a daemon becomes available); if that proves impossible, those tests skip here and run on the Mac mini and in CI, the fallback D20 and the register's Phase 1 assumptions allow. This session can still write every file, validate Mermaid and JSON fixtures, and run shell and Node tooling.

### 7.3 Versions

Fixed by the register: iOS and iPadOS 18.0 minimum (raised to the previous major at Phase 4 only if both devices run the current major, never below 18); Swift 6 language mode with `-strict-concurrency=complete`, with the server falling back to Swift 5 mode under an ADR if the chosen Vapor is not clean; Vapor 4 (Vapor 5 evaluated once if a stable release exists, never a pre-release); Fluent 4 with `fluent-postgres-driver`; JWTKit 5; APNSwift; swift-log; swift-crypto; PostgreSQL 17 by image tag, one major for the life of release 1 (D11, D20). The exact Xcode release, Swift minor, Vapor minor, Tailscale variant and the household devices' OS versions are "verify at Phase 1"; Phase 1 records the resolved versions in this section and in `Package.resolved` (D20).

## 8. Cross-reference table

| Register ID | Topic | Sections of this document | ADR |
|---|---|---|---|
| D01 | iPhone board and cross-column dragging | 1, 4c, 5.6 | none |
| D02 | iPad split view and compact windows | 1, 7.1 | none |
| D03 | Roles and permissions | 2.10, 2.11, 2.14, 3.1, 3.6, 4f, 4i, 4j, 5.3 | none |
| D04 | Invitation links | 2.8, 2.9, 3.1, 3.6, 4f, 5.3 | none |
| D05 | Optimistic concurrency | 3.3, 4c, 5.1, 5.2 | none |
| D06 | Card ranking and move contract | 2.1, 3.4, 3.5, 4c, 5.3, 5.4 | none |
| D07 | Local versus remote notifications | 2.6, 2.12, 2.18, 3.1, 4j | none |
| D08 | Offline cache and conflicts | 2.3, 2.4, 4c, 4d, 4e, 5.6 | none |
| D09 | Account deletion and token revocation | 2.2, 2.17, 3.1, 3.6, 4a, 4i, 5.3 | none |
| D10 | Local, Apple and ingress | 1, 2.8, 2.9, 6.1, 6.4 | none |
| D11 | Backend stack and runtime | 2.9, 2.12, 2.14, 6.1, 6.2, 6.3, 7.3 | ADR-0001 Vapor, Fluent and PostgreSQL |
| D12 | SwiftData as cache only | 2.3, 5.1 | ADR-0002 SwiftData as cache |
| D13 | WebSockets for realtime | 2.5, 2.11, 4b, 4d, 4j, 5.5, 5.6 | ADR-0003 WebSockets realtime |
| D14 | Local attachment storage | 2.7, 2.15, 3.1, 3.6, 4g | ADR-0004 Local attachment storage |
| D15 | MCP adapter semantics | 1, 2.13, 3.1, 4h, 5.6, 6.2 | ADR-0005 MCP proposal and confirmation |
| D16 | Session and token lifecycle | 2.2, 2.9, 3.1, 3.6, 4a, 4b | none |
| D17 | Activity log and sequence | 3.1, 3.3, 3.5, 5.5 | none |
| D18 | Idempotency | 3.7, 4c, 4e, 5.1, 5.2 | none |
| D19 | Workflow invariants and templates | 2.10, 2.14, 3.1, 3.6, 5.3 | none |
| D20 | Platform targets and where work runs | 1, 7.1, 7.2, 7.3 | none |
| D21 | Backup and restore | 2.16, 6.1, 6.2 | none |
| D22 | API conventions | 2.1, 2.4, 2.9, 3.1, 5.2, 5.5 | none |

The five ADRs live beside docs/decisions/decision-register.md and expand D11 to D15; this document does not restate their rationale.
