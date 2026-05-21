<!--
SYNC IMPACT REPORT
==================
Version change: (none) → 1.0.0 (initial ratification)
Modified principles: N/A — initial constitution
Added sections:
  - Core Principles (I–V)
  - Technology Stack & Standards
  - Development Workflow & Quality Gates
  - Governance
Removed sections: N/A
Templates requiring updates:
  ✅ .specify/templates/plan-template.md — Constitution Check gates now derivable from principles below
  ✅ .specify/templates/spec-template.md — User stories and acceptance scenarios align with REST + pytest focus
  ✅ .specify/templates/tasks-template.md — Task categories cover FastAPI setup, error handling, and pytest suites
Deferred TODOs: none
-->

# Python FastAPI REST API Constitution

## Core Principles

### I. Clean REST API Design

Every endpoint MUST follow RESTful conventions without exception:

- Resource URLs MUST use nouns, never verbs (e.g., `/users/{id}`, not `/getUser`).
- HTTP methods MUST be used semantically: `GET` for reads, `POST` for creation,
  `PUT`/`PATCH` for updates, `DELETE` for removal.
- HTTP status codes MUST be accurate: `200 OK`, `201 Created`, `204 No Content`,
  `400 Bad Request`, `404 Not Found`, `422 Unprocessable Entity`, `500 Internal Server Error`.
- All request/response bodies MUST be modelled with Pydantic schemas; raw `dict`
  types are prohibited in endpoint signatures.
- API paths MUST be versioned under a prefix (e.g., `/api/v1/`) from day one.
- Pagination, filtering, and sorting parameters MUST follow a consistent convention
  across all list endpoints (`page`, `page_size`, `sort_by`, `order`).

### II. Comprehensive Error Handling

Errors are first-class citizens and MUST be handled uniformly:

- Every FastAPI application MUST register global exception handlers for at least:
  `RequestValidationError`, `HTTPException`, and a catch-all `Exception` handler.
- All error responses MUST conform to a shared `ErrorResponse` Pydantic schema
  containing `status_code`, `error`, `message`, and an optional `details` field.
- Business-logic errors MUST be raised as typed custom exceptions that map to
  specific HTTP status codes via the global handler; naked `raise Exception(...)` is
  prohibited in route handlers.
- Validation errors from Pydantic MUST surface field-level detail in `details`
  so clients can identify exactly which fields failed.
- Unhandled exceptions MUST be logged with full tracebacks before returning a
  generic `500` response; stack traces MUST NOT be exposed in API responses.

### III. pytest-First Testing (NON-NEGOTIABLE)

Test-driven development with pytest is mandatory for all route and service code:

- Tests MUST be written and approved before any implementation code is committed
  (Red → Green → Refactor cycle strictly enforced).
- The test suite MUST use `pytest` with `httpx.AsyncClient` + FastAPI's
  `TestClient` (or `anyio`/`pytest-asyncio`) for end-to-end route testing.
- Every endpoint MUST have: a happy-path test, at least one validation-error test,
  and at least one not-found / business-error test.
- Fixtures MUST be defined in `conftest.py`; test helpers MUST NOT be duplicated
  across test files.
- Test coverage MUST remain at or above **90 %** for `src/`; coverage gates are
  enforced in CI and block merges when violated.
- Parametrized tests (`@pytest.mark.parametrize`) MUST be used wherever multiple
  input variants share the same assertion logic.

### IV. FastAPI Framework Standards

FastAPI's built-in capabilities MUST be leveraged before reaching for third-party
libraries:

- Dependency injection (`Depends`) MUST be used for: database sessions, auth
  context, and shared service instances — never instantiated inline in route bodies.
- All application settings MUST be managed via a `pydantic_settings.BaseSettings`
  class loaded from environment variables; hard-coded config values are prohibited.
- OpenAPI documentation MUST be kept accurate and complete: every endpoint MUST
  declare `summary`, `response_model`, and `responses` for all documented status
  codes.
- Background tasks MUST use FastAPI's `BackgroundTasks` or a dedicated worker
  queue; blocking I/O MUST NOT be called from async route handlers.
- Lifespan events (`@asynccontextmanager` lifespan) MUST be used for startup/
  shutdown resource management (DB pools, caches, etc.).

### V. API Versioning & Observability

The API MUST be designed for evolution and production visibility:

- All public routes MUST live under a versioned router prefix (`/api/v1`, `/api/v2`).
  Introducing a breaking change MUST result in a new version prefix, never an
  in-place modification of an existing versioned route.
- Structured logging MUST be emitted for every request/response cycle including
  method, path, status code, and latency; log lines MUST be JSON-formatted in
  non-development environments.
- Health-check endpoints (`GET /health` and `GET /health/ready`) MUST exist,
  return `200` when healthy, and MUST NOT require authentication.
- Deprecation of a versioned endpoint MUST be signalled via a `Deprecation` response
  header for at least one minor release cycle before removal.

## Technology Stack & Standards

This constitution governs projects built with the following canonical stack:

- **Language**: Python 3.11+
- **Web Framework**: FastAPI (latest stable)
- **Data Validation**: Pydantic v2
- **Settings Management**: `pydantic-settings`
- **ASGI Server**: Uvicorn (development), Gunicorn + Uvicorn workers (production)
- **Testing**: pytest, pytest-asyncio, httpx, pytest-cov
- **Linting / Formatting**: Ruff (lint + format), mypy (strict type checking)
- **Dependency Management**: `uv` or `pip` with `pyproject.toml`

All third-party dependencies MUST be pinned to minor versions in `pyproject.toml`
and reviewed for security advisories before adoption.

## Development Workflow & Quality Gates

### Branching & Review

- All work MUST occur on feature branches following the naming convention
  `###-short-description` (e.g., `001-user-auth`).
- Pull requests MUST pass all CI checks (lint, type-check, tests, coverage) before
  merging; bypassing CI gates requires written justification in the PR description.

### Quality Gates (ordered — each MUST pass before the next phase begins)

1. **Spec gate**: Feature spec reviewed and acceptance scenarios written.
2. **Test gate**: pytest test stubs written, reviewed, and confirmed failing (red).
3. **Implementation gate**: All tests pass (green); coverage ≥ 90 %.
4. **Type gate**: `mypy --strict` reports zero errors on `src/`.
5. **Lint gate**: `ruff check` and `ruff format --check` report zero violations.
6. **OpenAPI gate**: New or changed endpoints have complete OpenAPI annotations.
7. **Review gate**: At least one peer review approval on the pull request.

### Constitution Check (for plan.md)

Every implementation plan MUST verify the following before Phase 0 research:

- [ ] Endpoints follow REST resource naming and correct HTTP verbs.
- [ ] `ErrorResponse` schema is defined or extended (not bypassed).
- [ ] pytest tests are scoped before implementation tasks begin.
- [ ] All new settings use `BaseSettings`; no hard-coded config.
- [ ] OpenAPI annotations are included in the task list.

## Governance

This constitution supersedes all other project practices and coding guidelines.
Amendments MUST:

1. Be proposed as a pull request modifying this file with a written rationale.
2. Receive approval from at least one project maintainer.
3. Include a migration plan if any existing code violates the amended principle.
4. Increment `CONSTITUTION_VERSION` according to semantic versioning:
   - **MAJOR** — removal or redefinition of a principle.
   - **MINOR** — addition of a new principle or materially expanded guidance.
   - **PATCH** — clarifications, wording improvements, typo fixes.

All PRs and code reviews MUST verify compliance with this constitution.
Complexity beyond what principles require MUST be justified in writing.

**Version**: 1.0.0 | **Ratified**: 2026-05-20 | **Last Amended**: 2026-05-20
