# FastAPI Service Base

A starting point for FastAPI microservices: layered structure, async
PostgreSQL/MySQL, Redis, structured logging, health checks and one response
shape for successes and errors. Everything external is opt-in, so a fresh clone
runs with no database and no Redis.

## Quick start

Needs [uv](https://docs.astral.sh/uv/).

```bash
cp .env.example .env
uv sync --all-extras --group dev
uv run python main.py          # http://localhost:8000/docs
```

## Start a new service

1. Click **Use this template** on GitHub, or clone and run `rm -rf .git && git init`.
2. Rename it in `pyproject.toml` and set `APP_NAME` in `.env`.
3. Run `uv run pre-commit install`.
4. Turn on what you need in `.env`: `DB_ENABLED`, `REDIS_ENABLED`.

## Commands

| Task | Command |
|---|---|
| Run tests | `uv run pytest` |
| Lint, format, type check | `uv run pre-commit run --all-files` |
| New migration | `uv run alembic revision --autogenerate -m "add users"` |
| Apply migrations | `uv run alembic upgrade head` |
| Run with Docker | `docker compose up --build -d` |

## Project layout

```
app/
├── controllers/   HTTP only: one @cbv class per feature
├── services/      business logic
├── helpers/       non-business code that services reuse
├── db/            models/ (tables) and services/ (queries)
├── schemas/       request/ and response/ models
├── exceptions/    error classes and handlers
├── clients/       database and Redis connections
├── settings.py    all configuration (field names = env var names)
└── router.py      mounts routers under /api/v1
```

## Adding an endpoint

Follow `app/controllers/health_controller.py`:

1. **Model** in `app/db/models/` and import it in `app/db/models/__init__.py`,
   so Alembic can see it.
2. **Schemas** in `app/schemas/request/` and `app/schemas/response/`.
3. **Queries** in `app/db/services/`, **business logic** in `app/services/`.
4. **Controller** in `app/controllers/`, taking `DBSession` or `RedisDep` and
   returning `BaseResponse[YourResponse]`.
5. **Mount it** in `app/router.py`, then create and apply a migration.

## Conventions

- **Responses**: every response, success or error, has the shape
  `{ "message", "payload", "status", "detail" }`.
- **Errors**: raise the classes in `app/exceptions` (`NotFoundError`,
  `ConflictError`, ...), not `HTTPException`. Unexpected errors never leak
  internals.
- **Transactions**: one per request, committed before the response is sent and
  rolled back on error. Handlers don't commit.
- **Migrations**: run `alembic upgrade head` as a deploy step, not on startup.
- **Health**: `/health` is liveness and never checks dependencies.
  `/health/detailed` is readiness and returns 503 if the DB or Redis is down.
- **Logging**: every request gets an id from `X-Request-ID` (or a new one),
  shown in each log line and error. Set `APP_LOG_FORMAT=json` in production.
