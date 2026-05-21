# Research: TODO API — FastAPI + SQLAlchemy + SQLite

**Feature**: 001-todo-api-crud
**Phase**: 0 — Outline & Research
**Date**: 2026-05-20

---

## Resolved Technical Decisions

### Decision 1: ORM — SQLAlchemy 2.x (Async)

**Decision**: Use SQLAlchemy 2.x with `AsyncSession` and the `aiosqlite` driver.

**Rationale**: FastAPI is an async framework. SQLAlchemy 2.x provides a first-class async API (`AsyncEngine`, `AsyncSession`) that integrates cleanly with FastAPI's dependency injection via `Depends`. Using sync SQLAlchemy inside async route handlers would block the event loop, violating FastAPI best practices.

**Note on user input**: The user referenced "Entity Framework Core" — that is a .NET ORM. The Python/SQLAlchemy equivalent is SQLAlchemy, which provides identical capabilities: declarative models, migrations (via Alembic), and session lifecycle management.

**Alternatives considered**:
- Sync SQLAlchemy with `sqlite` driver — rejected; blocks the event loop in async handlers.
- Tortoise ORM — rejected; less mature ecosystem and heavier learning curve.
- Raw `aiosqlite` — rejected; too low-level, no schema management, no query builder.

**Dependencies**: `sqlalchemy[asyncio]>=2.0`, `aiosqlite>=0.19`

---

### Decision 2: Testing Framework — pytest (not "pyunit")

**Decision**: Use `pytest` with `pytest-asyncio` and FastAPI `TestClient` (via `httpx`).

**Rationale**: Python has no framework called "pyunit". The user likely meant `pytest` (industry standard) or the built-in `unittest` module. The project constitution **mandates pytest** as non-negotiable. `pytest-asyncio` enables async test functions; `httpx.AsyncClient` + `anyio` or `TestClient` enables in-process HTTP testing without a live server.

**Alternatives considered**:
- `unittest` — rejected; constitution mandates pytest, and pytest is a strict superset.
- `pytest-anyio` — considered; `pytest-asyncio` is more widely documented for FastAPI patterns.

**Dependencies**: `pytest>=8.0`, `pytest-asyncio>=0.23`, `httpx>=0.27`

---

### Decision 3: Vertical Slice Architecture Layout

**Decision**: Organize source code by **feature module** (`todos/`) rather than by technical layer (`models/`, `routes/`, `services/`). Each slice owns its models, schemas, repository, service, and router. A `shared/` package holds only cross-cutting infrastructure (error schemas, health router, DB session).

**Rationale**: Vertical slice architecture improves cohesion — all code for a feature is co-located and can be developed, tested, and reasoned about independently. This aligns with the spec's independently-testable user stories. The single `todos/` slice for this feature naturally maps to User Stories 1–4.

**Project layout**:
```
todo_api/
├── main.py              ← app factory, lifespan, global exception handlers
├── config.py            ← BaseSettings (DATABASE_URL, APP_ENV, LOG_LEVEL)
├── database.py          ← async engine, AsyncSession, get_db Depends
├── shared/
│   ├── exceptions.py    ← typed domain exceptions (TodoNotFoundError, etc.)
│   ├── error_schemas.py ← ErrorResponse Pydantic schema
│   └── health_router.py ← GET /health, GET /health/ready
└── todos/
    ├── models.py        ← SQLAlchemy ORM: Todo table
    ├── schemas.py       ← Pydantic: TodoCreate, TodoUpdate, TodoResponse
    ← repository.py      ← async DB queries (add, get, list, update, delete)
    ├── service.py       ← business rules, calls repository
    └── router.py        ← APIRouter wired via Depends

tests/
├── conftest.py          ← app fixture, async_client, override get_db
└── todos/
    ├── test_create_todo.py
    ├── test_read_todo.py
    ├── test_update_todo.py
    └── test_delete_todo.py
```

**Alternatives considered**:
- Layer-first layout (`models/`, `routes/`, `services/`) — rejected; scatter related code across many directories, harder to navigate.
- One file per feature — rejected; too condensed for testability and future growth.

---

### Decision 4: Todo Identifier — UUID vs. Auto-increment Integer

**Decision**: Use UUID v4 as the primary key, represented as a string in the SQLite column (SQLite has no native UUID type).

**Rationale**: UUID prevents enumeration attacks on the `/api/v1/todos/{id}` endpoint, is globally unique for future federation, and avoids leaking record counts. FastAPI's Pydantic models serialize UUID as a string automatically.

**Alternatives considered**:
- Auto-increment integer — rejected; leaks cardinality, trivially enumerable.
- ULID — considered; better sortability but adds an extra dependency for minimal gain here.

**Implementation**: `uuid.uuid4()` generated in Python at model instantiation, stored as `VARCHAR(36)` in SQLite.

---

### Decision 5: Null Due Date Sorting

**Decision**: When sorting by `due_date` ascending **or** descending, rows with `NULL` due dates are placed at the **end** of the result set in both directions (as mandated by FR-009).

**Rationale**: SQL's default `NULLS LAST` / `NULLS FIRST` behaviour varies by database. SQLAlchemy 2.x supports `nulls_last()` and `nulls_first()` via `sqlalchemy.nulls_last(column.asc())`. We always use `nulls_last()` regardless of sort direction.

**Implementation**: Repository uses `sqlalchemy.nulls_last(Todo.due_date.asc())` for ascending and `sqlalchemy.nulls_last(Todo.due_date.desc())` for descending.

---

### Decision 6: Partial Update Strategy — PATCH with Optional Fields

**Decision**: Use `PATCH` (not `PUT`) for updates. The `TodoUpdate` Pydantic schema has all fields optional. Only fields explicitly provided in the request body are applied.

**Rationale**: `PUT` semantics require the full resource to be sent, requiring clients to fetch-then-write. `PATCH` with optional Pydantic fields is idiomatic for FastAPI partial updates.

**Implementation**: `model.model_dump(exclude_unset=True)` in the service layer to extract only provided fields.

---

### Decision 7: Filter Parameter Convention

**Decision**: Filtering by status uses the query parameter `status` with values `complete`, `incomplete`. Omitting the parameter returns all todos. Sorting uses `sort_by=due_date` and `order=asc|desc` per the constitution's naming convention.

**Rationale**: Aligns with Principle I of the constitution ("Pagination, filtering, and sorting parameters MUST follow a consistent convention: `page`, `page_size`, `sort_by`, `order`"). We extend with a `status` filter that is domain-specific.

---

### Decision 8: Pagination — Explicitly Out of Scope

**Decision**: Pagination is **not implemented** in this version, per the spec assumption ("Pagination is out of scope for this version; all matching todos are returned in a single response").

**Constitution alignment**: The constitution mandates consistent `page`/`page_size` conventions on list endpoints. This is a justified, documented scope reduction. The list endpoint must be structurally prepared to accept `page`/`page_size` in a future version (parameters reserved but unused).

---

### Decision 9: Database Initialisation Strategy

**Decision**: Use SQLAlchemy's `Base.metadata.create_all()` via the FastAPI lifespan context manager (`@asynccontextmanager`) to create tables on startup. No Alembic migrations for this version.

**Rationale**: Alembic adds complexity not required for an initial implementation against a local SQLite file. The spec does not mention migration requirements. For a production deployment, Alembic would be added as a follow-on.

---

### Decision 10: Settings Management

**Decision**: Application settings (`DATABASE_URL`, `APP_ENV`, `LOG_LEVEL`) are managed via a `pydantic_settings.BaseSettings` subclass reading from environment variables and a `.env` file.

**Rationale**: Mandated by Principle IV of the constitution. Default `DATABASE_URL` is `sqlite+aiosqlite:///./todo.db` (local file). Tests override this to `sqlite+aiosqlite:///:memory:` via `get_db` dependency override.

**Dependencies**: `pydantic-settings>=2.0`

---

## Technology Stack Summary

| Concern | Technology | Version |
|---------|-----------|---------|
| Web framework | FastAPI | ≥ 0.111 |
| ASGI server | Uvicorn | ≥ 0.30 |
| ORM | SQLAlchemy (async) | ≥ 2.0 |
| DB driver | aiosqlite | ≥ 0.19 |
| Database | SQLite | (bundled) |
| Data validation | Pydantic v2 | ≥ 2.7 |
| Settings | pydantic-settings | ≥ 2.0 |
| HTTP client (tests) | httpx | ≥ 0.27 |
| Test runner | pytest | ≥ 8.0 |
| Async test support | pytest-asyncio | ≥ 0.23 |
| Runtime | Python | 3.11+ |
