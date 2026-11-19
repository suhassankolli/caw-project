# CollabHub — Frontend SPA

> React + TypeScript single-page app: chat, collaborative canvas, presence.
> Read [Platform Conventions](./00-platform-conventions.md) first.

**Status:** Draft · **Stack:** React + TypeScript (Vite) · React Native / PWA path for mobile
**Talks to:** Auth, Messaging, Canvas, Asset services (REST + Socket.IO), MinIO (direct uploads)

> The backend moved from .NET to Python; the SPA stays TypeScript. The visible changes for the
> client are the real-time library (`socket.io-client` instead of `@microsoft/signalr`) and
> that shared types now come from the Python services' OpenAPI schemas / Pydantic models.

---

## 1. Purpose & Responsibilities

The single client application. Combines a Slack-like chat experience and a Figma-like
collaborative canvas in one app. Communicates over **REST** for standard CRUD and **Socket.IO**
for all real-time interactions; runs the **Yjs CRDT** locally for the canvas (the backend is a
relay — Canvas doc §1).

**Owns:** all UI, client-side CRDT state, optimistic updates, presence rendering, the OIDC
login flow (PKCE), and direct-to-MinIO uploads.
**Does NOT own:** merge conflict resolution beyond what Yjs provides; any persistence
(everything authoritative lives server-side).

---

## 2. Tech & Key Libraries

| Concern | Choice | Notes |
|---------|--------|-------|
| Framework | React 18+ + TypeScript | |
| Build | Vite | Fast dev, PWA plugin for the mobile path. |
| Routing | React Router | |
| Server state / data fetching | TanStack Query | Caching, mutations, optimistic updates over REST. |
| Client/UI state | Zustand (or Redux Toolkit) | Lightweight global state (current user, presence). |
| Real-time | `socket.io-client` | Two connections: `/hubs/messaging` + `/hubs/canvas` (Conventions §6). |
| CRDT | `yjs` + `y-protocols` + a Socket.IO provider (custom, see §5.3) | Canvas document state. |
| Canvas rendering | `react-konva` / `pixi.js` / custom WebGL | Pick per perf needs (Open Decision). |
| Auth | `oidc-client-ts` | Authorization Code + PKCE against Auth service. |
| Forms / validation | React Hook Form + Zod | Zod schemas generated from (or checked against) the services' OpenAPI schemas, which derive from `collabhub-contracts`. |
| Typed REST clients | `openapi-typescript` (+ `openapi-fetch`) or Orval | Generated from each service's `/api/v1/openapi.json`. |
| Styling | (team choice) Tailwind / CSS Modules | |

---

## 3. App Structure

```
/src
  /app            # bootstrap, providers, router
  /lib
    /api          # typed REST clients per service (generated from OpenAPI)
    /realtime     # Socket.IO connection managers (messaging, canvas)
    /auth         # OIDC client, token storage, refresh, auth guard
    /yjs          # Yjs doc factory + Socket.IO sync provider
  /features
    /channels     # channel list, channel view, composer
    /threads
    /canvas       # document view, toolbar, layers, cursors
    /presence     # online users, live cursors, awareness
    /assets       # upload widget, image/file rendering
  /components      # shared UI
  /types           # shared DTO types (generated from OpenAPI; mirrors collabhub-contracts)
```

---

## 4. Auth Flow (OIDC + PKCE)

1. Unauthenticated → redirect to Auth service `/api/v1/auth/login/{provider}`.
2. After provider callback, the SPA receives an auth code at its redirect URI.
3. SPA exchanges code + PKCE verifier at `POST /api/v1/auth/token` → access + refresh tokens.
4. **Token storage:** access token in memory; refresh token in an HttpOnly cookie if the
   deployment supports same-site cookies (the Auth service then sets/reads the cookie and the
   refresh request has no body), else secure storage (Open Decision — avoid localStorage for
   refresh tokens).
5. A fetch interceptor attaches `Authorization: Bearer`; on 401 it calls `/api/v1/auth/refresh`
   once (rotation — serialise concurrent refreshes behind one in-flight promise, otherwise the
   second request presents a rotated token and revokes the whole family) and retries, else
   routes to login.
6. Socket.IO connections pass the access token in the handshake `auth` option as a callback
   (`io(url, { path: "/hubs/messaging", auth: (cb) => cb({ token: getAccessToken() }) })`), so
   each reconnect sends a fresh token. On `TokenExpired` or a `connect_error` with
   `unauthorized`, refresh the token and reconnect.

---

## 5. Real-Time Integration

### 5.1 Connection management
Two long-lived Socket.IO sockets — path `/hubs/messaging` and `/hubs/canvas` — created lazily,
with Socket.IO's built-in reconnection (exponential backoff). Transports: WebSocket first,
long-polling fallback (requires ingress affinity; or force `transports: ["websocket"]` —
Conventions §6). On reconnect, re-emit the join events for the channels/documents the user was
in and re-sync. A single shared "connection state" surfaces offline/reconnecting UI.

### 5.2 Messaging
- On entering a channel: `emitWithAck("JoinChannel", { channelId })`; load history via REST
  `GET /api/v1/channels/{id}/messages` (cursor paginated, newest-first) and merge.
- Send via `emitWithAck("SendMessage", {...})` (with a timeout); render **optimistically**
  with a temp `clientId`, reconcile on the ack's `Message` / the `MessageReceived` event.
- Listen for `MessageReceived/Edited/Deleted`, `ReactionChanged`, `ReadReceiptUpdated`,
  `UserTyping` and update TanStack Query caches.
- Debounce `Typing`; throttle `MarkRead`.

### 5.3 Canvas (Yjs over Socket.IO)
- One `Y.Doc` per open document. A **custom Socket.IO provider** bridges Yjs ↔ the canvas hub
  (using `y-protocols` sync/awareness encoders; payloads sent as `Uint8Array`, which Socket.IO
  transmits as binary):
  - On `JoinDocument`, perform the sync handshake (`SyncStep1`/`SyncStep2`, Canvas doc §3.2),
    feeding the server's state into the local `Y.Doc`.
  - Local Yjs `update` events (origin ≠ provider) → `SyncUpdate { documentId, update }`;
    keep un-acked updates and resend after reconnect.
  - Incoming `Update` → `Y.applyUpdate(doc, update, provider)`.
  - Yjs **awareness** (cursor, selection, user color) → `AwarenessUpdate`; incoming awareness
    renders remote cursors; `PeerLeft` clears them.
- Initial load may also use REST `GET /api/v1/documents/{id}/snapshot` to seed before the hub
  handshake on slow connections.
- The renderer subscribes to the `Y.Doc` and redraws on change; all editing mutates the
  `Y.Doc`, never local-only state.

---

## 6. Assets (direct upload)
1. `POST /api/v1/assets/upload-url` → presigned PUT.
2. `PUT` the file bytes **directly to MinIO** with the returned headers (show progress).
3. `POST /api/v1/assets/{id}/confirm`.
4. Reference the returned `assetId` in a message (`attachments`) or canvas image node.
5. Render via `GET /api/v1/assets/{id}/download-url` / `thumbnail-url` (short-lived URLs;
   refetch on expiry).

MinIO must allow CORS `PUT`/`GET` from the SPA origin.

---

## 7. Cross-Cutting
- **Error handling:** parse RFC 9457 Problem Details (Conventions §4.2) into typed UI errors;
  show `errors` map on forms. Socket.IO acks carry the same shape under `error`.
- **Offline / reconnect:** queue outgoing chat sends while disconnected; Yjs naturally
  reconciles canvas edits on reconnect.
- **Observability:** browser OTel SDK → collector; propagate `traceparent` on REST calls so
  traces stitch across SPA → service → worker.
- **Accessibility & i18n:** plan from the start (Open Decision on i18n lib).

## 8. Non-Functional & Limits
- First meaningful paint and channel-switch should feel instant (optimistic + cached).
- Canvas target 60 fps with N live cursors; throttle awareness broadcasts (~30–60 ms).
- Respect server limits (message length, attachment count/size) client-side before sending.
- PWA/offline shell for the mobile path; service worker caches the app shell, not data.

## 9. Open Decisions
- **React Native vs. PWA** for mobile — separate codebase vs. shared (the architecture lists
  both). Affects how much of `/lib` is platform-agnostic. (`socket.io-client` and `yjs` both
  run in React Native.)
- **Canvas renderer:** Konva (DOM/2D, simpler) vs. PixiJS/WebGL (perf) vs. custom. Driven by
  expected document complexity.
- **Refresh-token storage:** HttpOnly cookie vs. in-app secure storage.
- **Typed clients:** generate REST clients + types from each service's OpenAPI doc vs.
  hand-write. Recommend generated — FastAPI produces the schemas for free.
- State manager choice (Zustand vs. Redux Toolkit).
