# Quickstart: TODO API

**Feature**: 001-todo-api-crud
**Phase**: 1 — Design
**Date**: 2026-05-20

This guide gets you from zero to a running TODO API with passing tests in under 5 minutes.

---

## Prerequisites

- Python 3.11 or later
- `pip` (comes with Python)

---

## 1. Set Up the Project

```bash
# From the repository root
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

pip install -r requirements.txt
```

`requirements.txt` (generated during implementation):

```
fastapi>=0.111
uvicorn[standard]>=0.30
sqlalchemy[asyncio]>=2.0
aiosqlite>=0.19
pydantic>=2.7
pydantic-settings>=2.0
httpx>=0.27
pytest>=8.0
pytest-asyncio>=0.23
```

---

## 2. Configure Environment

Create a `.env` file in the project root (copy from `.env.example`):

```dotenv
# .env
DATABASE_URL=sqlite+aiosqlite:///./todo.db
APP_ENV=development
LOG_LEVEL=INFO
```

The application reads settings from `todo_api/config.py` via `pydantic-settings`.
Environment variables override `.env` values.

---

## 3. Run the API

```bash
uvicorn todo_api.main:app --reload
```

The server starts at **http://localhost:8000**.

| URL | Purpose |
|-----|---------|
| http://localhost:8000/docs | Interactive Swagger UI |
| http://localhost:8000/openapi.json | OpenAPI schema |
| http://localhost:8000/health | Liveness check |
| http://localhost:8000/health/ready | Readiness check |

The SQLite database file `todo.db` is created automatically on first startup.

---

## 4. Try It Out

### Create a todo

```bash
curl -s -X POST http://localhost:8000/api/v1/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "Buy groceries", "description": "Milk and eggs", "due_date": "2026-05-25"}' \
  | python -m json.tool
```

### List all todos

```bash
curl -s http://localhost:8000/api/v1/todos | python -m json.tool
```

### Filter by status

```bash
# Incomplete only
curl -s "http://localhost:8000/api/v1/todos?status=incomplete" | python -m json.tool

# Complete only
curl -s "http://localhost:8000/api/v1/todos?status=complete" | python -m json.tool
```

### Sort by due date

```bash
curl -s "http://localhost:8000/api/v1/todos?sort_by=due_date&order=asc" | python -m json.tool
```

### Get a single todo

```bash
# Replace <id> with the UUID returned from the create call
curl -s http://localhost:8000/api/v1/todos/<id> | python -m json.tool
```

### Mark a todo complete

```bash
curl -s -X PATCH http://localhost:8000/api/v1/todos/<id> \
  -H "Content-Type: application/json" \
  -d '{"is_complete": true}' \
  | python -m json.tool
```

### Delete a todo

```bash
curl -s -X DELETE http://localhost:8000/api/v1/todos/<id> -w "%{http_code}\n"
# Expect: 204
```

---

## 5. Run Tests

Tests use an **in-memory SQLite database** and never touch `todo.db`.

```bash
# Run all tests
pytest

# Run with verbose output
pytest -v

# Run with coverage report
pytest --cov=todo_api --cov-report=term-missing

# Run a specific test file
pytest tests/todos/test_create_todo.py -v
```

**Expected output** (all tests passing):

```
tests/todos/test_create_todo.py::test_create_todo_success PASSED
tests/todos/test_create_todo.py::test_create_todo_missing_title PASSED
tests/todos/test_create_todo.py::test_create_todo_invalid_date PASSED
tests/todos/test_read_todo.py::test_list_all_todos PASSED
tests/todos/test_read_todo.py::test_filter_by_complete PASSED
tests/todos/test_read_todo.py::test_filter_by_incomplete PASSED
tests/todos/test_read_todo.py::test_sort_by_due_date_asc PASSED
tests/todos/test_read_todo.py::test_sort_by_due_date_desc PASSED
tests/todos/test_read_todo.py::test_nulls_last_ascending PASSED
tests/todos/test_read_todo.py::test_nulls_last_descending PASSED
tests/todos/test_read_todo.py::test_get_single_todo PASSED
tests/todos/test_read_todo.py::test_get_nonexistent_todo_returns_404 PASSED
tests/todos/test_update_todo.py::test_update_title PASSED
tests/todos/test_update_todo.py::test_mark_complete PASSED
tests/todos/test_update_todo.py::test_mark_incomplete PASSED
tests/todos/test_update_todo.py::test_update_nonexistent_returns_404 PASSED
tests/todos/test_update_todo.py::test_update_empty_title_returns_422 PASSED
tests/todos/test_delete_todo.py::test_delete_todo_success PASSED
tests/todos/test_delete_todo.py::test_delete_nonexistent_returns_404 PASSED

19 passed in 0.42s
```

Coverage must remain at or above **90 %** (enforced by the constitution).

---

## 6. Project Layout

```
todo_api/
├── main.py              ← FastAPI app factory, lifespan, global exception handlers
├── config.py            ← BaseSettings (reads from env / .env)
├── database.py          ← Async SQLAlchemy engine, AsyncSession, get_db Depends
├── shared/
│   ├── exceptions.py    ← TodoNotFoundError and other domain exceptions
│   ├── error_schemas.py ← Shared ErrorResponse Pydantic schema
│   └── health_router.py ← GET /health, GET /health/ready
└── todos/               ← Vertical slice (all todo concerns live here)
    ├── models.py        ← SQLAlchemy ORM model
    ├── schemas.py       ← Pydantic request/response schemas
    ├── repository.py    ← Async DB access (add, get, list, update, delete)
    ├── service.py       ← Business logic, calls repository
    └── router.py        ← APIRouter, wired via Depends

tests/
├── conftest.py          ← Shared fixtures: app, async_client, in-memory DB override
└── todos/
    ├── test_create_todo.py
    ├── test_read_todo.py
    ├── test_update_todo.py
    └── test_delete_todo.py

specs/001-todo-api-crud/
├── spec.md              ← Feature specification
├── plan.md              ← This implementation plan
├── research.md          ← Technical decisions
├── data-model.md        ← Entity definitions
├── quickstart.md        ← This file
├── contracts/
│   └── todo-api.md      ← Full API contract
└── tasks.md             ← (generated by /speckit.tasks — not yet created)
```

---

## 7. Troubleshooting

| Problem | Solution |
|---------|----------|
| `ModuleNotFoundError: todo_api` | Activate venv: `source .venv/bin/activate` |
| `sqlite3.OperationalError` on startup | Delete `todo.db` and restart; it will be recreated |
| Tests fail with `RuntimeError: no running event loop` | Ensure `pytest-asyncio` is installed and `asyncio_mode = "auto"` is in `pyproject.toml` |
| `422` on create when date looks correct | Use ISO 8601 format: `"2026-05-25"`, not `"May 25 2026"` |
| Coverage below 90 % | Run `pytest --cov=todo_api --cov-report=term-missing` to see uncovered lines |
