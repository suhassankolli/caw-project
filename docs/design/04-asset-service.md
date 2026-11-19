# CollabHub — Asset Service

> File uploads, downloads, and metadata via presigned object-storage URLs.
> Read [Platform Conventions](./00-platform-conventions.md) first.

**Status:** Draft · **Runtime:** Python 3.13+ / FastAPI (Uvicorn)
**Owns:** asset metadata, presigned URL issuance, bucket layout
**Depends on:** PostgreSQL (own DB), MinIO (S3-compatible), Redis Streams (R3)

---

## 1. Purpose & Responsibilities

Handles all binary assets — images, fonts, exports, avatars, attachments. The defining design
choice: **large blobs never transit the service**. Clients upload and download **directly to
MinIO using presigned URLs**; the service only issues URLs and tracks metadata.

**Owns:** asset metadata (owner, type, size, status), presigned URL generation, bucket/key
layout, post-upload confirmation, enqueuing processing jobs.
**Produces:** thumbnail/optimisation jobs to `jobs:thumbnail` and file-search jobs
(`asset.index`) to `jobs:index` (R3), consumed by the Worker.
**Does NOT own:** actual image processing (Worker does it), referencing logic (Messaging/Canvas
store asset IDs and decide what an asset belongs to).

---

## 2. Runtime & Dependencies
- FastAPI REST only (no Socket.IO).
- S3-compatible async client (`aiobotocore` / `aioboto3` pointed at MinIO) **behind an
  `ObjectStore` protocol** (`typing.Protocol` in `collabhub-shared`) so MinIO ↔ Azure Blob is a
  config/implementation swap, not a code change (Conventions §8). Presigning is a local signing
  operation (no network call); `HEAD`/`DELETE` object calls are async I/O.
- SQLAlchemy 2.0 async + `asyncpg`; Alembic migrations.
- `redis.asyncio` for R3.

---

## 3. Public Interface (REST `/api/v1`)

The upload is a three-step handshake so the blob goes client → MinIO directly.

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| POST | `/assets/upload-url` | member | Reserve an asset; return presigned upload URL + asset id. |
| POST | `/assets/{id}/confirm` | owner | Mark upload complete; validate object exists; enqueue processing. |
| GET | `/assets/{id}` | viewer | Metadata (type, size, status, variants). |
| GET | `/assets/{id}/download-url` | viewer | Presigned **GET** URL (time-limited). |
| GET | `/assets/{id}/thumbnail-url?size=` | viewer | Presigned GET for a generated variant. |
| DELETE | `/assets/{id}` | owner | Soft-delete metadata; schedule blob deletion via retention job. |
| POST | `/internal/assets/{id}/variants` | service token | **Internal.** Worker reports generated variants (D14). Not exposed through the public ingress. |

"Viewer" for an asset is decided by workspace membership (`workspace_id` vs. the token's
`wsp`); finer-grained rules (e.g. attachment in a private channel) are an open decision.

`POST /assets/upload-url`
```json
// request
{ "fileName": "logo.png", "contentType": "image/png", "sizeBytes": 482113, "purpose": "attachment" }
// response
{
  "assetId": "uuid",
  "uploadUrl": "https://minio/collabhub-assets/...&X-Amz-Signature=...",
  "method": "PUT",
  "headers": { "Content-Type": "image/png" },
  "expiresAt": "2026-06-03T10:20:00Z"
}
```

`POST /internal/assets/{id}/variants`
```json
{ "variants": [ { "kind": "thumb-128", "objectKey": "...", "sizeBytes": 5120 } ] }
```
Idempotent: upserts on `(asset_id, kind)`.

Flow (matches the architecture sequence): client requests URL → uploads bytes straight to
MinIO → calls `confirm` → service verifies the object, writes metadata, enqueues a thumbnail
job. Until confirmed, the asset is `status = 'pending'` and not downloadable.

---

## 4. Data Model (PostgreSQL)

```sql
CREATE TABLE assets (
    id            uuid PRIMARY KEY,
    workspace_id  uuid NOT NULL,
    owner_id      uuid NOT NULL,
    file_name     text NOT NULL,
    content_type  text NOT NULL,
    size_bytes    bigint NOT NULL,
    purpose       text NOT NULL,            -- avatar | attachment | font | export | canvas-image
    bucket        text NOT NULL,
    object_key    text NOT NULL,            -- e.g. {workspace}/{purpose}/{assetId}/{fileName}
    status        text NOT NULL DEFAULT 'pending',  -- pending | ready | failed | deleted
    checksum      text NULL,                -- ETag / sha256 after confirm
    created_at    timestamptz NOT NULL DEFAULT now(),
    confirmed_at  timestamptz NULL,
    deleted_at    timestamptz NULL,
    UNIQUE (bucket, object_key)
);
CREATE INDEX ix_assets_pending ON assets (created_at) WHERE status = 'pending';  -- orphan sweep

-- Generated variants (thumbnails, optimised renditions) produced by the Worker.
CREATE TABLE asset_variants (
    id            uuid PRIMARY KEY,
    asset_id      uuid NOT NULL REFERENCES assets(id),
    kind          text NOT NULL,            -- thumb-128 | thumb-512 | webp | ...
    object_key    text NOT NULL,
    size_bytes    bigint NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now(),
    UNIQUE (asset_id, kind)
);
```

### 4.1 MinIO layout
- Bucket `collabhub-assets` (or per-purpose buckets). Object key
  `{workspaceId}/{purpose}/{assetId}/{fileName}` (file name sanitised). Variants under
  `{...}/{assetId}/variants/{kind}`.
- Presigned PUT/GET expiry short (e.g. 15 min). A presigned **PUT** signs `Content-Type` but
  **cannot enforce a maximum size**; size is enforced on confirm (HEAD the object, compare with
  `sizeBytes` and `ASSET__MAX_UPLOAD_BYTES`, mark `failed` and delete if over). If a hard upload
  cap is required, switch to a presigned **POST** policy with `content-length-range`.

### 4.2 Redis usage
- **R3:** `jobs:thumbnail` — `thumbnail.generate` `{ assetId, objectKey, variants: ["thumb-128","thumb-512"] }`
  enqueued on confirm (image content types only). Worker stores variants back in MinIO and
  reports them via `POST /internal/assets/{id}/variants`.
- **R3:** `jobs:index` — `asset.index` `{ assetId }` enqueued on confirm and on delete, for file search.

---

## 5. Internal Design — Upload & Process
1. `POST /assets/upload-url`: validate purpose/content type/declared size, insert `assets` row
   (`pending`), generate presigned PUT, return.
2. Client `PUT`s bytes directly to MinIO.
3. `POST /assets/{id}/confirm`: HEAD the object to confirm existence/size/ETag, set
   `status='ready'`, `confirmed_at`, `checksum`. Confirm is idempotent (re-confirming a `ready`
   asset returns it unchanged).
4. `XADD jobs:thumbnail` (images) and `XADD jobs:index`.
5. Worker consumes, generates variants (Pillow), stores them in MinIO, reports
   `asset_variants` via the internal endpoint.

Orphan cleanup: `pending` assets older than a TTL (no confirm) are swept by a retention job
(Worker) which asks this service to delete both the row and any stray object (see Worker §5.2).

---

## 6. Configuration
Common vars (Conventions §8). Plus:

| Var | Notes |
|-----|-------|
| `OBJECT_STORE__ENDPOINT` | MinIO endpoint (or Azure S3-compatible endpoint). |
| `OBJECT_STORE__PUBLIC_ENDPOINT` | Host used in presigned URLs if clients reach MinIO via a different hostname than the service. |
| `OBJECT_STORE__ACCESS_KEY` / `OBJECT_STORE__SECRET_KEY` | Secret. |
| `OBJECT_STORE__BUCKET` | Default bucket. |
| `OBJECT_STORE__PRESIGN_TTL_SECONDS` | Default 900. |
| `ASSET__MAX_UPLOAD_BYTES` | e.g. 50 MB (per purpose overrides). |
| `ASSET__ALLOWED_CONTENT_TYPES` | Allow-list per purpose. |

## 7. Cross-Cutting
Auth, errors, health, observability per Conventions. Metrics: `assets_uploaded_total`,
`asset_bytes_total`, `presign_issued_total{kind}`.

## 8. Non-Functional & Limits
- Service never streams blob bodies (memory-flat regardless of file size).
- Max upload size per purpose (enforced on confirm, §4.1); content-type allow-list; virus-scan
  hook optional (Open Decision).
- Download URLs are short-lived and per-request; no public buckets.

## 9. Open Decisions
- **Worker → variants callback:** Worker writes `asset_variants` directly to this service's DB
  (breaks the no-shared-DB rule) vs. calls the internal `POST /internal/assets/{id}/variants`
  endpoint (preferred, documented above) vs. emits an event the Asset service consumes.
- Whether avatars/exports use the same bucket or dedicated buckets.
- Virus/malware scanning of uploads before marking `ready`.
- Direct Azure Blob SDK vs. S3-compat layer for the Azure phase (Conventions favours S3-compat).
- Per-resource download authorization (e.g. attachments in private channels) beyond workspace
  membership.
