---
# Spec: Venv and Requirements for Problem Service

## Overview
This feature sets up the Python development environment for the `problem-service`. Following the project architecture, each service maintains its own virtual environment and dependency list to ensure isolation and reproducibility, especially when building Docker images.

## Depends on
No previous steps required for this specific setup, though it is the foundational part of the `problem-service` scaffolding.

## Routes
No new routes.

## Database changes
No database changes.

## Templates
No templates.

## Files to change
No files to change.

## Files to create
- `services/problem-service/requirements.txt`
- `services/problem-service/requirements-dev.txt`

## New dependencies
**Runtime (`requirements.txt`):**
- `fastapi`
- `uvicorn[standard]`
- `sqlalchemy[asyncio]`
- `asyncpg`
- `alembic`
- `pydantic`
- `pydantic-settings`

**Development (`requirements-dev.txt`):**
- `-r requirements.txt`
- `pytest`
- `pytest-asyncio`
- `httpx`
- `ruff`
- `mypy`

## Rules for implementation
- Virtual environment must be created inside `services/problem-service/.venv`.
- All dependencies must be pinned to exact versions after the first installation.
- `.venv/`, `__pycache__/`, `.env`, `.mypy_cache/`, and `.ruff_cache/` must be added to `.gitignore` (or verified to be there).
- Use Python 3.12.

## Definition of done
- `services/problem-service/requirements.txt` and `requirements-dev.txt` exist with the correct packages.
- A virtual environment is successfully created in `services/problem-service/.venv`.
- `pip install -r services/problem-service/requirements-dev.txt` completes without errors.
- `ruff --version` and `mypy --version` run successfully from the venv.
---
