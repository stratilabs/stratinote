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
GET /rest/v1/entries
  ?select=id,type_slug,title,tags,status,project_id,created_at,updated_at
  &deleted_at=is.null
  &order=updated_at.desc
  &limit=20
  &offset=0
```
Add filters: `&type_slug=eq.book`, `&tags=cs.{ai,notes}`, `&project_id=eq.<uuid>`

### Get single entry (with related data)
```
GET /rest/v1/entries?id=eq.<uuid>
  &select=*,entry_links!source_entry_id(target_entry_id,field_key,link_type),
          entry_type_definitions!type_definition_id(slug,name,field_definitions(*))
```

### Create entry
```
POST /rest/v1/entries
Content-Type: application/json
Prefer: return=representation

{
  "type_definition_id": "uuid",
  "type_slug": "book",
  "schema_version": 3,
  "title": "Thinking, Fast and Slow",
  "tags": ["psychology"],
  "status": "draft",
  "metadata": {
    "book_author": "Daniel Kahneman",
    "reading_status": "completed",
    "summary": "## Key Insight\nTwo systems..."
  }
}
```
Field values in `metadata` are validated by the `validate_entry_metadata` Edge Function before the PostgREST insert completes (invoked via a Postgres function wrapper).

### Update entry
```
PATCH /rest/v1/entries?id=eq.<uuid>
Content-Type: application/json
Prefer: return=representation

{ "title": "...", "metadata": { "summary": "Updated..." } }
```
> `type_definition_id` is immutable after creation (enforced by trigger).

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
  &select=source_entry_id,entries!source_entry_id(id,title,type_slug)
```

### Entry type definitions (user-facing)
```
# List all types available to the user (system types + own types)
GET /rest/v1/entry_type_definitions
  ?select=id,slug,name,description,icon,color,layer,owner_type,schema_version,
          field_definitions(*)
  &or=(owner_type.eq.system,owner_id.eq.<user_id>)
  &is_deprecated=eq.false
  &order=owner_type.asc,name.asc

# Get a single type with full field definitions
GET /rest/v1/entry_type_definitions?slug=eq.book
  &select=*,field_definitions(*)

# Create a user-defined type
POST /rest/v1/entry_type_definitions
{
  "slug": "research-note",
  "name": "Research Note",
  "description": "...",
  "owner_type": "user",
  "owner_id": "<user_id>",
  "layer": "knowledge_base"
}

# Update a user-defined type (name, description, icon, color)
PATCH /rest/v1/entry_type_definitions?id=eq.<uuid>&owner_id=eq.<user_id>
{ "name": "Updated Name" }
```

### Field definitions (within a user type)
```
# Add a field
POST /rest/v1/field_definitions
{
  "type_definition_id": "uuid",
  "field_key": "summary",
  "label": "Key Insights",
  "field_type": "markdown",
  "display_group": "document",
  "order": 1,
  "required": true
}

# Reorder / update label / hint / deprecate a field
PATCH /rest/v1/field_definitions?id=eq.<uuid>
{ "order": 2, "hint": "Summarise the key takeaways" }

# Deprecate a field (never delete if data exists)
PATCH /rest/v1/field_definitions?id=eq.<uuid>
{ "is_deprecated": true }
```

### RAG contexts
```
# List user's contexts
GET /rest/v1/rag_contexts
  ?select=id,name,description,is_default,filter,created_at
  &order=is_default.desc,name.asc

# Create a context
POST /rest/v1/rag_contexts
{
  "name": "Project Alpha + Psychology",
  "is_default": false,
  "filter": {
    "layers": ["knowledge_base", "project_workspace"],
    "project_ids": ["<project-alpha-uuid>"],
    "tags": ["psychology"]
  }
}

# Update a context
PATCH /rest/v1/rag_contexts?id=eq.<uuid>
{ "name": "...", "filter": { ... }, "is_default": true }

# Delete a context
DELETE /rest/v1/rag_contexts?id=eq.<uuid>
```

### Entry version history
```
# List versions for an entry (most recent first)
GET /rest/v1/entry_versions
  ?entry_id=eq.<uuid>
  &select=id,version_number,title,metadata,tags,change_summary,saved_by,created_at
  &order=version_number.desc

# Get a specific version
GET /rest/v1/entry_versions?entry_id=eq.<uuid>&version_number=eq.<n>
  &select=*
```
> Version restore is handled by the `restore-version` Edge Function (see §2).

### Trash (soft-deleted entries)
```
# List soft-deleted entries
GET /rest/v1/entries
  ?status=eq.deleted
  &select=id,type_slug,title,tags,deleted_at,updated_at
  &order=deleted_at.desc
```
> Restore and hard-delete from trash are handled by Edge Functions (see §2).

### Space shares
```
# List shares I have granted (outgoing)
GET /rest/v1/space_shares?grantor_id=eq.<user_id>
  &select=id,grantee_id,share_type,space_key,created_at,
          profiles!grantee_id(display_name,avatar_url)

# List shares granted to me (incoming)
GET /rest/v1/space_shares?grantee_id=eq.<user_id>
  &select=id,grantor_id,share_type,space_key,created_at,
          profiles!grantor_id(display_name,avatar_url)

# Revoke a share (grantor only; enforced by RLS)
DELETE /rest/v1/space_shares?id=eq.<uuid>
```
> Creating a share requires the `manage-shares` Edge Function (§2) to validate ownership and resolve the grantee's email to a user ID.

### User profile
```
GET  /rest/v1/profiles?id=eq.<user_id>&select=id,display_name,avatar_url,created_at
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
  "limit": 5,
  "include_shared": false
}
```
`include_shared` (default `false`): when `true`, entries from spaces explicitly shared with the requesting user are included in results.

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
      "updated_at": "ISO8601",
      "ownership": "owned | shared",
      "shared_by": "Alice (display name) | null"
    }
  ]
}
```

**Implementation notes:**
- `fulltext`: calls PostgreSQL `to_tsvector` + `ts_rank` via RPC.
- `semantic`: embeds the query via the configured embedding provider, then calls the `match_entries` DB function with `p_include_shared`.
- `hybrid`: runs both, merges and re-ranks results by combined score.
- All modes enforce user scoping internally. `include_shared=true` only surfaces entries covered by an active `space_shares` grant; it cannot be used to access arbitrary entries.

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
Exports non-deleted entries as a ZIP of Markdown files. Scope is configurable.

**Request:**
```json
{
  "scope": {
    "layers": ["knowledge_base"],
    "type_slugs": ["note", "book"],
    "project_id": "uuid | null",
    "rag_context_id": "uuid | null",
    "entry_ids": ["uuid", "..."]
  }
}
```
All scope fields are optional; omit all for a full export. `rag_context_id` applies the named context's filters exactly. Scope fields are ANDed together.

**Response 200:** `Content-Type: application/zip`

ZIP structure:
```
stratinote-export/
  <type_slug>/
    <entry-title-slug>.md
  projects/
    <project-title-slug>/
      project.md
      <linked-entry-slug>.md
```

Each `.md` file contains YAML front-matter with all field values and resolved `related_entries` titles, plus Markdown field values as document body sections.

**Errors:** `NOT_FOUND` if `project_id` or `rag_context_id` doesn't belong to the user; `422` if scope matches zero entries.

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
The Stratinote MCP server. Implements the MCP protocol and is registered as the Claude.ai integration endpoint. Authenticates exclusively via Personal API Token.

**MCP Tools exposed:**

| Tool | Input | Underlying operation |
|------|-------|---------------------|
| `list_entry_types` | (none) | PostgREST: system + user types, non-deprecated |
| `get_type_schema` | `type_slug: string` | PostgREST: type + field_definitions |
| `create_entry` | `type_slug`, `title`, `field_values: {key: value}`, `tags?` | validate → PostgREST insert |
| `update_entry` | `entry_id`, `field_values: {key: value}`, `title?` | validate → PostgREST patch |
| `get_entry` | `entry_id?`, `title?` | PostgREST lookup; title uses full-text search |
| `list_projects` | `include_shared?: boolean` | PostgREST: `type_slug=eq.project&status=eq.active`; when `include_shared=true`, also returns shared projects with `shared_by` field |
| `list_rag_contexts` | (none) | PostgREST: user's `rag_contexts` |
| `list_shared_spaces` | (none) | PostgREST: user's incoming `space_shares` with grantor display names |
| `search_knowledge_base` | `query`, `limit?`, `context_id?`, `type_slug?`, `project_id?`, `include_shared?` | `search` Edge Function with context filter and optional shared scope |

**`get_type_schema` response** (what Claude uses to guide the conversation):
```json
{
  "type_slug": "book",
  "name": "Book Review",
  "fields": [
    { "key": "book_author", "label": "Author", "type": "text", "required": true, "hint": null },
    { "key": "reading_status", "label": "Reading Status", "type": "select", "required": true,
      "options": [{"value":"to-read","label":"To Read"},{"value":"reading","label":"Reading"},{"value":"completed","label":"Completed"}] },
    { "key": "rating", "label": "Rating (1–5)", "type": "number", "required": false },
    { "key": "summary", "label": "Key Insights", "type": "markdown", "required": true, "hint": "Main takeaways and ideas" },
    { "key": "action_items", "label": "Action Items", "type": "markdown", "required": false }
  ]
}
```

**`create_entry` input:**
```json
{
  "type_slug": "book",
  "title": "Thinking, Fast and Slow",
  "tags": ["psychology", "decision-making"],
  "field_values": {
    "book_author": "Daniel Kahneman",
    "reading_status": "completed",
    "rating": 5,
    "summary": "## Core Argument\nKahneman distinguishes...",
    "action_items": "- Apply pre-mortem to next project"
  }
}
```

**`search_knowledge_base` input:**
```json
{
  "query": "string",
  "limit": 5,
  "context_id": "uuid | null",
  "type_slug": "note | null",
  "project_id": "uuid | null",
  "include_shared": false
}
```
If `context_id` is provided, the search applies the context's filter configuration (including `include_shared` from the context's filter, overridden by an explicit `include_shared` parameter). If omitted, the user's default context is applied (or owned entries only if no default exists). Returns full field values so Claude has complete structured context. Shared entry results include a `shared_by` field with the grantor's display name.

---

### `POST /functions/v1/manage-shares`
Create or revoke space shares. **Requires Supabase JWT** (share management is not callable via PAT for security).

**Create a share:**
```json
{
  "action": "create_share",
  "grantee_email": "alice@example.com",
  "share_type": "layer",
  "space_key": "knowledge_base"
}
```
```json
{
  "action": "create_share",
  "grantee_email": "alice@example.com",
  "share_type": "project",
  "space_key": "<project-entry-uuid>"
}
```
**Response 201:**
```json
{
  "id": "uuid",
  "grantee_id": "uuid",
  "grantee_display_name": "Alice",
  "share_type": "layer",
  "space_key": "knowledge_base",
  "created_at": "ISO8601"
}
```

**Revoke a share:**
```json
{ "action": "revoke_share", "share_id": "uuid" }
```
**Response 200:** `{ "revoked": true }`

**Errors:**
- `404` — grantee email not registered in the system.
- `403` — attempting to share a space the caller does not own, or attempting to re-share a received grant.
- `409` — share already exists (duplicate).
- `422` — invalid `share_type` or `space_key`.

Share creation and revocation are written to `audit_log`.

---

### `POST /functions/v1/admin`
Superadmin operations. **Requires Supabase JWT from a superadmin user.** Not callable via PAT.

**Actions:**

```json
{ "action": "list_users", "page": 1, "limit": 50 }
```
```json
{ "action": "suspend_user", "user_id": "uuid" }
{ "action": "unsuspend_user", "user_id": "uuid" }
{ "action": "delete_user", "user_id": "uuid" }
```
```json
{ "action": "create_invite", "email": "optional@email.com", "expires_at": "ISO8601 | null" }
{ "action": "list_invites", "status": "pending | used | expired | revoked | all" }
{ "action": "revoke_invite", "invite_id": "uuid" }
```
```json
{ "action": "get_health" }
```
```json
{ "action": "create_system_type", "slug": "...", "name": "...", "layer": "knowledge_base", "fields": [...] }
{ "action": "update_system_type", "type_id": "uuid", "fields": [...] }
{ "action": "deprecate_system_type", "type_id": "uuid" }
{ "action": "requeue_failed_embeddings" }
```

**`get_health` response:**
```json
{
  "users": { "total": 42, "active_7d": 18, "active_30d": 31, "suspended": 1 },
  "entries": { "total": 1204, "by_type": { "note": 400, "book": 120, "..." : "..." } },
  "embeddings": { "queue_depth": 3, "failed_24h": 0, "total": 1201 },
  "types": { "system": 6, "user_defined": 29 }
}
```

All superadmin actions are written to `audit_log`. Non-superadmin callers receive HTTP 403.

---

### `POST /functions/v1/restore-version`
Restores an entry to a saved version snapshot. Requires Supabase JWT or PAT.

**Request:**
```json
{ "entry_id": "uuid", "version_number": 12 }
```

**Response 200:**
```json
{ "restored_to_version": 12, "new_version_number": 34 }
```

The restore applies `title`, `metadata`, and `tags` from the specified version snapshot to the current entry, then creates a new version recording the restore (with `change_summary: "Restored to version 12"`). Entry `status` and `entry_links` are unchanged. The entry is queued for re-embedding.

**Errors:** `NOT_FOUND` if entry or version doesn't exist; `FORBIDDEN` if not owned by the user.

---

### `POST /functions/v1/trash`
Manage soft-deleted entries (restore, hard-delete individual, empty trash). Requires Supabase JWT or PAT.

**Restore a single entry:**
```json
{ "action": "restore", "entry_id": "uuid" }
```
Response: `{ "restored": true, "status": "active" }`

**Hard-delete a single entry from trash:**
```json
{ "action": "hard_delete", "entry_id": "uuid" }
```
Response: `{ "deleted": true }`

**Empty trash (all soft-deleted entries):**
```json
{ "action": "empty_trash" }
```
Response: `{ "deleted_count": 14 }`

All hard-delete actions write to `audit_log`.

---

### `POST /functions/v1/purge-trash` *(internal, cron-triggered)*
Not user-callable. Runs daily via `pg_cron`. Permanently deletes entries where `deleted_at < now() - interval 'TRASH_RETENTION_DAYS days'`. Writes aggregate to `audit_log`.

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
| `manage-shares` | 20 requests/hour |
| `restore-version` | 30 requests/hour |
| `trash` | 30 requests/hour |
| `mcp-server` | 120 requests/minute |
| All others | 120 requests/minute |

PostgREST endpoints are rate-limited at the Supabase project level (configurable in project settings).

Responses include:
- `X-RateLimit-Limit`
- `X-RateLimit-Remaining`
- `X-RateLimit-Reset` (Unix timestamp)
