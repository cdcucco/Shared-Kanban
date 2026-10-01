# ADR-0004: Local attachment storage on the Mac mini

Status: Accepted (2026-10-01)

## Context

Tasks may carry photos and documents. The brief requires attachment files on a mounted Mac mini volume, never in PostgreSQL, with metadata in the database, random keys instead of user filenames, type and size validation, and no serving of arbitrary paths. Uploads must survive app suspension through a background `URLSession`, whose file-based transfers can finish hours after the 60-minute access token (D16) has expired. The same origin serves the invite page and the AASA (D04), so a hostile file must never be served in a way a browser would execute. The disk belongs to an unattended home machine. This ADR expands register D14; the register is law where the two differ.

## Decision

**Storage.** Files live on the host at `/Users/kanban/SharedKanban/data/attachments`, bind-mounted at `/data/attachments` (mode 0700, owned by the api user) into the `api` and `worker` containers (D11). The storage key is 32 CSPRNG bytes as 64 lowercase hex characters, stored as `<k[0..2]>/<k[2..4]>/<k>` with no extension. Every key is validated against `^[0-9a-f]{64}$` before any path is built; no path is derived from user input; Vapor's `FileMiddleware` is not used; the MCP adapter has no filesystem access (D15).

**Metadata.** One table, `attachments { id, task_id, project_id (denormalized for authorization), uploader_id, storage_key, original_name, declared_mime, sniffed_mime, byte_size, sha256, state ∈ pending|complete, upload_token_hash, upload_expires_at, session_id, created_at, completed_at, deleted_at }`. `original_name` is sanitized (control characters and path separators stripped, at most 255 characters) and is display-only.

**Limits and allowlist.**

| Limit | Value | Error |
|---|---|---|
| File size | 25 MB | 413 `payload_too_large` |
| Attachments per task | 20 | 422 `validation_failed` |
| Per project | 2 GB, `SUM(byte_size)` of complete rows | 422 `validation_failed` |
| Global cap | configurable, default 50 GB | 422 `validation_failed` |
| Pending uploads per user | 10 | 429 `rate_limited`, `details.reason = "pendingUploads"` |
| Free space on the volume | below 10 % or below 10 GB | 507 `insufficient_storage` |
| Initiate calls | 30 per hour per user (D22) | 429 `rate_limited` |

Release-1 allowlist: `image/jpeg`, `image/png`, `image/heic`, `image/heif`, `image/gif`, `image/webp`, `application/pdf`, `text/plain`, `text/markdown`, `video/mp4`, `video/quicktime`. The server sniffs the first 512 bytes (`ftyp` brands for HEIC/HEIF/MP4/MOV, `%PDF`, valid UTF-8 without NUL for text); the sniffed family must match the declared type, else 415 `unsupported_media_type`. SVG, HTML, zip, OOXML, CSV and executables are excluded; OOXML and CSV are release-2 candidates.

**Upload flow.** Two requests; the second is the completion step.

1. `POST /v1/tasks/{id}/attachments/initiate { operationId, originalName, declaredMime, byteSize, sha256 }` runs the role check (editor and above, D03), the count, quota, free-space and allowlist checks, creates a `pending` row and returns 201 `{ attachmentId, uploadPath: "/v1/attachments/{id}/content", uploadToken, uploadExpiresAt (+24 h) }`. The upload token (`skut_` + 32 CSPRNG bytes base64url) is a single-purpose secret hashed at rest and bound to the attachment id, declared size, SHA-256, user and issuing session (D16); logout, session revocation and account deletion (D09) invalidate it. Initiate is online-only (D08); a file picked offline is staged as "waiting to upload" and initiated when connectivity returns.
2. The client stages the file under `Application Support/uploads/<attachmentId>` (copied from `fileImporter` inside `startAccessingSecurityScopedResource`, or as a `FileRepresentation` from `PhotosPicker`, never fully in memory), strips location EXIF from photos by default, hashes the staged bytes, records a `StagedUpload` in `outbox.store` (D12), and runs `uploadTask(with:fromFile:)` on `URLSession(configuration: .background("app.sharedkanban.uploads"))` with `Authorization: Bearer <uploadToken>`; the app delegate adaptor implements `handleEventsForBackgroundURLSession`.
3. `PUT /v1/attachments/{id}/content`: the server streams to `<k>.part` on the same volume, enforces the declared size while streaming, computes SHA-256, sniffs the first bytes, compares the hash with the initiate value, renames atomically to `<k>`, marks the row `complete`, emits `attachment.added` (D17; the only point where the project row lock is taken) and returns 200 with the DTO. The brief's `POST /v1/attachments/{id}/complete` is dropped (documented deviation, D22). The PUT is idempotent by state: a re-PUT on a complete row with a matching SHA-256 returns 200 and emits nothing; a mismatch returns 409 `version_conflict`. It carries no `operationId` (D18 does not apply to binary bodies). A token that expires before the PUT lands means a fresh initiate; the abandoned `pending` row is reaped below.

The task detail shows progress, Retry and Cancel per row; thumbnails are built only on the client with `QLThumbnailGenerator`, previews use `.quickLookPreview`, and bytes live in the 200 MB LRU attachment cache, never SwiftData (D08).

**Download.** `GET /v1/attachments/{id}/content` with the normal access token requires project membership (every role, D03), `state = complete` and `deleted_at IS NULL`; it responds with `Content-Type: <sniffed_mime>`, `Content-Disposition: attachment; filename="<ascii-safe>"; filename*=UTF-8''<percent-encoded>`, `X-Content-Type-Options: nosniff`, `Content-Security-Policy: default-src 'none'; sandbox`, `Cache-Control: private, no-store`, and supports `Range`. Ids outside the caller's projects return 404 `not_found` (D22).

**Deletion and cleanup.** `DELETE /v1/attachments/{id}` (uploader, or owner/admin for anyone's, D03; strict no-op when already deleted, D18) sets `deleted_at` and emits `attachment.deleted`; the worker unlinks the file 24 hours later. The worker (D11) also deletes `pending` rows older than 48 hours with their `.part` files hourly, and weekly quarantines files with no row (deleted after a further 7 days) and flags rows whose file is missing in the owner's System Status screen. Permanent task deletion (D19) and project purge (D03) use the same path.

**Backups.** restic snapshots the attachment directory alongside the six-hourly `pg_dump` (D21); `.part` files are excluded; the two are not point-in-time consistent, so the restore drill tolerates files without rows and re-hashes every complete file against `attachments.sha256`.

## Alternatives considered

- Blobs in PostgreSQL: bloats dumps and backups.
- Presigned object storage or MinIO: a cloud dependency or another service.
- Access token on background uploads: expires mid-transfer or must become long-lived.
- Multipart uploads: background sessions need a file body; one PUT is simplest.
- Server-generated thumbnails: image-parsing attack surface on the server.
- A separate completion endpoint: one more state and a worker publication path.
- A 7-day file grace period after delete: restic already holds prior versions.

## Consequences

### Positive

- Path traversal is impossible by construction: keys are random, validated and never user-derived.
- Allowlist, sniffing, `attachment` disposition, `nosniff` and the CSP keep HTML and SVG from executing on the origin that serves the invite page.
- Completion folded into the PUT means one state transition, no worker involvement and a single event.
- Quotas and the free-space floor keep the single disk from filling silently.

### Negative

- The client must hash a 25 MB file before initiate and keep a staged copy until the PUT succeeds.
- `tailscale serve` and Funnel body limits must exceed 25 MB (verify at Phase 10).
- Deleted files linger 24 hours on disk and longer in restic snapshots; the privacy inventory says so.

### Neutral

- The worker needs write access to the volume; the background session identifier is stable across launches; the pending-upload cap and free-space errors need understandable UI copy.

## Verification

- Phase 8 tests: traversal attempts in keys and names; sniff-versus-declared mismatch; oversized body; hash mismatch; upload-token reuse, expiry and revocation; unauthorized and deleted-row download; background resume after suspension; pending-upload cap; HEIC `ftyp` fixtures from real iPhone photos.
- Phase 8 exit gate: on a real device, attach a photo, suspend, resume, and see the upload complete (manual scenario 4); every error in the table renders a plain-language message.
- Phase 10: the restore drill re-hashes every complete attachment and fails on any mismatch; ingress body limits confirmed above 25 MB.
- Phase 11: attachment handling and path traversal are an explicit audit item.

## Related

- Register: D14 (expanded here), D03, D08, D09, D11, D12, D16, D17, D18, D19, D21, D22.
- ADRs: ADR-0001 (runtime and bind mounts), ADR-0002 (`StagedUpload` in `outbox.store`).
- Docs: docs/architecture.md, docs/threat-model.md, docs/decisions/decision-register.md.
