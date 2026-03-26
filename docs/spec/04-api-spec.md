# API Specification

All API endpoints are prefixed with `/api/v1`. All requests and responses use `application/json` unless otherwise noted.

Authentication is via Supabase JWT passed as `Authorization: Bearer <token>`. All endpoints require authentication unless marked `[public]`.

---

## Conventions

- Timestamps are ISO 8601 UTC strings.
- UUIDs are standard `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` format.
- Errors follow the format:
  ```json
  {
    "error": {
      "code": "ENTRY_NOT_FOUND",
      "message": "Entry with id '...' not found.",
      "details": {}
    }
  }
  ```
- Pagination uses cursor-based pagination: `?cursor=<last_id>&limit=<n>`.

---

## Error Codes

| Code | HTTP Status | Description |
|------|------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid auth token |
| `FORBIDDEN` | 403 | Authenticated but not allowed |
| `NOT_FOUND` | 404 | Resource does not exist or not visible to user |
| `VALIDATION_ERROR` | 422 | Request body fails schema validation |
| `CONFLICT` | 409 | Duplicate resource (e.g. duplicate link) |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 1. Entries

### `GET /entries`
List entries for the authenticated user.

**Query Parameters:**
| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `type` | `entry_type` | - | Filter by entry type |
| `status` | `entry_status` | `active,draft` | Filter by status |
| `tags` | `string` (comma-separated) | - | Filter entries containing ALL given tags |
| `project_id` | `uuid` | - | Filter by project |
| `sort` | `created_at\|updated_at\|title` | `updated_at` | Sort field |
| `order` | `asc\|desc` | `desc` | Sort order |
| `cursor` | `string` | - | Pagination cursor |
| `limit` | `int` | `20` | Max 100 |

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "type": "note",
      "title": "string",
      "tags": ["string"],
      "status": "active",
      "project_id": null,
      "created_at": "ISO8601",
      "updated_at": "ISO8601"
    }
  ],
  "next_cursor": "uuid | null"
}
```

---

### `POST /entries`
Create a new entry.

**Request Body:**
```json
{
  "type": "note",
  "title": "string",
  "body": "string (markdown)",
  "tags": ["string"],
  "status": "draft",
  "project_id": "uuid | null",
  "metadata": {}
}
```

**Response 201:**
```json
{
  "data": {
    "id": "uuid",
    "type": "note",
    "title": "string",
    "body": "string",
    "tags": ["string"],
    "status": "draft",
    "project_id": null,
    "metadata": {},
    "created_at": "ISO8601",
    "updated_at": "ISO8601"
  }
}
```

**Errors:** `VALIDATION_ERROR` if schema fails; `NOT_FOUND` if `project_id` does not exist.

---

### `GET /entries/:id`
Get a single entry by ID.

**Response 200:** Full entry object (same as POST response).
**Errors:** `NOT_FOUND`.

---

### `PATCH /entries/:id`
Partially update an entry.

**Request Body:** Any subset of `title`, `body`, `tags`, `status`, `metadata`. `type` is immutable.

**Response 200:** Updated full entry object.
**Errors:** `NOT_FOUND`, `VALIDATION_ERROR`.

---

### `DELETE /entries/:id`
Soft-delete an entry (sets `status = 'deleted'`, `deleted_at = now()`).

**Response 204:** No content.
**Errors:** `NOT_FOUND`.

---

### `DELETE /entries/:id?hard=true`
Permanently delete an entry and all associated links and embeddings.

**Response 204:** No content.
**Errors:** `NOT_FOUND`.

---

### `GET /entries/:id/backlinks`
List entries that link to this entry.

**Response 200:**
```json
{
  "data": [
    { "id": "uuid", "title": "string", "type": "note" }
  ]
}
```

---

## 2. Entry Links

### `POST /entries/:id/links`
Create an explicit link from this entry to another.

**Request Body:**
```json
{ "target_entry_id": "uuid" }
```

**Response 201:**
```json
{ "source_entry_id": "uuid", "target_entry_id": "uuid" }
```

**Errors:** `NOT_FOUND`, `CONFLICT` (duplicate link), `VALIDATION_ERROR` (self-link).

---

### `DELETE /entries/:id/links/:target_id`
Remove a link between two entries.

**Response 204:** No content.

---

## 3. Templates

### `GET /templates`
List all available entry type templates.

**Response 200:**
```json
{
  "data": [
    { "type": "note", "template": "string (markdown with front-matter)" }
  ]
}
```

---

### `GET /templates/:type`
Get the template for a specific entry type.

**Response 200:**
```json
{ "type": "note", "template": "string (markdown with front-matter)" }
```

**Errors:** `NOT_FOUND`.

---

## 4. Search

### `POST /search`
Search entries using full-text or semantic search.

**Request Body:**
```json
{
  "query": "string",
  "mode": "fulltext | semantic | hybrid",
  "filters": {
    "type": "entry_type | null",
    "tags": ["string"],
    "project_id": "uuid | null"
  },
  "limit": 5
}
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "type": "note",
      "title": "string",
      "excerpt": "string (150 char snippet)",
      "score": 0.92,
      "tags": ["string"],
      "updated_at": "ISO8601"
    }
  ]
}
```

**Errors:** `VALIDATION_ERROR`.

---

## 5. Claude Skills Integration

These endpoints are called by Claude skill implementations running server-side.

### `POST /skills/entry`
Programmatic entry creation from a Claude skill after user approval.

Same contract as `POST /entries`. Requires a valid user JWT (passed from the skill session context).

---

### `POST /skills/rag`
Retrieve top-N relevant entries for a RAG query within a Claude session.

**Request Body:**
```json
{
  "query": "string",
  "limit": 5,
  "filters": {
    "type": "entry_type | null",
    "project_id": "uuid | null"
  }
}
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "uuid",
      "title": "string",
      "type": "note",
      "body": "string (full markdown body)",
      "score": 0.91
    }
  ]
}
```

---

## 6. Export / Import

### `GET /export`
Download full knowledge base as a ZIP of Markdown files.

**Response 200:** `Content-Type: application/zip`, binary ZIP stream.

---

### `POST /import`
Import Markdown files into the knowledge base.

**Request:** `Content-Type: multipart/form-data`, files as `file[]`.

**Response 202:**
```json
{
  "imported": 12,
  "skipped": 2,
  "errors": [
    { "filename": "broken.md", "reason": "Missing required field: type" }
  ]
}
```

---

## 7. User Profile

### `GET /profile`
Get the current user's profile.

**Response 200:**
```json
{
  "id": "uuid",
  "display_name": "string",
  "avatar_url": "string | null",
  "created_at": "ISO8601"
}
```

---

### `PATCH /profile`
Update the current user's profile.

**Request Body:** `display_name`, `avatar_url` (any subset).

**Response 200:** Updated profile object.
