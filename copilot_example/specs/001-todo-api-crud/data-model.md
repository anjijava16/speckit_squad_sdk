# Data Model: TODO API CRUD

**Feature**: 001-todo-api-crud
**Phase**: 1 — Design
**Date**: 2026-05-20

---

## Entities

### Todo

Represents a single task item managed through the API.

#### Fields

| Field | Type | Nullable | Default | Notes |
|-------|------|----------|---------|-------|
| `id` | `VARCHAR(36)` | No | `uuid4()` on create | Primary key. UUID v4 string. Immutable after creation. |
| `title` | `VARCHAR(255)` | No | — | Required. Must be a non-empty string. |
| `description` | `TEXT` | Yes | `NULL` | Optional free-text description. |
| `due_date` | `DATE` | Yes | `NULL` | Optional target completion date. ISO 8601 date (`YYYY-MM-DD`). |
| `is_complete` | `BOOLEAN` | No | `false` | Completion status. `true` = complete, `false` = incomplete. |
| `created_at` | `DATETIME` | No | `utcnow()` on create | Immutable after creation. UTC. |
| `updated_at` | `DATETIME` | No | `utcnow()` on create | Updated on every successful PATCH. UTC. |

#### Constraints

- `id` — primary key, auto-assigned, immutable.
- `title` — NOT NULL, minimum length 1 character (enforced at application layer via Pydantic).
- `is_complete` — NOT NULL, default `false`.
- `created_at` — NOT NULL, set once at creation, never updated.
- `updated_at` — NOT NULL, refreshed on every `PATCH` operation.

#### Indexes

| Index | Columns | Purpose |
|-------|---------|---------|
| Primary key | `id` | Single-item lookup (FR-005) |
| `ix_todo_is_complete` | `is_complete` | Filter by status (FR-007) |
| `ix_todo_due_date` | `due_date` | Sort by due date (FR-008) |

---

## SQLAlchemy ORM Model (reference)

```python
import uuid
from datetime import date, datetime
from sqlalchemy import Boolean, Date, DateTime, String, Text, func
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

class Base(DeclarativeBase):
    pass

class Todo(Base):
    __tablename__ = "todos"

    id: Mapped[str] = mapped_column(
        String(36), primary_key=True, default=lambda: str(uuid.uuid4())
    )
    title: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    due_date: Mapped[date | None] = mapped_column(Date, nullable=True, index=True)
    is_complete: Mapped[bool] = mapped_column(Boolean, nullable=False, default=False, index=True)
    created_at: Mapped[datetime] = mapped_column(
        DateTime, nullable=False, default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime, nullable=False, default=func.now(), onupdate=func.now()
    )
```

---

## Pydantic Schemas (reference)

### TodoCreate — request body for POST

```python
from datetime import date
from pydantic import BaseModel, Field

class TodoCreate(BaseModel):
    title: str = Field(..., min_length=1, max_length=255)
    description: str | None = None
    due_date: date | None = None
```

### TodoUpdate — request body for PATCH (all fields optional)

```python
class TodoUpdate(BaseModel):
    title: str | None = Field(default=None, min_length=1, max_length=255)
    description: str | None = None
    due_date: date | None = None
    is_complete: bool | None = None
```

### TodoResponse — response body for all read operations

```python
from datetime import date, datetime
from pydantic import BaseModel

class TodoResponse(BaseModel):
    id: str
    title: str
    description: str | None
    due_date: date | None
    is_complete: bool
    created_at: datetime
    updated_at: datetime

    model_config = {"from_attributes": True}
```

---

## Validation Rules

| Rule | Field | Enforced By |
|------|-------|-------------|
| Title must not be empty or absent on create | `title` | Pydantic `min_length=1` on `TodoCreate` |
| Title must not be set to empty on update | `title` | Pydantic `min_length=1` on `TodoUpdate` (when provided) |
| Due date must be a valid ISO 8601 date | `due_date` | Pydantic `date` type coercion |
| `is_complete` must be a boolean | `is_complete` | Pydantic `bool` type coercion |
| `id` must not be provided by client on create | `id` | Not present in `TodoCreate` schema |
| `created_at` / `updated_at` must not be provided by client | timestamps | Not present in `TodoCreate` / `TodoUpdate` schemas |

---

## State Transitions

```
[Created]  is_complete = false
    │
    ├──── PATCH is_complete=true ───▶  [Complete]  is_complete = true
    │                                       │
    └──────────────────────────────────────── PATCH is_complete=false ──▶ [Incomplete]
    │
    └──── DELETE ───▶  [Deleted]  (record removed, permanently non-retrievable)
```

---

## Entity Relationships

The current version has a single entity (`Todo`) with no foreign-key relationships. The data model is intentionally flat to match the scope defined in the spec (no users, no categories, no tags).

---

## SQLite Schema (DDL)

```sql
CREATE TABLE todos (
    id          VARCHAR(36)  NOT NULL PRIMARY KEY,
    title       VARCHAR(255) NOT NULL,
    description TEXT,
    due_date    DATE,
    is_complete BOOLEAN      NOT NULL DEFAULT 0,
    created_at  DATETIME     NOT NULL,
    updated_at  DATETIME     NOT NULL
);

CREATE INDEX ix_todo_is_complete ON todos (is_complete);
CREATE INDEX ix_todo_due_date    ON todos (due_date);
```
