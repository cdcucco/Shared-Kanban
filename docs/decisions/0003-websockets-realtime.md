# ADR-0003: WebSockets for realtime project events

Status: Accepted (2026-10-01)

## Context

Two household members edit one board at once and expect the other device to follow within about a second; a device that was offline must come back without losing or duplicating a change. The server is authoritative, every project write is serialized on the project row and allocates a gap-free per-project sequence inside the mutation transaction (D06, D17), and the offline client already catches up by that sequence (D08). Realtime therefore only has to deliver committed events quickly and must never show a device anything the database may still roll back. Constraints: one `api` process on the Mac mini behind `tailscale serve` or Funnel (D10, D11), 60-minute access tokens (D16), and membership revocable while a board is open (D03). This ADR expands register D13 and the sequence parts of D17.

## Decision

**Transport.** Vapor's native WebSocket support serves `WSS /v1/realtime` on the same origin as REST (D11, D13); the client uses `URLSessionWebSocketTask`. The access token travels in the `Authorization: Bearer` header of the upgrade request, never in the query string; an invalid or expired token gets HTTP 401 before the upgrade (D16). A socket may outlive its access token: the hub keys every connection by session id, every revocation path (logout, per-session revoke, revoke-all, refresh reuse, account deletion, absolute expiry) closes that session's sockets with code 4401 `session_revoked`, and the hub re-validates every open socket's session against the database every 60 s so `admin` revocations land within a minute. Limits: 3 sockets per session, 20 subscriptions per socket, 10 inbound messages per second, 64 KB per frame; excess closes with 4429 (D13, D22).

**Protocol.** JSON, one object per text frame, discriminated by `type`, versioned with `/v1` and fixture-tested in `docs/api.md` (D22):

| Direction | `type` | Fields | Rule |
|---|---|---|---|
| client to server | `subscribe` | `projectId` | membership checked through `ProjectAuthorizer` (D03) |
| client to server | `unsubscribe` | `projectId` | |
| client to server | `ping` | | every 30 s in the foreground |
| server to client | `subscribed` | `projectId`, `sequence` | `projects.last_sequence` now |
| server to client | `event` | `projectId`, `sequence`, `event: ActivityEventDTO` | one per committed event |
| server to client | `unsubscribed` | `projectId`, `reason` | `reason` is `membership_revoked` or `project_unavailable` |
| server to client | `error` | `code`, `projectId?` | a non-member subscribe answers `not_found`, never `forbidden` |
| server to client | `pong` | | |

**Sequence and publication (D17).** Every project-scoped mutation runs in one transaction that takes `SELECT ... FOR UPDATE` on the project row, performs its writes, then executes `UPDATE projects SET last_sequence = last_sequence + 1 WHERE id = $1 RETURNING last_sequence` once per event. The lock is held until commit, so sequence order equals commit order per project and a rollback leaves no gap; `(project_id, sequence)` is unique. The single `Mutation.commit` helper writes the entity change, the events and the idempotency record (D18) and registers publication through a post-commit hook with the in-process `RealtimeHub` actor (keyed `projectId` to connections). Nothing is broadcast before commit. The worker never writes project events (D07, D11, D14), so one process and an in-memory hub suffice: no Redis, no LISTEN/NOTIFY.

**Catch-up is REST-only.** After `subscribed`, if `sequence > SyncState.lastSequence` the client calls `GET /v1/projects/{id}/events?after=<lastSequence>&limit=500` until caught up. A live `event` is applied only when `sequence == lastSequence + 1`; anything else triggers the same REST catch-up, so duplicate or out-of-order frames are harmless. Events are kept forever (D17); the 2,000-event cap is an efficiency limit, not a retention window: beyond it the server answers 410 `sequence_too_old` and the client reloads the snapshot `GET /v1/projects/{id}`, which replaces the project subtree and carries the current `sequence` (D08, D12). Events carry `operationId` so a client recognises its own echoes (D18). Events and snapshots are applied without version gating because sequence order is total per project (D08).

**Revocation.** Membership removal, a role change to self, project archive and project deletion call `hub.evict(userId:projectId:)` after commit; the socket receives `unsubscribed` and the subscription is dropped within one second (D03, D13). The client then re-fetches `GET /v1/projects/{id}`: a 404 removes the project subtree (D12); a 200 reloads it and re-subscribes only if the project is active and the user still a member. REST access ends at the same commit (D03), so the socket is never a side door.

**Lifecycle.** The server closes after 90 s of silence with 4408; close code 1001 from a server restart counts as a network drop. The client reconnects with backoff 1, 2, 4, 8, 16, 30 s plus 20 % jitter, resets on success, reconnects immediately on `NWPathMonitor` changes (including the Tailscale interface appearing) and on `scenePhase == .active`, re-subscribes to every cached project and follows the D08 reconnect order (refresh session, catch up, replay outbox, subscribe). The socket closes 5 s after backgrounding and is never opened there. While disconnected in the foreground the client polls `GET events` every 60 s; a "Reconnecting…" capsule appears after 5 s.

## Alternatives considered

- Token in the query string: logged by proxies and tunnels (D13).
- First-frame authentication handshake: needed only if headers were unavailable (verify at Phase 6).
- Event replay over the socket: duplicates the REST catch-up path.
- Server-Sent Events: no client-to-server subscribe; the brief fixes WebSockets.
- Redis pub/sub or LISTEN/NOTIFY: needed only with several api processes or a publishing worker (D11).
- Closing sockets at access-token expiry: hourly reconnects on every device (D16).

## Consequences

### Positive

- Header authentication keeps tokens out of URLs and proxy logs; subscriptions reuse the REST membership check.
- A gap-free, commit-ordered sequence makes catch-up one query and "no lost or duplicated event" testable.
- Post-commit publication guarantees nothing uncommitted reaches a device.
- Every revocation path closes sockets explicitly; an open board cannot outlive access.

### Negative

- The in-process hub is correct only while exactly one api instance runs; scaling out needs the pub/sub layer deliberately not built now.
- The client needs a small, tested state machine for duplicates, gaps and reconnects (D08).
- Write throughput is serialized per project because sequence and project lock are one mechanism (D06, D17).

### Neutral

- The service layer exposes an after-commit hook; the hub tracks sockets by session id.
- The ping interval is tunable against `tailscale serve` and Funnel idle timeouts.
- Protocol changes are versioned with `/v1` and need fixtures like REST (D22).

## Verification

- Phase 2: sequence tests prove gap-free, commit-ordered numbering and one event per entity version (D05, D17); a mutation without an event fails the suite.
- Phase 6 server tests: subscribe without membership answers `not_found`; duplicate delivery; gap detection triggers catch-up; revoke mid-connection and logout close with 4401; archive and delete evict with the right `reason`; reconnect storm respects backoff; 4429 on the inbound limit; catch-up by events equals a fresh snapshot at the same sequence.
- Phase 6 device checks: `URLSessionWebSocketTask` header support, ping behaviour and background suspension on the minimum iOS (verify at Phase 6); a two-device script moves different cards concurrently and reconnects after airplane mode.
- Phase 7 manual scenario: remove a member whose board is open; REST returns 404 and the subscription drops within one second.
- Phase 10: long-lived sockets through `tailscale serve` and Funnel, idle timeouts, the 1001 restart path (verify at Phase 10).
- Phase 11 audit: WebSocket authorization and reconnect behaviour.

## Related

- Register: D13, D17, and D03, D05, D06, D08, D10, D11, D12, D16, D18, D22 where cited.
- ADRs: ADR-0001 (`docs/decisions/0001-vapor-fluent-postgresql.md`), ADR-0002 (`docs/decisions/0002-swiftdata-as-cache.md`).
- Docs: `docs/architecture.md`, `docs/api.md`, `docs/decisions/decision-register.md`.
