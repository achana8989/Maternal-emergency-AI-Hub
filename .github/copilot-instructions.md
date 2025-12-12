<!-- .github/copilot-instructions.md for AI coding agents -->
# Copilot instructions — Maternal Emergency AI Hub (backend)

Summary
- This repository provides a small FastAPI backend that manages users and patients. The backend lives under `backend/app` and is the primary area where you'll make changes.

Architecture / Big picture
- FastAPI app entry: [backend/app/main.py](backend/app/main.py#L1-L20) — registers routers and runs `init_db()` on startup to create SQLAlchemy tables.
- Routers: [backend/app/routers](backend/app/routers) — `auth.py` (JWT auth + token endpoint `/api/auth/token`), `patients.py` (authenticated patient CRUD), `health.py` (health check `/api/health`).
- DB layer: SQLAlchemy (declarative) in [backend/app/db.py](backend/app/db.py#L1-L60) with `SessionLocal` and `Base`. Models are in [backend/app/models.py](backend/app/models.py#L1-L80).
- Business logic: [backend/app/crud.py](backend/app/crud.py#L1-L120) — single-source for DB operations. Prefer editing/adding functions here for data-layer changes.
- Schemas: Pydantic models in [backend/app/schemas.py](backend/app/schemas.py#L1-L120). All response Pydantic models use `orm_mode = True`.

Key patterns & conventions (use these exactly)
- DB session: use the `get_db()` generator which yields `SessionLocal()` and always `close()` it in `finally`. Example usage in routers: `db: Session = Depends(get_db)`.
- Auth: token payload uses `sub` as `user.id` (string). Token endpoint is `/api/auth/token`. Use `get_current_user_from_token` from `auth.py` as the dependency to secure endpoints.
- CRUD: low-level DB operations live in `crud.py`. Routers call `crud` functions and return Pydantic `response_model` types from `schemas.py`.
- Migrations: `alembic` is present in `requirements.txt` but no migrations folder checked in; do not assume migrations exist. `init_db()` will create tables automatically for local/dev use.
- Passwords: `passlib[bcrypt]` is used; hashing and verification are implemented in `crud.py` via `pwd_context`.

Run / debug / build (developer workflows)
- Local dev (Python): from the `backend` directory run:

```bash
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

- Docker: image built from `backend/Dockerfile`. Container runs `uvicorn app.main:app --host 0.0.0.0 --port 8000`.
- Environment: config reads env vars in `backend/app/core/config.py`. Defaults:
  - `DATABASE_URL` -> `sqlite:///./dev.db`
  - `JWT_SECRET` -> `change_this_to_a_long_random_secret`
  - `ACCESS_TOKEN_EXPIRE_MINUTES` -> `60`

Concrete examples to reference when coding
- Add a new authenticated endpoint:
  - Add route in `backend/app/routers/` (use APIRouter).
  - Use `db: Session = Depends(get_db)` and `current_user = Depends(get_current_user_from_token)`.
  - Call or add a helper in `backend/app/crud.py` for DB logic and return Pydantic `response_model` defined in `backend/app/schemas.py`.

- Create new model and endpoint:
  - Add SQLAlchemy model to `backend/app/models.py` and import it indirectly (no circular imports) so `init_db()` can pick it up.
  - Add corresponding Pydantic schema to `backend/app/schemas.py` with `orm_mode = True`.
  - Add CRUD functions in `backend/app/crud.py` and wire routes under `backend/app/routers/`.

Integration points & dependencies
- JWT tokens: `python-jose` encodes/decodes; token verification raises `HTTPException(401)` on failure — follow existing error handling style in `auth.py`.
- Database: SQLAlchemy engine created in `db.py`. For SQLite the code sets `check_same_thread=False`.
- Requirements: see `backend/requirements.txt`. Python 3.12 is used in the Dockerfile.

Style / small rules
- Keep router path prefixes as full paths (e.g., `/api/patients`) — do not add global prefixes in `main.py`.
- Use `response_model=` on FastAPI route decorators to enforce serialization via Pydantic. Use `orm_mode` for models returned straight from SQLAlchemy.
- Avoid raw SQL unless necessary; prefer SQLAlchemy query API in `crud.py`.

Notes & gotchas
- No user-facing frontend in this repo; assume the backend only. End-to-end tests or UI contracts may be external.
- There is an explicit `init_db()` call on startup — new models will be auto-created in dev but consider migrations for production.

If anything is unclear or you want the file to include additional project-specific shortcuts (CI commands, test commands, or examples), tell me what to add and I will iterate.
