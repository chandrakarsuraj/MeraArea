 problem-service: design, stack and build plan
 
Build the backend first. The web frontend starts only after the problem-service API
is stable (see the last section). This file is context for Claude Code: read it
together with the root `CLAUDE.md` and `docs/ROADMAP.md`.
 
Covers roadmap steps 01-05 (scaffold, location data, create/read, feed, MP/MLA lookup).
Auth and votes come later; problem-service must work before they exist.
 
## 1. What problem-service owns
 
- Problems: create, read, list (feed), update and delete (owner only)
- Location reference data: states, assembly and parliamentary constituencies
- MP and MLA lookup for a constituency
- Constituency search (autocomplete)
- Keeping `upvote_count` current by consuming `vote.cast` events (added in step 09)
It does not own users or votes. It stores only the user id taken from the JWT.
 
## 2. How a request flows (concept → analogy → diagram → code)
 
**Concept:** every request passes through four layers, each with one job.
**Analogy:** a restaurant. The router is the waiter (takes the order, checks it is valid),
the service is the chef (applies the rules), the repository is the pantry clerk
(the only one who touches storage), and the database is the pantry.
 
```
HTTP request
   │
   ▼
router        validate input with Pydantic, call the service, shape the response
   │
   ▼
service       business rules (e.g. locality required, constituency belongs to state)
   │
   ▼
repository    SQLAlchemy async queries only, no business rules
   │
   ▼
PostgreSQL    schema "problems"
```
 
```python
# router: thin
@router.post("/v1/problems", response_model=ProblemOut, status_code=201)
async def create_problem(
    body: ProblemCreate,
    user: CurrentUser = Depends(get_current_user),
    svc: ProblemService = Depends(get_problem_service),
):
    return await svc.create(body, user_id=user.id)
```
 
## 3. Tech stack and skills
 
| Area | Choice | Skill to be comfortable with |
|---|---|---|
| Language | Python 3.12 | async/await, type hints |
| API | FastAPI | routers, dependency injection, lifespan events, exception handlers |
| Validation | Pydantic v2 + pydantic-settings | schemas, validators, env-based config |
| ORM | SQLAlchemy 2.0 async (`Mapped`, `select()`, `AsyncSession`) | sessions per request, avoiding N+1 |
| Driver | asyncpg | connection pooling |
| Migrations | Alembic with async env | autogenerate then review by hand |
| Database | PostgreSQL | composite indexes, `pg_trgm`, keyset pagination, `EXPLAIN ANALYZE` |
| Tests | pytest, pytest-asyncio, httpx `AsyncClient` | fixtures, test database, dependency overrides |
| Quality | ruff, mypy | run both before every commit |
| Packaging | Docker, Docker Compose | multi-stage image, healthchecks |
| Later | Redis | cache, streams (steps 09-10) |
 
Pin exact versions in the requirements files right after the first install, then commit them.
 
Runtime dependencies (`requirements.txt`):
`fastapi`, `uvicorn[standard]`, `sqlalchemy[asyncio]`, `asyncpg`, `alembic`,
`pydantic`, `pydantic-settings`.
 
Dev dependencies (`requirements-dev.txt`, includes `-r requirements.txt`):
`pytest`, `pytest-asyncio`, `httpx`, `ruff`, `mypy`.
 
Ask before adding anything else (for example a JSON logging package).
 
## 4. Where to create the virtual environment
 
**One virtual environment per service, inside the service folder.** Not at the repo root.
Each service has different dependencies, and its Docker image installs its own set.
 
```
apnaarea/
└── services/
    └── problem-service/
        ├── .venv/                 # gitignored
        ├── requirements.txt
        ├── requirements-dev.txt
        └── ...
```
 
```bash
cd services/problem-service
python3.12 -m venv .venv
source .venv/bin/activate          # Windows PowerShell: .venv\Scripts\Activate.ps1
pip install -r requirements-dev.txt
```
 
Rules:
- Add `.venv/`, `__pycache__/`, `.env`, `.mypy_cache/`, `.ruff_cache/` to `.gitignore`.
- In your editor, select `services/problem-service/.venv` as the interpreter.
- Activate the venv first, then launch `claude` from the same terminal so it inherits the
  environment. To be safe, write commands in the service `CLAUDE.md` with explicit paths
  such as `.venv/bin/pytest` (Windows: `.venv\Scripts\pytest`).
- Docker does not use `.venv`. The Dockerfile runs its own `pip install`.
- Postgres and Redis run in Docker Compose, not in the venv.
## 5. Folder layout
 
```
services/problem-service/
├── CLAUDE.md                  # service-specific commands and notes
├── Dockerfile
├── requirements.txt
├── requirements-dev.txt
├── alembic.ini
├── alembic/                   # env.py (async), versions/
├── app/
│   ├── main.py                # app factory, lifespan, exception handlers
│   ├── config.py              # pydantic-settings
│   ├── db.py                  # async engine, session dependency
│   ├── logging.py             # JSON logging, request id
│   ├── errors.py              # AppError types and handlers
│   ├── deps.py                # get_current_user (dev stub, then JWT)
│   ├── models/                # SQLAlchemy models
│   ├── schemas/               # Pydantic request/response models
│   ├── repositories/
│   ├── services/
│   └── routers/               # health, locations, problems
└── tests/
    ├── conftest.py            # test DB, client, overrides
    └── ...
```
 
## 6. Data model (schema `problems`)
 
| Table | Key columns |
|---|---|
| `state` | id, name, code |
| `parliamentary_constituency` | id, state_id, number, name |
| `assembly_constituency` | id, state_id, parliamentary_constituency_id, number, name |
| `representative` | id, role (`MP` or `MLA`), name, party, assembly_constituency_id or parliamentary_constituency_id, term_start, source, as_of |
| `problem` | id (bigint identity), title, description, state_id, assembly_constituency_id, locality_type (`village` or `city`), locality_name, created_by (user id, no foreign key because it lives in another schema), upvote_count (default 0), status, created_at, updated_at |
 
Design decisions:
- A problem stores the **assembly** constituency. Its parliamentary constituency and the
  MP are derived through the assembly constituency.
- Locality starts as validated free text (`locality_type` + `locality_name`) because a
  reliable village-to-constituency mapping is not guaranteed in the open datasets.
  Revisit later with a locality table if the data allows.
- TODO(owner): confirm whether creating a problem requires all of state, constituency,
  village and city, or "village or city". The model above assumes the second.
Indexes:
- `problem (upvote_count DESC, id DESC)` for the national feed
- `problem (state_id, upvote_count DESC, id DESC)`
- `problem (assembly_constituency_id, upvote_count DESC, id DESC)`
- GIN `pg_trgm` index on `assembly_constituency.name` and `problem.locality_name`
- Enable the extension in a migration: `CREATE EXTENSION IF NOT EXISTS pg_trgm;`
## 7. API (v1)
 
| Method and path | Access | Notes |
|---|---|---|
| `GET /health` | public | liveness and DB check |
| `GET /v1/states` | public | list of states |
| `GET /v1/constituencies?state_id=&q=&limit=` | public | autocomplete with `pg_trgm` |
| `GET /v1/constituencies/{id}/representatives` | public | returns MLA and MP, with source and as-of date |
| `POST /v1/problems` | authenticated | state, constituency and locality required |
| `GET /v1/problems` | public | feed: filters `state_id`, `constituency_id`, `locality`, plus `cursor`, `limit` |
| `GET /v1/problems/{id}` | public | single problem |
| `PATCH /v1/problems/{id}` | owner only | title and description |
| `DELETE /v1/problems/{id}` | owner only | |
 
Feed rules:
- Order by `upvote_count DESC, id DESC`. **Keyset pagination only, never OFFSET.**
- The cursor is opaque: base64 of `{"v": <upvote_count>, "id": <id>}`. Default limit 20, max 50.
- Response: `{"items": [...], "next_cursor": "..." | null}`.
```sql
SELECT id, title, upvote_count
FROM problems.problem
WHERE (:constituency_id IS NULL OR assembly_constituency_id = :constituency_id)
  AND (upvote_count, id) < (:cursor_votes, :cursor_id)
ORDER BY upvote_count DESC, id DESC
LIMIT :limit;
```
 
Error shape (all errors): `{"error": {"code": "NOT_FOUND", "message": "..."}}`
with 401, 403, 404, 422 and a catch-all 500 that logs the stack trace.
 
## 8. Auth before auth-service exists
 
Add a `get_current_user` dependency with two modes, chosen by env var `AUTH_MODE`:
- `dev`: returns a fixed test user id (never allowed in production config)
- `jwt`: verifies signature and expiry of the JWT issued by auth-service (step 07)
Routers depend only on `get_current_user`, so swapping modes changes no router code.
 
## 9. Environment variables
 
`DATABASE_URL`, `AUTH_MODE`, `JWT_PUBLIC_KEY` or `JWT_SECRET` (step 07), `LOG_LEVEL`,
`REDIS_URL` (step 09). Keep `.env.example` current and never commit `.env`.
 
## 10. Build order for Claude Code
 
For each step: run `/create-spec <step> <name>`, review the spec, switch to Plan Mode,
implement, review the git diff, run the quality gates, then commit.
 
| Step | Goal | Done when |
|---|---|---|
| 01 | Scaffold: Compose with Postgres, venv, FastAPI app with `/health`, config, logging, error handlers, Alembic async setup, first test | `docker compose up -d`, `pytest -q` and `curl /health` all pass |
| 02 | ETL script in `scripts/`: load states, constituencies, MLAs and MPs from open datasets | Row counts logged; rerunning the script is idempotent; source and as-of stored |
| 03 | `POST /v1/problems` and `GET /v1/problems/{id}` with Pydantic validation and the dev auth stub | Tests for 201, 404, 422 and 401 pass; invalid constituency for state is rejected |
| 04 | Feed with filters, cursor pagination and constituency search | No duplicates or gaps across pages in a test; `EXPLAIN` shows index use |
| 05 | Representatives endpoint | Returns MLA and MP for a constituency, with source and as-of date |
| 05b | `PATCH` and `DELETE` for owners | Non-owner gets 403; missing gets 404 |
 
Starter prompt for step 01 (paste in Plan Mode):
 
> Read CLAUDE.md, docs/ROADMAP.md and docs/problem-service-plan.md. Plan step 01 only:
> the problem-service skeleton under services/problem-service, using the layout in
> section 5. Do not write code yet. List files, dependencies and the test you will add.
 
Quality gates before every commit, from `services/problem-service`:
 
```bash
ruff check . && ruff format --check .
mypy app
pytest -q
```
 
## 11. Claude Code practice points for this service
 
- Plan Mode for every step; keep one task per commit.
- Subagents: `code-reviewer` after each step; `security-reviewer` before step 07.
- Hook: run `ruff` after file edits; block edits to `.env`.
- Postgres MCP: let Claude inspect the real schema and run `EXPLAIN` on the feed query.
- Log any wrong, insecure or over-engineered suggestion in `docs/AI_FAILURE_LOG.md`.
  Likely candidates here: OFFSET pagination, sync SQLAlchemy calls inside async routes,
  missing indexes, trusting a user id from the request body.
## 12. Frontend (starts after step 05)
 
Only begin once the API is stable. Generate the TypeScript types from the service's
`/openapi.json` so the UI and API cannot drift.
 
Planned look, mobile first:
- Top bar: ApnaArea name, "Report a problem" button, Google sign-in
- Filter bar: State dropdown, Constituency search box with autocomplete, Village or City field
- Representatives panel: appears when a constituency is selected, showing MLA and MP with source and date
- Feed: cards with title, short description, location chips, upvote button with count, time posted
- Infinite scroll with an `IntersectionObserver` that requests the next cursor
- Create form with the mandatory location fields validated before submit
Use React + Vite + Tailwind. Let Claude Code build it and use the Playwright MCP so it can
open the running app and fix visual bugs itself. Decide the colours and detailed styling
when you reach this stage.