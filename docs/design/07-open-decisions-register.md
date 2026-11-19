# CollabHub — Open Decisions Register

> Consolidated list of every decision left open across the design docs, with the default
> currently baked into each doc and a recommendation where one exists. Use this as a living
> register — update **Status** as decisions are made and reflect them back into the source doc.

**Status legend:** 🔴 Open · 🟡 Leaning (default in docs) · 🟢 Decided
**Scope:** *Cross-cutting* = affects more than one component; resolve these first.

---

## Resolve First (cross-cutting)

These span service boundaries; deciding them keeps the individual docs consistent with each
other. Details in the tables below (IDs in brackets).

- **Auth revocation model** [D1] — whether every service checks the Redis denylist per request.
  Touches Conventions + all services.
- **Multi-workspace tenancy** [D2] — single vs. multiple active workspaces; shapes the `wsp`
  JWT claim. Touches Conventions + Auth + every service's authorization.
- **Real-time transport** [D27] — Socket.IO vs. plain WebSockets now that SignalR is out.
  Touches Conventions + Messaging + Canvas + Frontend.
- **Canvas state storage** [D9/D10] — JSONB (per original architecture) vs. bytea, plus the
  durability strategy. Touches Canvas + Worker (canvas search).
- **Worker result write-back** [D14] — how the Worker reports results into a service's DB
  without sharing schemas. Touches Asset + Worker.
- **Canvas search in Elasticsearch** [D12] — requires Canvas to emit a text projection the
  Worker indexes. Touches Canvas + Worker.
- **Retention orchestration** [D26] — the Worker may not touch service databases, so purges
  need an owning-service mechanism. Touches Worker + every service.

---

## Platform / Auth

| ID | Decision | Default in docs | Recommendation | Scope |
|----|----------|-----------------|----------------|-------|
| D0 | Backend language & framework | 🟢 Python 3.13+ · FastAPI · python-socketio · SQLAlchemy 2 async (replaces .NET 10 / ASP.NET Core / SignalR / EF Core) | — | Cross-cutting |
| D1 | Do all services consult the Redis denylist on every request, or rely only on short token lifetime? | 🟡 Check-with-fail-open (Conv §5.2) | Confirm; fail-open is a security trade-off worth an explicit sign-off | Cross-cutting |
| D2 | Single active workspace per token, or multiple with a switch flow? | 🟡 Single (`wsp` claim) | Decide early — drives token shape and every authz check | Cross-cutting (Conv, Auth) |
| D3 | Primary-key type | 🟡 UUID v7 | Keep unless there's a reason for bigint/ULID | Cross-cutting |
| D4 | Schema-per-service vs database-per-service in production | 🟡 Logically separate DBs | On-prem may co-locate; doesn't change app code | Cross-cutting |
| D5 | Auth acts as full OIDC OP for first-party clients, or only federates to an upstream IdP? | 🔴 Open | — | Auth |
| D6 | SAML federation in v1 or deferred? | 🔴 Open | Defer unless an enterprise customer needs it day one (Python SAML libs need native `xmlsec`) | Auth |
| D7 | Refresh tokens stored hashed in Postgres vs. in Redis (R1) | 🟡 Postgres (hashed) | Postgres for durability | Auth |
| D27 | Real-time transport: Socket.IO vs. plain WebSockets + custom Redis fan-out | 🟡 Socket.IO (`python-socketio` + `socket.io-client`) | Socket.IO — rooms, acks, reconnection, binary and Redis manager out of the box; needs ingress affinity only if polling fallback is kept | Cross-cutting (Conv, Messaging, Canvas, Frontend) |
| D28 | Job enqueue durability: fire-and-forget `XADD` vs. transactional outbox | 🟡 Fire-and-forget (Conv §7) | Outbox for jobs whose loss matters (index, notify); fire-and-forget acceptable for thumbnails | Cross-cutting |
| D29 | Service-to-service auth | 🟡 `client_credentials` grant from Auth, `roles: ["service"]` (Conv §5.1) | Confirm; alternative is mTLS via a service mesh | Cross-cutting (Auth, Worker, Asset) |

## Messaging

| ID | Decision | Default in docs | Recommendation | Scope |
|----|----------|-----------------|----------------|-------|
| D8a | Threading depth | 🟡 Single-level | Confirm against UX needs | Messaging |
| D8b | DM modelling | 🟡 `kind='dm'` channels | — | Messaging |
| D8c | `/search/messages` lives in Messaging vs. a dedicated search gateway | 🟡 Thin proxy in Messaging | Revisit if search grows | Messaging |
| D8d | Edit/delete windows and tombstone retention | 🔴 Open | Align with Worker retention (D16) | Messaging + Worker |
| D25 | Who produces `notify.dispatch` for chat events (mentions, DMs) | 🔴 Open | Messaging, on persist, for offline/mentioned users | Messaging + Worker |

## Canvas

| ID | Decision | Default in docs | Recommendation | Scope |
|----|----------|-----------------|----------------|-------|
| D9 | Yjs state column type | 🟡 `bytea` (+ optional derived JSONB) — note: original architecture said JSONB | Confirm bytea; Yjs is binary | Cross-cutting (Canvas, Worker) |
| D10 | Storage strategy: snapshot-only vs. snapshot + append log | 🟡 Append log (strategy b) | Snapshot + append log — required for correct snapshots when a document's editors span several pods | Canvas |
| D11 | Server-side Yjs (`pycrdt`) vs. trust a debounced client full-state push | 🟡 `pycrdt` server-side | `pycrdt` (Rust y-crdt bindings) — crash-correct and straightforward in Python | Canvas |
| D12 | Index canvas content in Elasticsearch? | 🔴 Open | Needs a Canvas → text projection (can be produced with `pycrdt` at snapshot time) | Cross-cutting (Canvas, Worker) |

## Asset

| ID | Decision | Default in docs | Recommendation | Scope |
|----|----------|-----------------|----------------|-------|
| D13 | Avatars/exports in shared bucket vs. dedicated buckets | 🔴 Open | — | Asset |
| D14 | How Worker reports generated variants back | 🟡 Internal endpoint `POST /internal/assets/{id}/variants` | Internal endpoint (no shared DB) | Cross-cutting (Asset, Worker) |
| D15 | Virus/malware scan before marking `ready` | 🔴 Open | Add a scan hook if uploads are user-shared | Asset |
| D15b | Azure phase: direct Blob SDK vs. S3-compat layer | 🟡 S3-compat | Keep S3-compat behind the `ObjectStore` protocol | Asset |
| D15c | Upload size enforcement: presigned PUT + check on confirm vs. presigned POST policy | 🟡 PUT + check on confirm | Switch to POST policy if oversized uploads must be blocked before they land | Asset |

## Worker

| ID | Decision | Default in docs | Recommendation | Scope |
|----|----------|-----------------|----------------|-------|
| D16 | Retention policy values (message days, pending-asset hours, etc.) | 🔴 Open | Set concrete numbers; coordinate with D8d | Worker (+ Messaging) |
| D17 | Specialised worker pools vs. one deployment for all streams | 🟡 Single, split suggested | Split CPU-heavy (thumbnail/export) from IO-heavy (index/notify) | Worker |
| D18 | Notification channels in v1 | 🔴 Open | In-app first; push/email later | Worker |
| D19 | Canvas export rendering engine (`pycrdt` + skia-python/CairoSVG, or headless browser via Playwright) | 🔴 Open | Non-trivial; consider deferring | Worker |
| D26 | Retention orchestration: Worker calls each service's internal purge endpoint vs. each service runs its own scheduled purge | 🟡 Worker-orchestrated via internal endpoints | Either is fine; Worker must not touch service DBs | Cross-cutting |

## Frontend SPA

| ID | Decision | Default in docs | Recommendation | Scope |
|----|----------|-----------------|----------------|-------|
| D20 | React Native vs. PWA for mobile | 🔴 Open | Drives how much of `/lib` is platform-agnostic | Frontend |
| D21 | Canvas renderer: Konva vs. PixiJS/WebGL vs. custom | 🔴 Open | Driven by expected document complexity | Frontend |
| D22 | Refresh-token storage: HttpOnly cookie vs. in-app secure storage | 🔴 Open | HttpOnly cookie if same-site allows; avoid localStorage | Frontend (+ Auth) |
| D23 | Typed REST clients: generated from OpenAPI vs. hand-written | 🟡 Generated suggested | Generate from each service's FastAPI OpenAPI schema | Frontend |
| D24 | Client state manager: Zustand vs. Redux Toolkit | 🔴 Open | — | Frontend |

---

## Notes
- "Default in docs" means the source doc already assumes this and built around it; changing it
  means editing that doc.
- Items marked 🟡 are safe to proceed on for early scaffolding; the 🔴 items mostly affect
  later-stage work (mobile path, exports, search breadth) and can be deferred without blocking
  a first build of the core services.
- 2026-11-19: D0 decided (Python replaces .NET). D27–D29, D15c, D25, D26 added during the
  migration review.
