# ADR-0001: Swift, Vapor, Fluent and PostgreSQL on a Mac mini

Status: Accepted (2026-10-01)

## Context

The brief fixes the backend as Swift and Vapor with Fluent over PostgreSQL, hosted on the owner's Mac mini with Docker Compose and a `launchd` escape hatch. Register decision D11 turns that constraint into exact choices; this ADR expands D11 so that the cloud Linux session and the Mac mini Cowork session build the same thing without guessing.

Three facts shape it. The client is native Swift, so one language on both sides lets `SharedDTOs` carry wire types, the rank generator, templates and error codes to both ends (D20, D22). The server is a household machine nobody babysits, so every extra service can break after a macOS update. And it serves two users, so the design optimizes for correctness and operability, not throughput.

## Decision

**Toolchain.** Swift 6 language mode with `-strict-concurrency=complete`, on the latest stable Swift 6.x toolchain at Phase 1, the same version on Linux and on the Mac mini (verify the exact minor at Phase 1). If the chosen Vapor release is not clean under Swift 6 mode, the server package pins Swift 5 language mode and a new ADR records it (D11).

**Libraries.** Vapor 4.x latest; if Vapor 5 has a stable release at Phase 1 it is evaluated once and recorded in an ADR, and no pre-release is ever used. Fluent 4 with `fluent-postgres-driver` on PostgresNIO; JWTKit 5 for Apple identity-token verification; APNSwift for token-based APNs, with a thin AsyncHTTPClient HTTP/2 client as fallback; swift-log with a JSON handler that redacts tokens and paths; swift-crypto. No other server dependency without an ADR (D11).

**Version pinning.** `Package.swift` declares `from:` minimums; the committed `Package.resolved` pins exact versions, recorded in `docs/architecture.md` at Phase 1. PostgreSQL is pinned to one major by image tag, `postgres:17-alpine`; verify at Phase 1 whether a newer major is the better pin for the life of release 1, then stay on that major (D11).

**PostgreSQL is the only queue.** No Redis, no Vapor Queues, no LISTEN/NOTIFY in release 1. The worker polls `projects.last_sequence` against per-project cursors every 2 s (D17), sends from the notification outbox with `FOR UPDATE SKIP LOCKED` (D07), and runs hourly and daily maintenance. The worker never publishes realtime events; the in-process hub in `api` does (D13).

**One binary, subcommands.** The executable `Run` exposes:

| Subcommand | Responsibility |
|---|---|
| `serve` | REST API, WebSocket hub, loopback MCP endpoint (D13, D15) |
| `worker` | APNs outbox, orphan and attachment cleanup, project purge, Apple revoke retries, session and operation purges, backup status |
| `migrate` | Explicit deploy step; never auto-run by `serve` |
| `mcp-stdio` | Credential-free stdio bridge to the loopback MCP endpoint (D15) |
| `mcp-token` | Create, list and revoke Claude connection tokens (D15) |
| `admin` | Owner-only maintenance such as `sessions revoke-all` and `status` |

`GET /health` reports process up; `GET /ready` reports database reachable, migrations current, last backup age (D11, D21).

**Compose.** Services `api` (publishes `127.0.0.1:8080`; the `dev` profile publishes on the LAN, D10), `worker`, `postgres` (internal network only, named volume; `127.0.0.1:5432` published only under `dev`) and `migrate` (one-shot, run with `docker compose run --rm migrate`, never listed in `depends_on`). Tailscale, backups (D21) and the stdio bridge run on the macOS host. Configuration comes exclusively from environment variables loaded from `/Users/kanban/SharedKanban/secrets/.env`, outside the repository, with `.env.example` committed. Writable host data lives under `/Users/kanban/SharedKanban/data/{attachments,backups}` and is bind-mounted into `api` and `worker`. Images are multi-stage builds from the official `swift:6.x` builder to a slim runtime: non-root, read-only root filesystem, writable mounts only for attachments and `/tmp`, health checks, restart policies, graceful shutdown (D11).

**Runtime on macOS.** Docker Desktop (OrbStack documented as the lighter alternative) under the dedicated auto-login service user `kanban` (D21 covers the FileVault consequence). Escape hatch: `deploy/launchd/` plists plus `deploy/launchd/build.sh` build the arm64 release binary with `swift build -c release` and run `serve` and `worker` as LaunchDaemons against Homebrew `postgresql@17` with identical environment variables. This path needs no GUI login and is exercised once in Phase 10 so it works before it is needed (D11).

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| Separate worker package | Duplicate models and migrations; one binary keeps API and worker code identical and makes the launchd fallback trivial (D11) |
| Redis or Vapor Queues | Another service to operate; PostgreSQL tables already give exactly-once delivery with `SKIP LOCKED` (D07, D11) |
| Hummingbird or a non-Swift backend | The brief chose Vapor; another language would lose the shared `SharedDTOs` package (D20) |
| Auto-migrate on `serve` | Dangerous with restores and rollbacks; an unattended restart must never mutate the schema (D11, D21) |
| `migrate` as a Compose service in `depends_on` | Auto-migration in practice; it is a one-shot run instead (D11) |
| Caddy or a backup container | No TLS terminator is needed behind `tailscale serve` (D10); host launchd backups reach the SSD and iCloud Drive natively (D21) |
| Native PostgreSQL by default | Compose keeps dev, test and restore drills reproducible; native PostgreSQL remains the escape-hatch pairing (D11) |

## Consequences

### Positive

- One language and one DTO package on both sides; strict concurrency forces `Sendable` DTOs and surfaces data races at compile time.
- Two fewer services and one container image for API and worker.
- Explicit migrations and pinned versions: no unattended change alters the schema or dependencies.
- The launchd path is proven in Phase 10, not merely documented.

### Negative

- Docker Desktop on a headless, auto-login Mac mini is an operational soft spot (GUI session, VM memory, bind-mount file sharing, macOS updates); production may move to the launchd path.
- Docker Desktop's first launch needs GUI acceptance by the human (D20).
- Write throughput is serialized per project by the project-row lock (D06, D17); fine for a household, documented.
- Vapor 5 timing, APNSwift maintenance and Swift 6 cleanliness of the chosen Vapor release are unknown until Phase 1.

### Neutral

- Bind-mounted paths stay under `/Users` to avoid Docker file-sharing configuration.
- The server package depends on `SharedDTOs` for the rank generator and templates (D06, D19).
- The in-process realtime hub is correct only while exactly one `api` instance runs (D13).

## Verification

- Phase 1 exit gate: the Linux container runs `swift build` and `swift test` for `server/` and `shared/` against a PostgreSQL reached through `DATABASE_URL`, skipping with `XCTSkip` when absent (D20); `/health` and `/ready` answer; GitHub Actions on ubuntu runs the same tests; resolved versions are recorded in `docs/architecture.md` and `Package.resolved`; the Swift 6 cleanliness and Vapor 5 questions are settled.
- Phase 2: Fluent plus raw SQLKit can express the savepoint-based idempotency record (D18) and the two-pass rank renormalization (D06) inside one transaction; `COLLATE "C"` is byte-wise on the pinned PostgreSQL major.
- Phase 8: outbox delivery under `FOR UPDATE SKIP LOCKED` produces no duplicate pushes across a worker restart (D07).
- Phase 10 exit gate: the Compose stack validates from a clean environment with non-root, read-only containers; `migrate` runs as the explicit deploy step; the launchd path is built and run once against Homebrew `postgresql@17`; a restore drill into the disposable `sharedkanban-drill` stack passes `/ready`, fake-login, project and attachment integrity checks (D21).
- Phase 11: Mac mini reboot behavior, dependency review and the auto-restart chain (`pmset autorestart`, auto-login, `health.sh`) are audited (D21).

## Related

- Register: D11 (primary); D06, D07, D10, D13, D14, D15, D17, D18, D19, D20, D21, D22.
- ADRs: ADR-0002 (`0002-swiftdata-as-cache.md`), ADR-0003 (`0003-websockets-realtime.md`), ADR-0004 (`0004-local-attachment-storage.md`), ADR-0005 (`0005-mcp-proposal-confirmation.md`).
- Docs: `docs/architecture.md`, `docs/decisions/decision-register.md`.
