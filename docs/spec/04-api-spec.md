# API Specification

There is no custom backend server. The frontend and MCP server communicate with Supabase directly:

- **CRUD on entries, links, profiles** → Supabase PostgREST (auto-generated from schema, secured by RLS)
- **Complex operations** → Supabase Edge Functions invoked at `https://<project>.supabase.co/functions/v1/<name>`

Authentication for all calls:
- **Web UI sessions**: Supabase JS client handles JWT automatically (from Supabase Auth session).
- **MCP server / PAT calls**: `Authorization: Bearer <personal-api-token>` passed to Edge Functions; Edge Functions resolve the user from the PAT, then operate as that user via the Supabase service client scoped to that user's `user_id`.

---

## 1. PostgREST — Entry CRUD

Supabase exposes all tables as REST resources at `https://<project>.supabase.co/rest/v1/<table>`.
RLS policies enforce ownership on every call. No custom routing needed.

### List entries
```
GET /rest/v1/entries?select=id,type,title,tags,status,project_id,created_at,updated_at
  &deleted_at=is.null
  &order=updated_at.desc
  &limit=20
  &offset=0
```
Add filters: `&type=eq.note`, `&tags=cs.{ai,notes}`, `&project_id=eq.<uuid>`

### Get single entry (with related data)
```
GET /rest/v1/entries?id=eq.<uuid>
  &select=*,entry_links!source_entry_id(target_entry_id)
```

### Create entry
```
POST /rest/v1/entries
Content-Type: application/json
Prefer: return=representation

{ "type": "note", "title": "...", "body": "...", "tags": [], "status": "draft", "metadata": {} }
```

### Update entry
```
PATCH /rest/v1/entries?id=eq.<uuid>
Content-Type: application/json
Prefer: return=representation

{ "title": "...", "body": "...", "updated_at": "now()" }
```
> `type` is immutable; include it in the `PATCH` body and the DB will reject a change via a trigger.

### Soft-delete entry
```
PATCH /rest/v1/entries?id=eq.<uuid>
{ "status": "deleted", "deleted_at": "<ISO8601>" }
```

### Hard-delete entry
Hard delete requires the `hard-delete` Edge Function (see §2) to cascade cleanly and log the action.

### Backlinks
```
GET /rest/v1/entry_links?target_entry_id=eq.<uuid>
  &select=source_entry_id,entries!source_entry_id(id,title,type)
```

### Create entry link
```
POST /rest/v1/entry_links
{ "source_entry_id": "<uuid>", "target_entry_id": "<uuid>" }
```

### Delete entry link
```
DELETE /rest/v1/entry_links?source_entry_id=eq.<uuid>&target_entry_id=eq.<uuid>
```

### List templates (stored in a `templates` table, read-only)
```
GET /rest/v1/templates?select=type,template_markdown
GET /rest/v1/templates?type=eq.note&select=type,template_markdown
```

### User profile
```
GET  /rest/v1/profiles?id=eq.<user_id>&select=*
PATCH /rest/v1/profiles?id=eq.<user_id>
{ "display_name": "...", "avatar_url": "..." }
```

---

## 2. Edge Functions

Edge Functions are deployed to Supabase and invoked over HTTPS. They use the Supabase service client internally but scope all queries to the authenticated user's `user_id`.

All Edge Functions accept `Authorization: Bearer <jwt-or-pat>` and return `application/json`.

**Base URL:** `https://<project>.supabase.co/functions/v1/`

---

### `POST /functions/v1/search`
Full-text and semantic search.

**Request:**
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

**Implementation notes:**
- `fulltext`: calls PostgreSQL `to_tsvector` + `ts_rank` via RPC.
- `semantic`: embeds the query via the configured embedding provider, then calls the `match_entries` DB function.
- `hybrid`: runs both, merges and re-ranks results by combined score.
- All modes enforce user scoping internally.

---

### `POST /functions/v1/manage-tokens`
Create or revoke Personal API Tokens. **Requires Supabase JWT** (not callable via PAT).

**Create token:**
```json
{ "action": "create", "name": "Claude.ai integration" }
```
**Response 201:**
```json
{
  "id": "uuid",
  "name": "Claude.ai integration",
  "token": "sn_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "token_prefix": "sn_xxxxxx",
  "created_at": "ISO8601"
}
```
> `token` is returned **once only**. The plaintext is never persisted server-side.

**List tokens:**
```json
{ "action": "list" }
```
**Response 200:**
```json
{
  "data": [
    { "id": "uuid", "name": "string", "token_prefix": "sn_xxxxxx", "last_used_at": "ISO8601|null", "created_at": "ISO8601" }
  ]
}
```

**Revoke token:**
```json
{ "action": "revoke", "token_id": "uuid" }
```
**Response 200:** `{ "revoked": true }`

**Errors:**
- `401` — no valid Supabase JWT.
- `422` — missing required fields.
- `429` — 10-token limit reached (on create).

---

### `POST /functions/v1/hard-delete-entry`
Permanently deletes an entry and all associated data. Requires Supabase JWT or PAT.

**Request:**
```json
{ "entry_id": "uuid" }
```

**Response 200:** `{ "deleted": true }`

**Implementation:** Deletes from `entries` (cascades to `entry_links` and `entry_embeddings`), writes to `audit_log`.

---

### `POST /functions/v1/export`
Exports all non-deleted entries as a ZIP of Markdown files. Returns a binary ZIP stream.

**Request:** Empty body (user identity from auth header).

**Response 200:** `Content-Type: application/zip`

ZIP structure:
```
stratinote-export/
  note/
    my-first-note.md
  book/
    thinking-fast-and-slow.md
  meeting/
    project-alpha/
      kickoff-2026-01-15.md
```

Each `.md` file contains the full entry with YAML front-matter (including `related_entries` populated from `entry_links`).

---

### `POST /functions/v1/import`
Imports Markdown files. Accepts `multipart/form-data` with `file[]` fields.

**Response 200:**
```json
{
  "imported": 12,
  "skipped": 2,
  "errors": [
    { "filename": "broken.md", "reason": "Missing required field: type" }
  ]
}
```

Import is synchronous. Files are processed in the request; results are returned immediately. Each imported entry is queued for embedding via the `embedding_queue` trigger.

---

### `POST /functions/v1/mcp-server`
The Stratinote MCP server. Implements the Model Context Protocol and is registered as the Claude.ai integration endpoint. Authenticates exclusively via Personal API Token.

**MCP Tools exposed:**

| Tool | Description | Underlying operation |
|------|-------------|---------------------|
| `get_template` | Returns Markdown template for an entry type | `GET /rest/v1/templates?type=eq.<type>` |
| `create_entry` | Persists a new entry from Markdown | Parses front-matter → `POST /rest/v1/entries` + resolve wiki-links |
| `update_entry` | Updates an existing entry | `PATCH /rest/v1/entries?id=eq.<id>` |
| `get_entry` | Retrieves an entry by ID or title | PostgREST lookup; title uses full-text match |
| `list_projects` | Lists active project entries | `GET /rest/v1/entries?type=eq.project&status=eq.active` |
| `search_knowledge_base` | Semantic/full-text search | Calls `search` Edge Function |

**`create_entry` accepts full Markdown:**
```json
{
  "markdown": "---\ntype: note\ntitle: My Note\ntags: [ai]\n---\n\nBody here."
}
```
The Edge Function parses the YAML front-matter, extracts `related_entries` (resolves to `entry_links`), and inserts the structured entry.

**`search_knowledge_base` input:**
```json
{
  "query": "string",
  "limit": 5,
  "type": "entry_type | null",
  "project_id": "uuid | null"
}
```
Returns full entry body for Claude's context window.

---

### `POST /functions/v1/process-embedding-queue` *(internal, cron-triggered)*
Not user-callable. Triggered by `pg_cron` every 60 seconds. Claims pending items from `embedding_queue`, generates embeddings via the configured provider, and upserts to `entry_embeddings`.

---

## 3. Error Format

All Edge Functions return errors in a consistent shape:

```json
{
  "error": {
    "code": "ENTRY_NOT_FOUND",
    "message": "Entry with id '...' not found.",
    "details": {}
  }
}
```

Standard error codes:

| Code | HTTP Status | Description |
|------|------------|-------------|
| `UNAUTHORIZED` | 401 | Missing or invalid auth |
| `FORBIDDEN` | 403 | Authenticated but not the owner |
| `NOT_FOUND` | 404 | Resource not found or not visible to user |
| `VALIDATION_ERROR` | 422 | Input fails schema validation |
| `CONFLICT` | 409 | Duplicate resource |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Unexpected error |

PostgREST errors follow the [PostgREST error format](https://postgrest.org/en/stable/references/errors.html) and are wrapped by the Supabase JS client.

---

## 4. Rate Limiting

Edge Functions enforce rate limiting via an in-memory sliding window per `user_id`:

| Function | Limit |
|----------|-------|
| `search` | 30 requests/minute |
| `export` | 5 requests/hour |
| `import` | 10 requests/hour |
| `manage-tokens` | 20 requests/hour |
| `mcp-server` | 120 requests/minute |
| All others | 120 requests/minute |

PostgREST endpoints are rate-limited at the Supabase project level (configurable in project settings).

Responses include:
- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset` (Unix timestamp)
