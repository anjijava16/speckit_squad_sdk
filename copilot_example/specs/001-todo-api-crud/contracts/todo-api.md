# API Contract: TODO REST API v1

**Feature**: 001-todo-api-crud
**Phase**: 1 — Design
**Date**: 2026-05-20
**Base URL**: `/api/v1`
**Content-Type**: `application/json` (all requests and responses)

---

## Global Conventions

### Request Headers

| Header | Required | Value |
|--------|----------|-------|
| `Content-Type` | Yes (POST/PATCH) | `application/json` |

### Error Response Schema

All error responses share a single `ErrorResponse` structure:

```json
{
  "status_code": 422,
  "error": "Validation Error",
  "message": "Request body is invalid.",
  "details": [
    {
      "field": "title",
      "message": "Field required"
    }
  ]
}
```

| Field | Type | Always Present | Description |
|-------|------|----------------|-------------|
| `status_code` | `integer` | Yes | Mirrors the HTTP status code |
| `error` | `string` | Yes | Short error category label |
| `message` | `string` | Yes | Human-readable description |
| `details` | `array` | No | Field-level detail for validation errors |

### HTTP Status Code Reference

| Code | Meaning | When Used |
|------|---------|-----------|
| `200 OK` | Success | GET (single or list), PATCH |
| `201 Created` | Resource created | POST |
| `204 No Content` | Success, no body | DELETE |
| `400 Bad Request` | Malformed request | Body cannot be parsed |
| `404 Not Found` | Resource missing | ID does not exist |
| `422 Unprocessable Entity` | Validation failure | Field-level validation errors |
| `500 Internal Server Error` | Unexpected error | Unhandled server exception |

---

## Endpoints

---

### POST `/api/v1/todos`

Create a new todo item.

#### Request Body

```json
{
  "title": "Buy groceries",
  "description": "Milk, eggs, bread",
  "due_date": "2026-05-25"
}
```

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `title` | `string` | **Yes** | min length 1, max length 255 |
| `description` | `string \| null` | No | Free text |
| `due_date` | `string (date)` | No | ISO 8601 format: `YYYY-MM-DD` |

`is_complete` is **not** accepted in the create request — it always defaults to `false`.

#### Responses

**`201 Created`**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "title": "Buy groceries",
  "description": "Milk, eggs, bread",
  "due_date": "2026-05-25",
  "is_complete": false,
  "created_at": "2026-05-20T14:30:00Z",
  "updated_at": "2026-05-20T14:30:00Z"
}
```

**`422 Unprocessable Entity`** — title absent or empty
```json
{
  "status_code": 422,
  "error": "Validation Error",
  "message": "Request body is invalid.",
  "details": [{"field": "title", "message": "Field required"}]
}
```

**`422 Unprocessable Entity`** — invalid date format
```json
{
  "status_code": 422,
  "error": "Validation Error",
  "message": "Request body is invalid.",
  "details": [{"field": "due_date", "message": "Input should be a valid date"}]
}
```

---

### GET `/api/v1/todos`

Retrieve all todo items, with optional filtering and sorting.

#### Query Parameters

| Parameter | Type | Required | Values | Default | Description |
|-----------|------|----------|--------|---------|-------------|
| `status` | `string` | No | `complete`, `incomplete` | (all) | Filter by completion status |
| `sort_by` | `string` | No | `due_date` | (none — created_at desc) | Sort field |
| `order` | `string` | No | `asc`, `desc` | `asc` | Sort direction; only applied when `sort_by` is set |

**Note**: Todos without a `due_date` are placed at the end of the list regardless of `order` direction (FR-009).

**Note**: `page` and `page_size` parameters are reserved for future use but not active in this version. Including them has no effect.

#### Response

**`200 OK`** — array of TodoResponse objects (empty array when no todos match)

```json
[
  {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "title": "Buy groceries",
    "description": "Milk, eggs, bread",
    "due_date": "2026-05-25",
    "is_complete": false,
    "created_at": "2026-05-20T14:30:00Z",
    "updated_at": "2026-05-20T14:30:00Z"
  }
]
```

**Examples**:

| URL | Returns |
|-----|---------|
| `GET /api/v1/todos` | All todos, default order (created_at desc) |
| `GET /api/v1/todos?status=incomplete` | Only incomplete todos |
| `GET /api/v1/todos?status=complete` | Only complete todos |
| `GET /api/v1/todos?sort_by=due_date&order=asc` | All todos, earliest due first, nulls last |
| `GET /api/v1/todos?status=incomplete&sort_by=due_date&order=desc` | Incomplete, latest due first, nulls last |

---

### GET `/api/v1/todos/{id}`

Retrieve a single todo item by its unique identifier.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `string (UUID)` | Yes | The todo's unique identifier |

#### Responses

**`200 OK`**
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "title": "Buy groceries",
  "description": "Milk, eggs, bread",
  "due_date": "2026-05-25",
  "is_complete": false,
  "created_at": "2026-05-20T14:30:00Z",
  "updated_at": "2026-05-20T14:30:00Z"
}
```

**`404 Not Found`**
```json
{
  "status_code": 404,
  "error": "Not Found",
  "message": "Todo with id '3fa85f64-5717-4562-b3fc-2c963f66afa6' was not found."
}
```

---

### PATCH `/api/v1/todos/{id}`

Update one or more fields of an existing todo item. Only fields included in the request body are modified.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `string (UUID)` | Yes | The todo's unique identifier |

#### Request Body

All fields are optional. Include only the fields to be changed.

```json
{
  "title": "Buy groceries and cook dinner",
  "is_complete": true
}
```

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `title` | `string \| null` | No | min length 1, max length 255 if provided |
| `description` | `string \| null` | No | Pass `null` to clear the description |
| `due_date` | `string (date) \| null` | No | ISO 8601 `YYYY-MM-DD`; pass `null` to clear |
| `is_complete` | `boolean \| null` | No | `true` = complete, `false` = incomplete |

#### Responses

**`200 OK`** — returns the full updated todo
```json
{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "title": "Buy groceries and cook dinner",
  "description": "Milk, eggs, bread",
  "due_date": "2026-05-25",
  "is_complete": true,
  "created_at": "2026-05-20T14:30:00Z",
  "updated_at": "2026-05-20T15:00:00Z"
}
```

**`404 Not Found`** — ID does not exist
```json
{
  "status_code": 404,
  "error": "Not Found",
  "message": "Todo with id '3fa85f64-5717-4562-b3fc-2c963f66afa6' was not found."
}
```

**`422 Unprocessable Entity`** — title set to empty string
```json
{
  "status_code": 422,
  "error": "Validation Error",
  "message": "Request body is invalid.",
  "details": [{"field": "title", "message": "String should have at least 1 character"}]
}
```

---

### DELETE `/api/v1/todos/{id}`

Permanently remove a todo item.

#### Path Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | `string (UUID)` | Yes | The todo's unique identifier |

#### Responses

**`204 No Content`** — deletion successful, no body returned.

**`404 Not Found`**
```json
{
  "status_code": 404,
  "error": "Not Found",
  "message": "Todo with id '3fa85f64-5717-4562-b3fc-2c963f66afa6' was not found."
}
```

---

## Health Endpoints

### GET `/health`

Basic liveness check. No authentication required.

**`200 OK`**
```json
{"status": "ok"}
```

### GET `/health/ready`

Readiness check (verifies DB connectivity). No authentication required.

**`200 OK`**
```json
{"status": "ready"}
```

**`503 Service Unavailable`** — DB unreachable
```json
{"status": "unavailable", "detail": "Database connection failed"}
```

---

## OpenAPI / Swagger

The API exposes an interactive OpenAPI UI at `/docs` and a JSON schema at `/openapi.json`. All endpoints declare `summary`, `response_model`, and documented status codes per Principle IV of the constitution.
