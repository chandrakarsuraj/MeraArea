# Spec: Folder Structure

## Overview
This feature establishes the initial directory layout and boilerplate for the ApnaArea monorepo. It ensures that all three backend services (`auth-service`, `problem-service`, `vote-service`) and the `web` frontend have their basic folder hierarchies in place, adhering to the architecture defined in `CLAUDE.md`.

## Depends on
None. This is the foundation step.

## Routes
No new routes.

## Database changes
No database changes.

## Templates
No templates.

## Files to change
No files to change.

## Files to create
- `services/auth-service/app/__init__.py`
- `services/auth-service/app/routers/`
- `services/auth-service/app/services/`
- `services/auth-service/app/repositories/`
- `services/auth-service/app/models/`
- `services/auth-service/app/schemas/`
- `services/auth-service/tests/`
- `services/auth-service/Dockerfile`
- `services/auth-service/CLAUDE.md`
- `services/problem-service/app/__init__.py`
- `services/problem-service/app/routers/`
- `services/problem-service/app/services/`
- `services/problem-service/app/repositories/`
- `services/problem-service/app/models/`
- `services/problem-service/app/schemas/`
- `services/problem-service/tests/`
- `services/problem-service/Dockerfile`
- `services/problem-service/CLAUDE.md`
- `services/vote-service/app/__init__.py`
- `services/vote-service/app/routers/`
- `services/vote-service/app/services/`
- `services/vote-service/app/repositories/`
- `services/vote-service/app/models/`
- `services/vote-service/app/schemas/`
- `services/vote-service/tests/`
- `services/vote-service/Dockerfile`
- `services/vote-service/CLAUDE.md`
- `web/src/`
- `web/public/`
- `deploy/k8s/`
- `scripts/`
- `docs/`

## New dependencies
No new dependencies.

## Rules for implementation
- Follow the layering: router → service → repository.
- Ensure all `app` directories contain `__init__.py` to be treated as Python packages.
- Keep directories empty initially (just the structure).
- No SQLAlchemy or ORMs (as per general rules).
- Parameterised queries only.
- Use CSS variables — never hardcode hex values.
- All templates extend `base.html`.

## Definition of done
- All directories listed in "Files to create" exist on disk.
- `git status` shows only the newly created folder structure.
- The structure matches the layout described in `CLAUDE.md`.
