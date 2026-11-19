# CollabHub — Canvas Service

> Collaborative design documents: Yjs CRDT relay, presence, and snapshot persistence.
> Read [Platform Conventions](./00-platform-conventions.md) first.

**Status:** Draft · **Runtime:** Python 3.13+ / FastAPI + python-socketio
**Owns:** documents, document membership/permissions, snapshot persistence
**Depends on:** PostgreSQL (own DB), Redis Real-time (R2), Redis Cache (R1), Redis Streams (R3)

---

## 1. Purpose & Responsibilities

The Figma-like half of CollabHub. Hosts real-time, conflict-free collaborative editing of
design documents.

**Critical design point:** the **CRDT merge logic lives in the Yjs client library**.
This service is a **relay + persistence layer**, not a merge engine. It moves opaque Yjs
binary updates between clients and periodically snapshots document state to Postgres. It does
not interpret the document's internal structure.

**Owns:** document metadata, membership/permissions, persisted Yjs state (snapshots + update
log), awareness/presence routing.
**Produces:** export jobs to `jobs:export` (R3); `document.index` jobs to `jobs:index` if canvas
search is enabled (D12).
**Does NOT own:** the document's semantic structure (layers/components live inside the Yjs
doc, opaque to the backend), thumbnails/exports (Asset + Worker), font/image blobs (Asset).

---

## 2. Runtime & Dependencies
- FastAPI REST (metadata + lifecycle) + a Socket.IO server ("canvas hub") on the R2 Redis
  manager, mounted in the same ASGI app.
- SQLAlchemy 2.0 async + `asyncpg`; Yjs state stored as `bytea` (see §4).
- `redis.asyncio`: R1 (active-doc cache, sequence counters, snapshot locks), R3 (export jobs);
  R2 is used by the Socket.IO manager.
- **`pycrdt`** — Python bindings to the Rust `y-crdt` implementation (binary-compatible with
  Yjs). The relay path treats updates as opaque `bytes` and never needs it; it is used only to
  **merge updates into compacted snapshots** and to compute state vectors / diffs for
  `SyncStep2`. Merging is CPU work: run it via `asyncio.to_thread` (pycrdt releases the GIL in
  native code) or in a small process pool so it never stalls the event loop.

---

## 3. Public Interface

### 3.1 REST (`/api/v1`)

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/documents` | member | List documents the user can access (cursor paginated). |
| POST | `/documents` | member | Create document (empty Yjs doc). |
| GET | `/documents/{id}` | doc viewer | Metadata (name, owner, updatedAt, members). |
| PATCH | `/documents/{id}` | doc editor | Rename / move / change visibility. |
| DELETE | `/documents/{id}` | doc owner | Soft-delete. |
| GET | `/documents/{id}/snapshot` | doc viewer | Latest Yjs state (`application/octet-stream`) for initial load over REST. |
| GET | `/documents/{id}/members` | doc viewer | Membership + roles. |
| POST | `/documents/{id}/members` | doc owner/editor | Share with a user/role. |
| DELETE | `/documents/{id}/members/{userId}` | doc owner | Unshare. |
| POST | `/documents/{id}/export` | doc viewer | Enqueue `canvas.export` on `jobs:export` → `202` with `{ jobId }`. |

### 3.2 Socket.IO hub — path `/hubs/canvas`

The sync protocol mirrors the Yjs sync protocol carried over Socket.IO events. All update
payloads are raw `bytes` (Yjs binary, sent as Socket.IO binary attachments), opaque to the relay.

**Client → Server events**

| Event | Payload | Effect |
|-------|---------|--------|
| `JoinDocument` | `{ documentId }` | Authorize, join rooms `doc:{id}` + `presence:{id}`. Server emits `SyncStep1` (its state vector). Viewers join read-only. |
| `SyncStep1` | `{ documentId, stateVector: bytes }` | Client's state vector; server replies `SyncStep2` with the diff it holds. |
| `SyncUpdate` | `{ documentId, update: bytes }` | Editors only. Broadcast to others in `doc:{id}` and persist (§5). Ack after the update is relayed and appended to the log. |
| `AwarenessUpdate` | `{ documentId, awareness: bytes }` | Cursor/selection/presence; fan out to `presence:{id}`; **not persisted**. |
| `LeaveDocument` | `{ documentId }` | Leave rooms; emit awareness removal. |

**Server → Client events**

| Event | Payload | Notes |
|-------|---------|-------|
| `SyncStep1` | `{ documentId, stateVector: bytes }` | Sent on join so the client can compute its diff. |
| `SyncStep2` | `{ documentId, update: bytes }` | The server's persisted state (snapshot + log) as a Yjs update, diffed against the client's vector. |
| `Update` | `{ documentId, update: bytes }` | A peer's `SyncUpdate`, relayed. |
| `AwarenessUpdate` | `{ documentId, awareness: bytes }` | A peer's awareness, relayed. |
| `PeerLeft` | `{ documentId, userId }` | For clearing remote cursors (also emitted on disconnect). |
| `TokenExpired` | `{}` | Client must reconnect with a fresh token. |

**Join sequence:** client `JoinDocument` → server `SyncStep1` → client `SyncStep1` / server
`SyncStep2` exchange (and the client sends its own diff as a `SyncUpdate`) → steady state where
`SyncUpdate`/`Update` flow both ways. Relayed events go through the R2 manager so any pod can
serve any client.

---

## 4. Data Model (PostgreSQL)

```sql
CREATE TABLE documents (
    id            uuid PRIMARY KEY,
    workspace_id  uuid NOT NULL,
    name          text NOT NULL,
    owner_id      uuid NOT NULL,
    visibility    text NOT NULL DEFAULT 'private',  -- private | workspace | link
    created_at    timestamptz NOT NULL DEFAULT now(),
    updated_at    timestamptz NOT NULL DEFAULT now(),
    deleted_at    timestamptz NULL,
    version       integer NOT NULL DEFAULT 0
);

CREATE TABLE document_members (
    document_id   uuid NOT NULL REFERENCES documents(id),
    user_id       uuid NOT NULL,
    role          text NOT NULL DEFAULT 'editor',   -- owner | editor | viewer
    PRIMARY KEY (document_id, user_id)
);

-- Persisted Yjs state. Two viable storage strategies (pick one — see Open Decisions):
--  (a) latest compacted snapshot only
--  (b) snapshot + an append log of updates since the snapshot (recommended)
CREATE TABLE document_state (
    document_id   uuid PRIMARY KEY REFERENCES documents(id),
    yjs_state     bytea NOT NULL,            -- compacted Yjs document (binary, Y.encodeStateAsUpdate)
    state_vector  bytea NOT NULL,            -- for fast SyncStep1
    snapshot_at   timestamptz NOT NULL DEFAULT now(),
    update_seq    bigint NOT NULL DEFAULT 0  -- highest log seq folded into this snapshot
);

-- Append log (strategy b): durable between snapshots, replayed on recovery.
CREATE TABLE document_updates (
    document_id   uuid NOT NULL REFERENCES documents(id),
    seq           bigint NOT NULL,           -- per-document, allocated via R1 INCR canvas:seq:{id}
    payload       bytea NOT NULL,            -- one Yjs update
    created_at    timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (document_id, seq)
);
```

> The architecture mentions storing canvas docs as **JSONB**. Yjs state is binary, so `bytea`
> is the natural fit; a JSONB column can hold a *derived, human-readable* representation for
> search indexing if needed. Confirm in Open Decisions.

### 4.1 Redis usage
- **R2:** Socket.IO Redis manager (`channel="sio:canvas"`) for relaying `Update`/`AwarenessUpdate`;
  awareness is ephemeral and lives only in R2/in-memory, never Postgres.
- **R1:** cache of the active document's latest state vector + recent updates to serve fast
  `SyncStep1`/`SyncStep2` while a doc is "hot"; per-document sequence counter
  `canvas:seq:{id}`; snapshot lock `canvas:snaplock:{id}`.
- **R3:** `jobs:export` (`canvas.export`), and `jobs:index` (`document.index`) if D12 is adopted.

---

## 5. Internal Design — Edit & Snapshot
Mirrors the architecture's sequence diagram:
1. Client A emits `SyncUpdate(documentId, update)`.
2. Server emits `Update` to room `doc:{id}` (skip sender) via the R2 manager; other clients
   receive it.
3. The **receiving pod** persists the update: allocate `seq` (`INCR canvas:seq:{id}`) and insert
   into `document_updates` (strategy b). Clients of one document may be connected to different
   pods, so each pod persists only what it receives — no single pod sees every update in memory.
4. **Snapshot trigger** — whichever comes first: every N seconds (default 10) of activity, or
   every M logged updates (default 200), or on last-editor-leaves. The pod that wins
   `SET canvas:snaplock:{id} NX PX <ttl>` loads `yjs_state` + log rows with `seq > update_seq`,
   merges them with `pycrdt` (off the event loop), writes the new `yjs_state` / `state_vector` /
   `update_seq` and deletes the folded log rows — in one transaction.
5. On cold start / first joiner, load `yjs_state` + replay `document_updates` to seed the relay.

With strategy (a) there is no shared log, so cross-pod snapshotting relies on a client pushing
its full state on a debounce (see D11).

---

## 6. Configuration
Common vars (Conventions §8). Plus `CANVAS__SNAPSHOT_INTERVAL_SECONDS` (10),
`CANVAS__SNAPSHOT_EVERY_UPDATES` (200), `CANVAS__MAX_DOC_BYTES` (e.g. 25 MB),
`CANVAS__AWARENESS_TIMEOUT_SECONDS` (30).

## 7. Cross-Cutting
Auth on the hub connection per Conventions §6 (handshake `auth` payload). Errors, health,
observability per Conventions. Metrics: `canvas_updates_relayed_total`,
`canvas_snapshots_total`, `canvas_active_docs` gauge, `canvas_awareness_msgs_total`.

## 8. Non-Functional & Limits
- Update relay p99 < 500 ms.
- Max document size cap (`CANVAS__MAX_DOC_BYTES`) enforced on snapshot; Socket.IO
  `max_http_buffer_size` set to bound a single update/initial sync.
- Awareness is best-effort and lossy by design; updates are not.
- Relay is not delayed by persistence; the append log (strategy b) prevents data loss between
  snapshots. The `SyncUpdate` ack is sent only after the log insert succeeds, so a client can
  retry un-acked updates (Yjs updates are idempotent).
- Ingress session affinity on `/hubs/canvas` if long-polling fallback is enabled (Conventions §6).

## 9. Open Decisions
- **Storage strategy:** (a) snapshot-only with periodic client-driven full state, vs.
  (b) snapshot + append log replayed on recovery (stronger durability). Recommend (b).
- **Server-side Yjs:** compute snapshots server-side with `pycrdt`, vs. trust a debounced
  full-state push from a designated client. Affects correctness on crash; `pycrdt` makes the
  server-side option straightforward in Python.
- **JSONB vs bytea** for `yjs_state` (architecture says JSONB; binary suggests bytea + optional
  derived JSONB for search).
- Whether canvas content is indexed in Elasticsearch (the architecture lists "canvas content"
  under search) — requires a derived text projection (producible here with `pycrdt` at
  snapshot time) indexed by the Worker.
