# Use Python instead of .NET for the backend services

## Status
Accepted
Date: 2026-11-19 (recorded 2026-11-20)

Corresponds to decision **D0** in the [open decisions register](../design/07-open-decisions-register.md).

## Context

CollabHub's backend is five services (Auth, Messaging, Canvas, Asset, Worker) that run on
Kubernetes on-prem and later on Azure. The first design drafts assumed **.NET 10, ASP.NET Core,
C#, SignalR and EF Core**.

Before the build started, we looked at the stack again for these reasons:

- **The team is comfortable in both .NET and Python**, so skills don't decide it either way.
- **Speed of delivery matters most right now.** The design is still a draft with about 30
  open decisions, so we expect a lot of iteration and want the stack that gets working
  services out quickest.
- **We expect much of the code to be written with Claude Code.** We judged that it has the
  most experience in Python, which should make AI-assisted development faster and more
  reliable in Python than in C#.
- **The workload is mainly I/O.** It's REST, real-time fan-out over Redis, Postgres, object
  storage and search indexing. The CPU-heavy jobs (thumbnails, canvas export) are isolated
  in the Worker service.

## Decision

We will build all backend services in **Python 3.13+**. The baseline is set out in
[Platform Conventions §2.1](../design/00-platform-conventions.md):

- FastAPI on Uvicorn for REST, with Pydantic v2 for validation and DTOs
- `python-socketio` (Socket.IO) for real-time, in place of SignalR
- SQLAlchemy 2 (async) + `asyncpg` + Alembic, in place of EF Core
- `redis-py` asyncio, `httpx`, PyJWT/Authlib, `pycrdt` (Yjs), Pillow, structlog, OpenTelemetry
- A `uv` workspace mono-repo, with type checking enforced by `mypy --strict` (or pyright) in CI

The frontend stays in TypeScript/React.

## Consequences

### Positive
- Development should be faster because there's less boilerplate, the edit-run loop is
  quicker, and AI-assisted coding works better.
- FastAPI generates OpenAPI schemas automatically, so the SPA can use generated typed
  clients (D23).
- `pycrdt` binds to the Rust `y-crdt` library, so server-side Yjs handling in Canvas is
  straightforward (D11).
- The data, imaging and search tooling Python offers is mature and suits the Worker jobs.

### Negative / risks
- **SignalR has no maintained Python server.** Real-time moves to Socket.IO. That brings a
  Redis backplane per service, and ingress session affinity if HTTP long-polling stays
  enabled (D27, Conventions §6).
- **The GIL limits CPU-bound work.** CPU-heavy handlers must run in a process pool, and the
  event loop only coordinates I/O (Worker §8). Each pod will likely deliver less raw
  throughput than .NET would, so plan for more replicas.
- **Type safety is opt-in.** Correctness relies on strict mypy/pyright checking in CI and
  on Pydantic at the boundaries, not on a compiler.
- **SAML gets harder.** The Python SAML libraries need the native `xmlsec` library, which
  supports deferring SAML (D6).
- **Async discipline is needed.** A single blocking call inside an async handler stalls the
  pod. Code review and linting must catch blocking I/O.

### Follow-up actions
- Keep all design docs in line with the Python baseline (done in the 2026-11-19 revision).
- Set up the `uv` workspace, the shared `collabhub-shared` and `collabhub-contracts`
  packages, and CI with strict type checking.
- Settle D27 (Socket.IO vs. plain WebSockets) before building the real-time services.
- Load-test Messaging and Canvas fan-out early to confirm the number of pods per load.

## Alternatives considered

### .NET 10 / ASP.NET Core / SignalR / EF Core (original design)
- **Summary:** The stack the first drafts were written for.
- **Pros:** High throughput per pod with no GIL. SignalR is first-party with an Azure
  SignalR Service upgrade path. Compile-time type safety. Strong fit with Azure.
- **Cons:** More ceremony per feature. We judged AI-assisted development to work less well
  than in Python.
- **Why not chosen:** Both are viable and the team knows both. Delivery speed and
  AI-assisted development were weighted highest, and they favour Python.

### Mixed stack (Python for most services, .NET for the real-time services)
- **Summary:** Keep SignalR and .NET performance for Messaging and Canvas only.
- **Pros:** First-party real-time where load is highest.
- **Cons:** Two toolchains, two sets of shared libraries, and duplicated cross-cutting
  code (auth, telemetry, Problem Details).
- **Why not chosen:** The duplicated effort cancels out the speed gain, and Socket.IO is
  good enough for the expected load.

## References
- [00 Platform Conventions](../design/00-platform-conventions.md) §2.1 (runtime baseline) and §6 (real-time)
- [05 Worker Service](../design/05-worker-service.md) §8 (GIL and process pool)
- [07 Open Decisions Register](../design/07-open-decisions-register.md): D0, D6, D11, D23, D27
- Commit `88be193`: "Switch backend stack from .NET to Python; fix design review issues"
- FastAPI: https://fastapi.tiangolo.com/
- python-socketio: https://python-socketio.readthedocs.io/
- ASP.NET Core SignalR: https://learn.microsoft.com/aspnet/core/signalr/introduction
