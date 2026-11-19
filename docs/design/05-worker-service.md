# CollabHub — Worker Service

> Headless background processor: indexing, thumbnails, notifications, exports, retention.
> Read [Platform Conventions](./00-platform-conventions.md) first.

**Status:** Draft · **Runtime:** Python 3.13+ asyncio worker (headless, no HTTP except health)
**Owns:** all async job processing + the Elasticsearch index lifecycle
**Depends on:** Redis Streams (R3), Elasticsearch, MinIO, owning services' internal APIs (§5.2)

---

## 1. Purpose & Responsibilities

A headless Python worker that consumes jobs from **Redis Streams (R3)** and does the heavy,
non-interactive work. Scales horizontally via **KEDA** based on stream backlog.

**Owns:**
- Consuming all `jobs:*` streams (Conventions §7).
- The **Elasticsearch index lifecycle**: mappings, aliases, reindexing, and writes. ES is
  written only by the Worker; services read from ES directly (or via a search gateway).
- Image thumbnailing/optimisation (Pillow).
- Notification dispatch.
- Export generation (canvas → PNG/SVG/PDF).
- Data retention / cleanup sweeps (orchestration — the owning service performs its own deletes).

**Does NOT:** serve user requests, or read/write any service's primary database. It gets data
from the job payload or the owning service's API, and writes results back via the owning
service's internal endpoint (§5.2).

---

## 2. Runtime & Dependencies
- Plain `asyncio` application (`python -m collabhub_worker`): one consumer task per configured
  stream, a handler registry keyed by job `type`, and a `asyncio.TaskGroup` supervising them.
  (A framework like Celery/arq/Dramatiq is not used because the transport is the platform's
  own Redis Streams contract, Conventions §7.)
- `redis.asyncio` (R3 consumer groups: `XREADGROUP`, `XAUTOCLAIM`, `XPENDING`, `XACK`).
- `elasticsearch[async]` (`AsyncElasticsearch`, `async_bulk` helper) for ES.
- **Pillow** for image work, run in a `concurrent.futures.ProcessPoolExecutor`
  (via `loop.run_in_executor`) so CPU-bound resizing does not block the event loop or contend
  on the GIL.
- `ObjectStore` protocol (shared with Asset via `collabhub-shared`; `aiobotocore` against MinIO).
- `httpx.AsyncClient` with a cached service token (client-credentials grant, Conventions §5.1)
  for internal calls to owning services.
- Exposes only `/health/live` and `/health/ready` (Conventions §10) from a minimal Starlette
  app served by Uvicorn in the same event loop; no business HTTP.

---

## 3. Interface — Jobs Consumed

All jobs use the Conventions §7 envelope. One consumer group `worker` per stream; handlers keyed
by `type`.

| Stream | `type` | Payload | Producer | Handler action |
|--------|--------|---------|----------|----------------|
| `jobs:index` | `message.upsert` / `message.delete` | `{ messageId, channelId, workspaceId, authorId, body, createdAt }` (delete: `{ messageId }`) | Messaging | Upsert/delete in the `messages` ES index. Payload is self-contained so the Worker never reads Messaging's DB. |
| `jobs:index` | `document.index` | `{ documentId, workspaceId, name, textProjection, updatedAt }` | Canvas (if D12) | Index canvas text projection. |
| `jobs:index` | `asset.index` / `asset.delete` | `{ assetId, workspaceId, fileName, contentType, ownerId, createdAt }` | Asset | Index file metadata for file search. |
| `jobs:thumbnail` | `thumbnail.generate` | `{ assetId, objectKey, variants[] }` | Asset | Fetch from MinIO, generate variants (Pillow), store back, report to Asset `POST /internal/assets/{id}/variants`. |
| `jobs:notify` | `notify.dispatch` | `{ userId, kind, data }` | Messaging (TBD, D25), Worker (export done) | Deliver notification (in-app/push/email per D18). |
| `jobs:export` | `canvas.export` | `{ documentId, format, requestedBy }` | Canvas | Render document to PNG/SVG/PDF, store in MinIO, notify requester. |
| `jobs:retention` | `retention.sweep` | `{ scope }` | K8s `CronJob` (scheduled `XADD`) | Ask the owning service to hard-delete soft-deleted rows / objects past policy; clean orphan `pending` assets. |

The Worker emits no events of its own except results delivered back through owning services
(e.g. Asset `POST /internal/assets/{id}/variants`) or notifications.

> **Payload contract change:** `message.*` and `asset.*` payloads now carry the fields the index
> needs (previously `{ messageId, op }`), because reading another service's database is
> forbidden (Conventions §2). Producers must include them; ES writes are idempotent by document ID.

---

## 4. Elasticsearch Index Design

Worker owns mappings + aliases. Read-side queries (from Messaging/SPA) hit the aliases.

| Alias | Doc shape (key fields) | Source |
|-------|------------------------|--------|
| `messages` | `messageId, channelId, workspaceId, authorId, body, createdAt` | `jobs:index` |
| `files` | `assetId, workspaceId, fileName, contentType, ownerId, createdAt` | `jobs:index` |
| `canvas` | `documentId, workspaceId, name, textProjection, updatedAt` | `jobs:index` (if enabled) |

- Use index-per-alias with date or version suffix to allow zero-downtime reindex
  (`messages-v1` ← alias `messages`).
- Analyzer: standard + edge-ngram field for autocomplete on names/file names.
- Out-of-order delivery: use ES external versioning (`version_type=external`, version =
  message `version` / `updatedAt` epoch) so a late, older upsert cannot overwrite a newer one.
- Reindex is a maintenance job (manual trigger or a `jobs:index` `reindex` type). Because the
  Worker does not read service databases, a full reindex requires the owning service to
  re-emit index jobs (an internal "replay" endpoint per service).

---

## 5. Internal Design

### 5.1 Consumer loop (per stream)
1. `XREADGROUP GROUP worker {consumer} COUNT n BLOCK m STREAMS jobs:x >`
   (`consumer` = pod name, from the downward API).
2. For each entry: parse envelope (Pydantic), restore the trace context from `traceparent`,
   dispatch by `type`, run handler **idempotently** (keyed on `jobId`/natural key).
3. On success `XACK`. On handler exception, do **not** ack; log and record the error.
4. Periodically `XAUTOCLAIM` stale pending entries (crashed or failed consumers) past the
   visibility timeout and re-run them. Before re-running, read the entry's delivery count via
   `XPENDING` (extended form); if it has reached `maxAttempts` (5), `XADD` it to `jobs:x:dead`
   with the last error and `XACK` the original. (Stream entries are immutable — the envelope
   carries no attempt counter.)
5. On `SIGTERM`: stop issuing new `XREADGROUP`s, wait for in-flight handlers up to the grace
   period, leave the rest unacked for reclaim.

### 5.2 Write-back rule
Worker must not read or write another service's primary tables. For results that must
update a service's DB (e.g. asset variants, retention purges), call that service's internal
endpoint (`POST /internal/assets/{id}/variants`, `POST /internal/retention/sweep`) authenticated
with a service token, or emit a result the owning service consumes. ES and MinIO are shared
infrastructure the Worker is authorized to write. (See Asset Open Decisions.)

### 5.3 Scaling (KEDA)
`ScaledObject` with the Redis Streams scaler using **`lagCount`** (entries not yet delivered to
the group; Redis 7+) per stream. Do **not** scale on `pendingEntriesCount` alone: with zero
replicas nothing reads the stream, so the pending count stays 0 and the deployment would never
scale up from zero. Scale 0→N on lag, scale to zero when idle (except a floor of 1 for
latency-sensitive `jobs:notify`).

---

## 6. Configuration
Common vars (Conventions §8; the Worker has no `POSTGRES__DSN`). Plus:

| Var | Notes |
|-----|-------|
| `ELASTICSEARCH__URL` | ES endpoint. |
| `WORKER__STREAMS` | Which streams this deployment consumes (allows specialised worker pools). |
| `WORKER__MAX_ATTEMPTS` | Default 5. |
| `WORKER__VISIBILITY_TIMEOUT_SECONDS` | Reclaim threshold. |
| `WORKER__BATCH_SIZE` | `COUNT` per read. |
| `WORKER__PROCESS_POOL_SIZE` | Processes for CPU-bound handlers (thumbnail/export). |
| `WORKER__CLIENT_ID` / `WORKER__CLIENT_SECRET` | Service credentials for the client-credentials grant. |
| `OBJECT_STORE__*` | As in the Asset service. |
| `RETENTION__MESSAGE_DAYS` / `RETENTION__ASSET_PENDING_HOURS` / etc. | Retention policy knobs. |
| `NOTIFY__*` | Provider config for push/email. |

## 7. Cross-Cutting
No inbound auth (no business HTTP); uses a **service token** for outbound internal calls.
Trace continuity: read `traceparent` from the job envelope and continue the span (Conventions §9).
Metrics: `jobs_processed_total{type,result}`, `jobs_dead_total{stream}`, handler duration
histograms, ES bulk latency.

## 8. Non-Functional & Limits
- Every handler idempotent and safe to retry.
- No user-facing latency contract except `jobs:notify` (target < 5 s p95).
- Back-pressure handled by stream depth + KEDA, never by dropping jobs.
- Poison messages land in `*:dead` for inspection, never block the stream.
- Python's GIL means CPU-bound handlers must run in the process pool; the event loop only
  orchestrates I/O.

## 9. Open Decisions
- **Specialised worker pools** (separate deployments per stream for independent scaling) vs.
  one deployment consuming all streams. Recommend splitting CPU-heavy (thumbnail/export) from
  IO-heavy (index/notify).
- Notification channels in scope for v1 (in-app only vs. push + email), and which service
  produces `notify.dispatch` for chat events.
- Whether canvas search (`canvas` index) ships in v1 — depends on Canvas producing a text
  projection.
- Result write-back mechanism (internal endpoint vs. event) — align with Asset doc.
- Retention: Worker-orchestrated via internal endpoints (assumed) vs. each service running its
  own scheduled purge.
- Export rendering engine for canvas (server-side `pycrdt` to read the doc + a renderer such as
  `skia-python`/CairoSVG, or a headless browser running the SPA renderer via Playwright) —
  non-trivial; may defer.
