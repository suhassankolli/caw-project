# CollabHub — Auth Service

> Identity, token issuance, and authorization source of truth.
> Read [Platform Conventions](./00-platform-conventions.md) first.

**Status:** Draft · **Runtime:** Python 3.13+ / FastAPI (Uvicorn)
**Owns:** users, external identities, roles, workspace membership, tokens, service clients
**Depends on:** PostgreSQL (own DB), Redis Cache (R1)

---

## 1. Purpose & Responsibilities

The Auth Service is the identity provider and authorization source of truth for CollabHub.

**Owns:**
- User accounts and profiles.
- Federation with external identity providers via **OpenID Connect** (internal SSO) and
  optionally **SAML 2.0** (enterprise federation).
- Issuance, refresh, and revocation of CollabHub access/refresh tokens (JWTs).
- Service-client credentials and the `client_credentials` grant for internal callers
  (Conventions §5.1).
- The JWKS endpoint that all other services use to verify tokens.
- Workspace membership and workspace-level roles (`owner`/`admin`/`member`/`guest`).

**Does NOT own:**
- Per-resource permissions (each service owns its own — see Conventions §5.3).
- Sessions for real-time connections (services validate tokens themselves).

---

## 2. Runtime & Dependencies

- FastAPI routers (async), Pydantic v2 models.
- **Authlib** (Starlette/httpx integration) as the OIDC relying party for external IdPs.
- **python3-saml** (OneLogin) or **pysaml2** for SAML, if SAML is in scope (D6). Both depend on
  `xmlsec`/libxml2 native libraries — bake them into the image.
- **PyJWT** (`pyjwt[crypto]`, backed by `cryptography`) for RS256 signing and JWKS generation.
- SQLAlchemy 2.0 async + `asyncpg`; Alembic migrations.
- `redis.asyncio` for the R1 denylist + profile cache.
- Signing keys: RSA key pair stored as a secret; supports key rotation (multiple `kid`s,
  one active for signing, several valid for verification).
- OAuth server-side state for login (`state`, `nonce`, PKCE verifier toward the upstream IdP)
  is held in a signed, short-lived cookie (Starlette `SessionMiddleware`) or R1 — not in process
  memory, since any pod may receive the callback.

---

## 3. Public Interface (REST)

Business endpoints live under `/api/v1`; discovery endpoints live at the root
(Conventions §4). Token, login/callback and well-known endpoints are unauthenticated; all
others require Bearer.

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/.well-known/jwks.json` | none | Public verification keys (JWKS). |
| GET | `/.well-known/openid-configuration` | none | OIDC discovery (if acting as OP — D5). |
| GET | `/api/v1/auth/login/{provider}` | none | Begin external OIDC/SAML login (redirect). |
| GET/POST | `/api/v1/auth/callback/{provider}` | none | Provider callback (GET for OIDC, POST for SAML ACS); establishes identity, redirects to SPA with auth code. |
| POST | `/api/v1/auth/token` | none / client secret | Exchange auth code (PKCE) for tokens; or `client_credentials` for service tokens. |
| POST | `/api/v1/auth/refresh` | none | Rotate refresh token → new access + refresh. |
| POST | `/api/v1/auth/logout` | Bearer | Revoke current token's `jti` + refresh-token family. |
| GET | `/api/v1/auth/userinfo` | Bearer | Claims for the current token. |
| GET | `/api/v1/users/me` | Bearer | Current user profile. |
| PATCH | `/api/v1/users/me` | Bearer | Update own profile (display name, avatar ref). |
| GET | `/api/v1/users/{id}` | Bearer | Public profile of a user (name, avatar). |
| GET | `/api/v1/workspaces/{id}/members` | Bearer (member) | List members + roles. |
| POST | `/api/v1/workspaces/{id}/members` | Bearer (admin) | Invite/add member. |
| PATCH | `/api/v1/workspaces/{id}/members/{userId}` | Bearer (admin) | Change role. |
| DELETE | `/api/v1/workspaces/{id}/members/{userId}` | Bearer (admin) | Remove member. |

### 3.1 Token endpoint payloads

Field names here are snake_case because RFC 6749 defines them (exception to the camelCase
rule in Conventions §4).

`POST /api/v1/auth/token` (authorization code)
```json
// request
{ "grant_type": "authorization_code", "code": "...", "code_verifier": "...", "redirect_uri": "..." }
// response
{ "access_token": "ey...", "refresh_token": "def...", "token_type": "Bearer", "expires_in": 900 }
```

`POST /api/v1/auth/token` (service token)
```json
// request
{ "grant_type": "client_credentials", "client_id": "worker", "client_secret": "..." }
// response — no refresh_token
{ "access_token": "ey...", "token_type": "Bearer", "expires_in": 900 }
```

`POST /api/v1/auth/refresh`
```json
// request (omit the body field if the refresh token is sent as an HttpOnly cookie — D22)
{ "refresh_token": "def..." }
// response — same shape as the authorization-code response; old refresh_token is now invalid
```

Access-token claims follow Conventions §5.1. The Auth Service is the only issuer.

---

## 4. Data Model (PostgreSQL)

```sql
CREATE EXTENSION IF NOT EXISTS citext;

CREATE TABLE users (
    id            uuid PRIMARY KEY,
    email         citext NOT NULL UNIQUE,
    display_name  text NOT NULL,
    avatar_asset  uuid NULL,                 -- references Asset service id (no FK, cross-service)
    status        text NOT NULL DEFAULT 'active',  -- active | suspended | deactivated
    created_at    timestamptz NOT NULL DEFAULT now(),
    updated_at    timestamptz NOT NULL DEFAULT now(),
    deleted_at    timestamptz NULL,
    version       integer NOT NULL DEFAULT 0
);

-- Links a user to an external IdP identity (one row per provider).
CREATE TABLE external_identities (
    id            uuid PRIMARY KEY,
    user_id       uuid NOT NULL REFERENCES users(id),
    provider      text NOT NULL,             -- 'oidc:acme' | 'saml:corp'
    subject       text NOT NULL,             -- the IdP's sub / NameID
    created_at    timestamptz NOT NULL DEFAULT now(),
    UNIQUE (provider, subject)
);

CREATE TABLE workspaces (
    id            uuid PRIMARY KEY,
    name          text NOT NULL,
    created_at    timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE workspace_members (
    workspace_id  uuid NOT NULL REFERENCES workspaces(id),
    user_id       uuid NOT NULL REFERENCES users(id),
    role          text NOT NULL,             -- owner | admin | member | guest
    joined_at     timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (workspace_id, user_id)
);

-- Refresh tokens are persisted (rotation + revocation). Access tokens are NOT stored.
CREATE TABLE refresh_tokens (
    id            uuid PRIMARY KEY,
    family_id     uuid NOT NULL,             -- shared by every token in one rotation chain (= id of the first)
    user_id       uuid NOT NULL REFERENCES users(id),
    token_hash    bytea NOT NULL UNIQUE,     -- SHA-256 of the opaque refresh token
    issued_at     timestamptz NOT NULL DEFAULT now(),
    expires_at    timestamptz NOT NULL,
    rotated_to    uuid NULL REFERENCES refresh_tokens(id),  -- set when rotated
    revoked_at    timestamptz NULL
);
CREATE INDEX ix_refresh_user   ON refresh_tokens (user_id)   WHERE revoked_at IS NULL;
CREATE INDEX ix_refresh_family ON refresh_tokens (family_id) WHERE revoked_at IS NULL;

-- Internal service clients for the client_credentials grant.
CREATE TABLE service_clients (
    client_id     text PRIMARY KEY,          -- 'worker', ...
    secret_hash   text NOT NULL,             -- argon2id hash (argon2-cffi)
    roles         text[] NOT NULL DEFAULT '{service}',
    created_at    timestamptz NOT NULL DEFAULT now(),
    disabled_at   timestamptz NULL
);
```

### 4.1 Redis (R1) usage
- `auth:revoked:{jti}` → `"1"`, TTL = remaining access-token lifetime. Written on logout.
- `auth:userinfo:{userId}` → cached profile blob, short TTL, to speed up `/userinfo`.
- `auth:login:{state}` → upstream-IdP login state (if not using a signed cookie), TTL ~10 min.

---

## 5. Internal Design

### 5.1 Login flow (OIDC, PKCE)
1. SPA hits `/api/v1/auth/login/{provider}` → 302 to external IdP (Authlib `authorize_redirect`).
2. IdP redirects to `/api/v1/auth/callback/{provider}`; service validates (Authlib
   `authorize_access_token`, ID-token signature + `nonce`), upserts `users` +
   `external_identities`, then redirects to the SPA redirect URI with a short-lived,
   single-use auth code (stored in R1, bound to the SPA's PKCE `code_challenge`).
3. SPA calls `/api/v1/auth/token` with the code + PKCE `code_verifier` → access + refresh tokens.

### 5.2 Refresh rotation
On `/auth/refresh`: look up by `token_hash`, ensure not revoked/expired, mint a new pair in the
same `family_id`, set `rotated_to` and `revoked_at` on the old row — in one transaction with
`SELECT ... FOR UPDATE` so two concurrent refreshes cannot both succeed. Presenting an
already-rotated token ⇒ revoke every row in its `family_id` and return 401 (token-theft
detection).

### 5.3 Token verification (for other services)
Other services do NOT call this service per request. They fetch JWKS and verify locally
(Conventions §5). This service only needs to keep JWKS current and maintain the denylist.

---

## 6. Configuration

Common vars per Conventions §8, plus:

| Var | Notes |
|-----|-------|
| `AUTH__SIGNING_KEY` | PEM RSA private key (secret). Active `kid`. |
| `AUTH__PREVIOUS_KEYS` | Prior public keys still valid for verification (rotation). |
| `AUTH__ACCESS_TOKEN_MINUTES` | Default 15. |
| `AUTH__REFRESH_TOKEN_DAYS` | Default 30. |
| `AUTH__SESSION_SECRET` | Secret for the signed login-state cookie. |
| `OIDC__<PROVIDER>__AUTHORITY` / `__CLIENT_ID` / `__CLIENT_SECRET` | Per external IdP. |
| `SAML__<PROVIDER>__METADATA_URL` | Per SAML IdP. |
| `SPA__REDIRECT_URI` | Allowed post-login redirect. |

---

## 7. Cross-Cutting
Error envelope, health checks, observability, logging per Conventions §4.2, §9, §10.
Emit metrics `auth_tokens_issued_total`, `auth_logins_total{provider}`, `auth_refresh_reuse_total`.

---

## 8. Non-Functional & Limits
- Token issuance p99 < 100 ms. RSA signing is CPU-bound but sub-millisecond for 2048-bit keys;
  it can run inline on the event loop.
- JWKS cacheable (`Cache-Control`), so verification scales without hitting this service.
- Rate-limit `/auth/token` and `/auth/refresh` per IP + per user (R1-backed limiter, e.g.
  `limits` with a Redis storage, or at the ingress).

---

## 9. Open Decisions
- Acting as a full OIDC OP for first-party clients vs. only federating to an upstream IdP.
- Multi-workspace membership and the workspace-switch flow (affects the `wsp` claim — see
  Conventions §12.2).
- Whether SAML is in scope for v1 or deferred.
- Refresh tokens stored hashed in Postgres here; could move to R1 — confirm durability need.
