# ApnaArea

Civic problem platform for India. Users sign in with Google, post problems for their
area (state → constituency → village/city), browse a ranked feed, and upvote.
Selecting a constituency also shows its MLA and MP.

Budget is zero: free tiers and self-hosted components only. No paid services.

## Architecture

Monorepo. Three backend services plus a static frontend, routed by one Ingress.

| Part | Port | Owns |
|---|---|---|
| `services/auth-service` | 8001 | Google ID-token verification, users, issuing our own JWT |
| `services/problem-service` | 8002 | Problems, filters, search, location data, MP/MLA lookup, feed |
| `services/vote-service` | 8003 | Upvotes, duplicate protection, emits `vote.cast` events |
| `web` | 8080 | React + Vite + Tailwind, built to static files, served by nginx |

- PostgreSQL: one instance, **one schema per service** (`auth`, `problems`, `votes`).
  A service may only read and write its own schema.
- Redis: feed cache (page 1), vote dedupe, rate limiting, and the `vote.cast` stream.
  `vote-service` publishes; `problem-service` consumes and updates `upvote_count`.
- Services talk only via HTTP or Redis. **Never import code from another service.**
- Deploy target: k3s on a free VM. Local dev: Docker Compose, then `kind` for k8s tests.

## Repo layout

```
services/<name>/   app/ (routers, services, repositories, models, schemas), tests/, Dockerfile, CLAUDE.md
web/               React app
deploy/k8s/        Kubernetes manifests
deploy/compose.yaml
scripts/           ETL: load location, MLA and MP data
docs/              design doc, scaling.md, AI_FAILURE_LOG.md
.claude/           shared settings, subagents, skills
```

## Commands (update as the project is scaffolded)

- Start local stack: `docker compose -f deploy/compose.yaml up -d`
- Run a service: `cd services/<name> && uvicorn app.main:app --reload --port <port>`
- Tests: `cd services/<name> && pytest -q`
- Lint and format: `ruff check . && ruff format .`
- Types: `mypy app`
- Migrations: `cd services/<name> && alembic upgrade head`
- Build image: `docker build -t apnaarea/<name>:dev services/<name>`

## Tech and code conventions

- Python 3.12, FastAPI, SQLAlchemy 2.0 async with asyncpg, Alembic, Pydantic v2,
  pydantic-settings, httpx for outbound calls.
- Layering: router → service → repository. No SQL in routers. No business logic in repositories.
- All config from environment variables via pydantic-settings. Never hardcode or commit secrets.
  `.env` is gitignored; keep `.env.example` current.
- Type hints everywhere. Async all the way down; no blocking calls in request handlers.
- Errors return one consistent JSON shape: `{"error": {"code": "...", "message": "..."}}`.
  Use 401/403 for auth, 404 for missing, 422 for validation, 502/504 for upstream failures.
  Add a catch-all handler for unexpected errors that logs the stack trace and returns a generic 500.
- Logging: structured JSON logs with a request id. Never log tokens or personal data.
- Dependencies: ask before adding a new one.

## Domain rules

- Location hierarchy: India → State → Assembly Constituency → Village or City.
- Each assembly constituency belongs to exactly one parliamentary constituency.
  Store the assembly constituency and **derive** the parliamentary one. MLA comes from the
  assembly constituency, MP from its parliamentary constituency.
- Creating a problem requires: state, constituency, and a locality (village or city).
  TODO(owner): confirm whether all four fields are mandatory or "village or city".
- Filters (state, constituency, village/city) are optional on reads. Constituency search
  is autocomplete using `pg_trgm`.
- MP/MLA data is seeded by `scripts/` from open datasets (Local Government Directory and
  Election Commission derived data). The UI shows the data source and an "as of" date.
- Feed: ranked by `upvote_count DESC, id DESC`, **keyset (cursor) pagination only**.
  Never use OFFSET. Page size 20. Cache the national page 1 in Redis for 30-60 seconds.
- Votes: unique `(user_id, problem_id)`. Upvote and un-upvote are idempotent.
- Auth: verify JWT signature and expiry in every protected endpoint. Take the user id from
  the token, never from the request body.

## Testing

- pytest + pytest-asyncio, httpx `AsyncClient`, a real Postgres via compose for integration tests.
- Every endpoint needs tests for success, 401 (if protected), 404 and 422.
- Mock Google token verification and any external HTTP call. Test timeout and 5xx paths.

## How to work in this repo (instructions for Claude)

- For any new service or feature, start in Plan Mode and show the plan before editing.
- Keep changes small: one task per change, so I can review each git diff.
- Before saying a task is done, run `ruff`, `mypy` and `pytest` for the touched service and
  show the results. Do not claim success without running them.
- Do not read or edit `.env` or anything under `secrets/`.
- Do not add abstractions, layers or patterns I did not ask for. Prefer the simplest thing
  that satisfies the requirement.
- If you are unsure about a requirement, ask instead of guessing.
- When I say "log this failure", append an entry to `docs/AI_FAILURE_LOG.md` with:
  what was proposed, why it was wrong, how it was detected, the fix, and the lesson.