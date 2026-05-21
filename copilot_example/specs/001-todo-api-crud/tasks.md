# Tasks: TODO API CRUD

**Input**: Design documents from `specs/001-todo-api-crud/`

**Branch**: `001-todo-api-crud` | **Date**: 2026-05-20

**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/todo-api.md ✅, quickstart.md ✅

**User Stories**:
- **US1** — Create a New Todo Item (P1) 🎯 MVP
- **US2** — Retrieve and Browse Todo Items (P1)
- **US3** — Update an Existing Todo Item (P2)
- **US4** — Delete a Todo Item (P2)

---

## Format: `[ID] [P?] [Story?] Description`

- **[P]**: Can run in parallel (different files, no blocking dependencies)
- **[Story]**: Which user story this task belongs to (`[US1]`, `[US2]`, `[US3]`, `[US4]`)
- Exact file paths are included in every task description

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Initialize the project skeleton, package manifests, and config files. No application logic — just the scaffolding needed for everything that follows.

- [ ] T001 Create `pyproject.toml` with project metadata, `[tool.pytest.ini_options]` (`asyncio_mode = "auto"`, `addopts = "--cov=todo_api --cov-fail-under=90"`)
- [ ] T002 Create `requirements.txt` with pinned dependencies: `fastapi>=0.111`, `uvicorn[standard]`, `sqlalchemy[asyncio]>=2.0`, `aiosqlite>=0.19`, `pydantic>=2.7`, `pydantic-settings>=2.0`, `httpx>=0.27`, `pytest>=8.0`, `pytest-asyncio>=0.23`, `pytest-cov`
- [ ] T003 [P] Create `.env.example` with `DATABASE_URL=sqlite+aiosqlite:///./todo.db`, `APP_ENV=development`, `LOG_LEVEL=INFO`
- [ ] T004 [P] Create `.gitignore` entries: `todo.db`, `.venv/`, `__pycache__/`, `.env`, `.coverage`, `htmlcov/`, `*.pyc`, `dist/`
- [ ] T005 [P] Create `README.md` with single sentence and pointer to `specs/001-todo-api-crud/quickstart.md`
- [ ] T006 Create all `__init__.py` package files: `todo_api/__init__.py`, `todo_api/shared/__init__.py`, `todo_api/todos/__init__.py`, `tests/__init__.py`, `tests/todos/__init__.py`

**Checkpoint**: Project skeleton in place — all package imports will resolve

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that every user story depends on. No user story work can begin until this phase is complete.

**⚠️ CRITICAL**: All US1–US4 implementation tasks block on this phase.

- [ ] T007 [P] Create `todo_api/config.py` — `Settings(BaseSettings)` with fields `DATABASE_URL: str`, `APP_ENV: str = "development"`, `LOG_LEVEL: str = "INFO"`; expose a module-level `settings = Settings()` singleton
- [ ] T008 Create `todo_api/database.py` — `create_async_engine(settings.DATABASE_URL)`, `async_sessionmaker(engine, expire_on_commit=False)`, `async def get_db()` generator yielding `AsyncSession` for `Depends` injection
- [ ] T009 [P] Create `todo_api/shared/error_schemas.py` — `ErrorResponse(BaseModel)` with `status_code: int`, `error: str`, `message: str`, `details: list[dict] | None = None`
- [ ] T010 [P] Create `todo_api/shared/exceptions.py` — `TodoNotFoundError(Exception)` carrying `todo_id: str` attribute; used by service layer to signal missing resources
- [ ] T011 Create `todo_api/shared/health_router.py` — `APIRouter(tags=["health"])`; `GET /health` → `{"status": "ok"}` 200; `GET /health/ready` → `{"status": "ready"}` 200 or `{"status": "unavailable"}` 503 if DB unreachable
- [ ] T012 Create `todo_api/todos/models.py` — `Todo(Base)` SQLAlchemy 2.x ORM model with all 7 fields (`id` VARCHAR(36) UUID PK, `title` VARCHAR(255) NOT NULL, `description` TEXT nullable, `due_date` DATE nullable index, `is_complete` BOOLEAN NOT NULL default False index, `created_at` DATETIME NOT NULL, `updated_at` DATETIME NOT NULL onupdate); import `Base` from `todo_api/database.py`
- [ ] T013 [P] Create `todo_api/todos/schemas.py` — `TodoCreate` (title required min_length=1, description nullable, due_date date nullable), `TodoUpdate` (all fields optional, title min_length=1 when provided), `TodoResponse` (all 7 fields, `model_config = {"from_attributes": True}`)
- [ ] T014 Create `todo_api/main.py` — `@asynccontextmanager lifespan`: call `Base.metadata.create_all(engine)`; `FastAPI(lifespan=lifespan, title="TODO API", version="1.0.0")`; global exception handlers for `RequestValidationError` → 422 `ErrorResponse`, `HTTPException` → `ErrorResponse`, bare `Exception` → 500 `ErrorResponse`; include `health_router` and `todos_router`
- [ ] T015 [P] Create `tests/conftest.py` — `async_engine` fixture using `sqlite+aiosqlite:///:memory:`; `create_all`/`drop_all` around test session; `AsyncSession` fixture; `app.dependency_overrides[get_db]` override; `AsyncClient(app=app, base_url="http://test")` fixture

**Checkpoint**: Foundation complete — all four user story phases can now begin

---

## Phase 3: User Story 1 — Create a New Todo Item (Priority: P1) 🎯 MVP

**Goal**: `POST /api/v1/todos` accepts title + optional fields, persists the todo, returns 201 with `TodoResponse` including system-generated `id`, `is_complete=false`, and timestamps.

**Independent Test**: `POST /api/v1/todos` with valid body → 201 response with unique `id` and `is_complete=false`. Missing title → 422. Invalid date format → 422. All four US-1 acceptance scenarios pass.

### Implementation

- [ ] T016 [US1] Add `async def add(session, todo: Todo) -> Todo` to `todo_api/todos/repository.py` — `session.add(todo)`, `await session.commit()`, `await session.refresh(todo)`, return todo
- [ ] T017 [US1] Add `async def create_todo(session, data: TodoCreate) -> TodoResponse` to `todo_api/todos/service.py` — generate `str(uuid.uuid4())` for `id`, construct `Todo(**data.model_dump(), id=id, created_at=utcnow, updated_at=utcnow)`, call `repository.add`, return `TodoResponse.model_validate(todo)`
- [ ] T018 [US1] Create `todo_api/todos/router.py` with `APIRouter(prefix="/api/v1/todos", tags=["todos"])`; implement `POST /` → `create_todo`, `status_code=201`, `response_model=TodoResponse`, `responses={422: {"model": ErrorResponse}}`

### Tests

- [ ] T019 [P] [US1] Write `tests/todos/test_create_todo.py` covering: valid title→201+UUID+`is_complete=false`; title+desc+due_date all persisted→201; missing title body→422 with field detail; `due_date="not-a-date"`→422 with field detail

**Checkpoint**: US1 fully functional and independently testable. MVP deliverable.

---

## Phase 4: User Story 2 — Retrieve and Browse Todo Items (Priority: P1)

**Goal**: `GET /api/v1/todos` returns all todos with optional `?status=` filter and `?sort_by=due_date&order=` sort (nulls always last). `GET /api/v1/todos/{id}` returns a single todo or 404.

**Independent Test**: Create 3 todos (1 complete, 1 incomplete with due_date, 1 incomplete without due_date). `GET /api/v1/todos?status=incomplete` returns 2. `GET /api/v1/todos?sort_by=due_date&order=asc` puts null-due_date todo last. `GET /api/v1/todos/{id}` returns correct todo. `GET /api/v1/todos/nonexistent-id` → 404.

### Implementation

- [ ] T020 [US2] Add `async def get_by_id(session, todo_id: str) -> Todo | None` to `todo_api/todos/repository.py` — `await session.get(Todo, todo_id)`
- [ ] T021 [US2] Add `async def list_all(session, status: str | None, sort_by: str | None, order: str) -> list[Todo]` to `todo_api/todos/repository.py` — build `select(Todo)` query; apply `where(Todo.is_complete == ...)` if status set; apply `order_by(nulls_last(asc(Todo.due_date)))` or `order_by(nulls_last(desc(Todo.due_date)))` if `sort_by="due_date"`; default order `Todo.created_at.desc()`; `await session.execute(stmt)`, return `scalars().all()`
- [ ] T022 [US2] Add `async def get_todo(session, todo_id: str) -> TodoResponse` to `todo_api/todos/service.py` — call `repository.get_by_id`; raise `TodoNotFoundError(todo_id)` if None; return `TodoResponse.model_validate(todo)`
- [ ] T023 [US2] Add `async def list_todos(session, status, sort_by, order) -> list[TodoResponse]` to `todo_api/todos/service.py` — call `repository.list_all`; return `[TodoResponse.model_validate(t) for t in todos]`
- [ ] T024 [US2] Add `GET /` and `GET /{id}` to `todo_api/todos/router.py` — list endpoint with `status: str | None = None`, `sort_by: str | None = None`, `order: str = "asc"` query params; single endpoint with path `{todo_id}` → 200/404; `responses={404: {"model": ErrorResponse}}`

### Tests

- [ ] T025 [P] [US2] Write `tests/todos/test_read_todo.py` covering: list all returns all; `?status=complete` returns only complete; `?status=incomplete` returns only incomplete; `?sort_by=due_date&order=asc` nulls last; `?sort_by=due_date&order=desc` nulls last; get by valid id→200; get by nonexistent id→404; empty list returns `[]`

**Checkpoint**: US1 + US2 both independently functional. Full read/write API now usable.

---

## Phase 5: User Story 3 — Update an Existing Todo Item (Priority: P2)

**Goal**: `PATCH /api/v1/todos/{id}` applies partial updates (any combination of title, description, due_date, is_complete). Returns 200 with updated `TodoResponse` including refreshed `updated_at`. Returns 404 for unknown id. Returns 422 for empty title.

**Independent Test**: Create todo → PATCH title → verify new title + updated `updated_at`. PATCH `is_complete=true` → verify complete. PATCH `is_complete=false` → verify reverted. PATCH unknown id → 404. PATCH `{"title":""}` → 422.

### Implementation

- [ ] T026 [US3] Add `async def update(session, todo_id: str, updates: dict) -> Todo | None` to `todo_api/todos/repository.py` — `await session.get(Todo, todo_id)`; if None return None; apply `setattr(todo, k, v)` for each key in updates; `await session.commit()`, `await session.refresh(todo)`, return todo
- [ ] T027 [US3] Add `async def update_todo(session, todo_id: str, data: TodoUpdate) -> TodoResponse` to `todo_api/todos/service.py` — call `data.model_dump(exclude_unset=True)` to get only provided fields; call `repository.update(session, todo_id, updates)`; raise `TodoNotFoundError(todo_id)` if None; return `TodoResponse.model_validate(todo)`
- [ ] T028 [US3] Add `PATCH /{todo_id}` to `todo_api/todos/router.py` — `response_model=TodoResponse`, `responses={404: {"model": ErrorResponse}, 422: {"model": ErrorResponse}}`

### Tests

- [ ] T029 [P] [US3] Write `tests/todos/test_update_todo.py` covering: update title→200+new title+updated `updated_at`; mark complete→200+`is_complete=true`; revert to incomplete→200+`is_complete=false`; PATCH nonexistent id→404; PATCH `{"title":""}` →422 (min_length violation)

**Checkpoint**: US1 + US2 + US3 all independently functional.

---

## Phase 6: User Story 4 — Delete a Todo Item (Priority: P2)

**Goal**: `DELETE /api/v1/todos/{id}` permanently removes the todo and returns 204 No Content. Returns 404 for unknown id.

**Independent Test**: Create todo → DELETE → 204 → GET by id → 404. DELETE nonexistent id → 404.

### Implementation

- [ ] T030 [US4] Add `async def delete(session, todo_id: str) -> bool` to `todo_api/todos/repository.py` — `await session.get(Todo, todo_id)`; if None return False; `await session.delete(todo)`; `await session.commit()`; return True
- [ ] T031 [US4] Add `async def delete_todo(session, todo_id: str) -> None` to `todo_api/todos/service.py` — call `repository.delete`; raise `TodoNotFoundError(todo_id)` if returns False
- [ ] T032 [US4] Add `DELETE /{todo_id}` to `todo_api/todos/router.py` — `status_code=204`, `response_model=None`, `responses={404: {"model": ErrorResponse}}`

### Tests

- [ ] T033 [P] [US4] Write `tests/todos/test_delete_todo.py` covering: delete existing todo→204+no body; GET deleted todo→404; delete nonexistent id→404

**Checkpoint**: All four user stories fully functional. Complete CRUD API delivered.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Final hardening, observability, and validation that touch multiple slices.

- [ ] T034 [P] Add structured logging in `todo_api/main.py` — configure Python `logging` with JSON formatter when `APP_ENV != "development"`; log startup, request errors, and 500-level exceptions with `exc_info=True`
- [ ] T035 [P] Add OpenAPI metadata to all routes in `todo_api/todos/router.py` and `todo_api/shared/health_router.py` — `summary`, `description`, correct `responses` dict on every operation
- [ ] T036 Run full pytest suite and confirm coverage gate passes: `pytest --cov=todo_api --cov-report=term-missing --cov-fail-under=90`
- [ ] T037 [P] Validate running server against `specs/001-todo-api-crud/quickstart.md` curl examples — all five operations return expected status codes and response bodies

---

## Dependencies & Execution Order

### Phase Dependencies

```
Phase 1 (Setup)         → no dependencies, start immediately
Phase 2 (Foundational)  → depends on Phase 1 ▶ BLOCKS all user story phases
Phase 3 (US1)           → depends on Phase 2
Phase 4 (US2)           → depends on Phase 2 (can run in parallel with Phase 3 once P2 done)
Phase 5 (US3)           → depends on Phase 2 (can run in parallel with P3/P4)
Phase 6 (US4)           → depends on Phase 2 (can run in parallel with P3/P4/P5)
Phase 7 (Polish)        → depends on all user story phases
```

### User Story Dependencies

| Story | Depends On | Independent? |
|-------|-----------|-------------|
| US1 (Create) | Foundational | ✅ Yes |
| US2 (Read/Browse) | Foundational | ✅ Yes |
| US3 (Update) | Foundational | ✅ Yes — no hard dependency on US1/US2 code paths |
| US4 (Delete) | Foundational | ✅ Yes — no hard dependency on US1/US2/US3 code paths |

**Note on file sharing**: `repository.py`, `service.py`, and `router.py` are extended incrementally across US1–US4. When working in parallel, each developer adds new methods/routes to these files without modifying existing ones.

### Within Each User Story

1. Repository method → Service method → Router endpoint (sequential within story)
2. Tests can be written alongside or after — they do not block implementation

### Parallel Opportunities

- T003, T004, T005 — independent scaffolding files (Phase 1)
- T007, T009, T010, T013, T015 — independent foundational files (Phase 2)
- T019, T025, T029, T033 — test files are each independent of other test files
- T034, T035, T037 — polish tasks operate on different files

---

## Parallel Example: US1 + US2 (after Phase 2 complete)

```
Developer A (US1):                  Developer B (US2):
  T016 repository.add()               T020 repository.get_by_id()
  T017 service.create_todo()          T021 repository.list_all()   [P with T020]
  T018 router POST /                  T022 service.get_todo()
  T019 test_create_todo.py [P]        T023 service.list_todos()    [P with T022]
                                      T024 router GET / and /{id}
                                      T025 test_read_todo.py [P]
```

Both stories are independently testable at their checkpoints.

---

## Implementation Strategy

### MVP First (US1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (**CRITICAL** — blocks everything)
3. Complete Phase 3: US1
4. **STOP and VALIDATE**: `POST /api/v1/todos` → 201, `pytest tests/todos/test_create_todo.py`
5. Deploy or demo if ready

### Incremental Delivery

| Milestone | Phases | What You Get |
|-----------|--------|-------------|
| MVP | 1 + 2 + 3 | Create todo. Immediately useful as a data-entry API. |
| v0.2 | + 4 | Read + browse + filter + sort. API is fully consumable. |
| v0.3 | + 5 | Update todos. Users can mark tasks done. |
| v1.0 | + 6 + 7 | Delete + polish. Full CRUD with ≥90% test coverage. |

---

## Notes

- `[P]` = different files, no incomplete dependencies → safe to parallelize
- `[US?]` label maps each task to its user story for traceability
- Each user story phase ends with an independent checkpoint — stop and validate there
- `repository.py`, `service.py`, `router.py` are extended per story; never modify completed methods
- The `TodoNotFoundError` global handler (wired in `main.py`) means every 404 path in service just raises — no per-route try/except needed
- SQLAlchemy `nulls_last()` must be imported from `sqlalchemy` and applied to **both** asc and desc sort directions to satisfy FR-009
