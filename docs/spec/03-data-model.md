# Data Model

---

## Entity Relationship Overview

```
profiles (1) ──── (N) entries
profiles (1) ──── (N) entry_type_definitions   [user-owned types]
profiles (1) ──── (N) rag_contexts
entry_type_definitions (1) ──── (N) field_definitions
entry_type_definitions (1) ──── (N) entries    [via type_definition_id]
entries   (N) ──── (N) entries                 [via entry_links]
entries   (1) ──── (1) entry_embeddings
entries   (1) ──── (N) entry_versions
auth.users (1) ──── (N) api_tokens
auth.users (1) ──── (N) invites                [created_by]
```

---

## Tables

### `profiles`

Extends Supabase Auth `auth.users`. Created automatically on first sign-in via a database trigger.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, FK → `auth.users.id` | Matches Auth user ID |
| `display_name` | `text` | NOT NULL | User's chosen display name |
| `avatar_url` | `text` | NULLABLE | URL to avatar image |
| `is_superadmin` | `boolean` | NOT NULL, DEFAULT false | Superadmin flag; set only via DB migration, never via API |
| `last_active_at` | `timestamptz` | NULLABLE | Updated on authenticated activity for health metrics |
| `suspended_at` | `timestamptz` | NULLABLE | Set when superadmin suspends the account |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**RLS:**
- Users can read and update their own profile (except `is_superadmin` and `suspended_at`).
- Superadmins can read all profiles and update `suspended_at`.

---

### `entry_type_definitions`

Defines a schema for an entry type. Can be owned by the system (superadmin-managed) or by a user.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `slug` | `text` | NOT NULL, UNIQUE | URL-safe identifier (e.g. `book`, `my-meeting-v2`) |
| `name` | `text` | NOT NULL | Display name (e.g. "Book Review") |
| `description` | `text` | NULLABLE | Optional description shown in the type picker |
| `icon` | `text` | NULLABLE | Icon identifier (emoji or icon name) |
| `color` | `text` | NULLABLE | Hex color for visual identity |
| `owner_type` | `text` | NOT NULL | `system` or `user` |
| `owner_id` | `uuid` | NULLABLE, FK → `auth.users.id` | NULL for system types; user ID for user types |
| `layer` | `text` | NOT NULL, DEFAULT 'knowledge_base' | `knowledge_base` or `project_workspace` |
| `is_deprecated` | `boolean` | NOT NULL, DEFAULT false | Hidden from creation picker; existing entries unaffected |
| `forked_from_slug` | `text` | NULLABLE, FK → `entry_type_definitions.slug` | Set when a user forks a system type |
| `schema_version` | `integer` | NOT NULL, DEFAULT 1 | Incremented on any field definition change |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Constraints:**
- CHECK: `owner_type = 'system' AND owner_id IS NULL` OR `owner_type = 'user' AND owner_id IS NOT NULL`
- UNIQUE `(slug)` — global uniqueness; user-defined types auto-prefix with a short user hash to avoid collisions

**RLS:**
- System types (`owner_type = 'system'`) are readable by all authenticated users, mutable by superadmins only.
- User types are readable and mutable by the owning user only.

---

### `field_definitions`

Defines an individual field within an entry type definition.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `type_definition_id` | `uuid` | NOT NULL, FK → `entry_type_definitions.id` ON DELETE CASCADE | Parent type |
| `field_key` | `text` | NOT NULL | Unique slug within the type (e.g. `book_author`, `summary`) |
| `label` | `text` | NOT NULL | Display name shown in editor and to Claude (e.g. "Key Insights") |
| `field_type` | `field_type` (enum) | NOT NULL | See enum below |
| `display_group` | `text` | NOT NULL, DEFAULT 'properties' | `properties` (strip) or `document` (body section) |
| `order` | `integer` | NOT NULL, DEFAULT 0 | Sort order within `display_group` |
| `required` | `boolean` | NOT NULL, DEFAULT false | |
| `default_value` | `jsonb` | NULLABLE | Pre-populated value for new entries |
| `options` | `jsonb` | NULLABLE | Array of `{value: string, label: string}` for select/multi_select |
| `entry_reference_type` | `text` | NULLABLE | For `entry_reference`: constrains target to a specific type slug |
| `hint` | `text` | NULLABLE | Helper text shown beneath the field in the editor |
| `is_deprecated` | `boolean` | NOT NULL, DEFAULT false | Hidden in editor; data preserved |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Constraints:**
- UNIQUE `(type_definition_id, field_key)`
- CHECK: `field_key ~ '^[a-z][a-z0-9_]*$'`
- CHECK: `field_type NOT IN ('select', 'multi_select') OR (options IS NOT NULL AND jsonb_array_length(options) > 0)`
- `field_type` cannot be updated after the field has been used in any entry (enforced via trigger)

**RLS:** Inherits access from parent `entry_type_definitions` row.

---

### `field_type` (enum)

```sql
CREATE TYPE field_type AS ENUM (
  'text',
  'long_text',
  'markdown',
  'date',
  'datetime',
  'number',
  'url',
  'select',
  'multi_select',
  'boolean',
  'entry_reference'
);
```

---

### `entry_status` (enum)

```sql
CREATE TYPE entry_status AS ENUM (
  'draft',
  'active',
  'archived',
  'deleted'
);
```

---

### `entries`

The core table. All field values are stored in `metadata` keyed by `field_key`. The `body` column is removed; `search_text` is a generated column for full-text indexing.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `user_id` | `uuid` | NOT NULL, FK → `auth.users.id` | Owner |
| `type_definition_id` | `uuid` | NOT NULL, FK → `entry_type_definitions.id` | Resolves the schema |
| `type_slug` | `text` | NOT NULL | Denormalized slug for fast filtering without JOIN |
| `schema_version` | `integer` | NOT NULL | Schema version at write time; for future migration tooling |
| `title` | `text` | NOT NULL | Always present; used for display, search, wiki-link resolution |
| `metadata` | `jsonb` | NOT NULL, DEFAULT '{}' | All field values: `{ "field_key": value, ... }` |
| `search_text` | `text` | GENERATED ALWAYS AS (...) STORED | Concatenation of title + all text-type field values; used for full-text GIN index |
| `tags` | `text[]` | NOT NULL, DEFAULT '{}' | Searchable tags (system field) |
| `status` | `entry_status` | NOT NULL, DEFAULT 'draft' | |
| `project_id` | `uuid` | NULLABLE, FK → `entries.id` | Denormalized from the `entry_reference` field with `entry_reference_type: project`; maintained by trigger |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `deleted_at` | `timestamptz` | NULLABLE | Soft delete timestamp |

**Notes on `search_text`:**
The generated column concatenates `title` and the values of all fields with `field_type IN ('text', 'long_text', 'markdown')`. Because PostgreSQL generated columns cannot call user-defined functions directly, this concatenation is maintained by the `update_search_text` trigger instead of a true generated column. The column is functionally equivalent.

**Indexes:**
- `(user_id, type_slug)` — filtered list queries
- `(user_id, status)` — status filtering
- `(project_id)` — project workspace queries
- `(tags)` — GIN index for array containment
- GIN on `to_tsvector('english', search_text)` — full-text search

**RLS:** Users can only access rows where `user_id = auth.uid()` and `deleted_at IS NULL`.

---

### `entry_links`

Explicit cross-links between entries, sourced from `entry_reference` fields and wiki-style `[[...]]` links.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `source_entry_id` | `uuid` | NOT NULL, FK → `entries.id` ON DELETE CASCADE | |
| `target_entry_id` | `uuid` | NOT NULL, FK → `entries.id` ON DELETE CASCADE | |
| `link_type` | `text` | NOT NULL, DEFAULT 'wiki' | `wiki` (from `[[...]]`) or `field` (from `entry_reference` field) |
| `field_key` | `text` | NULLABLE | Set for `link_type = 'field'`; the originating field key |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Constraints:**
- UNIQUE `(source_entry_id, target_entry_id, field_key)` — allows one wiki link and one field link between the same pair
- CHECK `source_entry_id <> target_entry_id`

**RLS:** A user can only see/create links where both entries belong to them.

---

### `entry_embeddings`

Stores the vector embedding for semantic search.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `entry_id` | `uuid` | PK, FK → `entries.id` ON DELETE CASCADE | |
| `embedding` | `vector(EMBEDDING_DIMENSIONS)` | NOT NULL | Dimension set at deploy time via `EMBEDDING_DIMENSIONS` secret |
| `model` | `text` | NOT NULL | Embedding model used |
| `generated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Indexes:** IVFFlat on `embedding vector_cosine_ops`.

**RLS:** Blocked for direct access (`USING (false)`). Read exclusively via `match_entries` (`SECURITY DEFINER`).

---

### `api_tokens`

Personal API Tokens for MCP authentication.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `user_id` | `uuid` | NOT NULL, FK → `auth.users.id` ON DELETE CASCADE | |
| `name` | `text` | NOT NULL | User-assigned label |
| `token_hash` | `text` | NOT NULL, UNIQUE | SHA-256 of plaintext token |
| `token_prefix` | `text` | NOT NULL | First 8 chars for display |
| `last_used_at` | `timestamptz` | NULLABLE | |
| `revoked_at` | `timestamptz` | NULLABLE | |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Constraints:** Max 10 active tokens per user (enforced via trigger).

**RLS:** Users read/delete their own tokens. Insert via `manage-tokens` Edge Function only.

---

### `embedding_queue`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `entry_id` | `uuid` | NOT NULL, UNIQUE, FK → `entries.id` ON DELETE CASCADE | |
| `status` | `text` | NOT NULL, DEFAULT 'pending' | `pending` \| `processing` \| `done` \| `failed` |
| `attempts` | `int` | NOT NULL, DEFAULT 0 | |
| `last_error` | `text` | NULLABLE | |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**RLS:** Service role only.

---

### `invites`

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `token_hash` | `text` | NOT NULL, UNIQUE | SHA-256 of the invite token |
| `token_prefix` | `text` | NOT NULL | First 8 chars for display in admin UI |
| `email` | `text` | NULLABLE | Optional pre-fill; does not restrict who can use the link |
| `created_by` | `uuid` | NOT NULL, FK → `auth.users.id` | Superadmin who created it |
| `used_by` | `uuid` | NULLABLE, FK → `auth.users.id` | Set on use |
| `expires_at` | `timestamptz` | NULLABLE | |
| `revoked_at` | `timestamptz` | NULLABLE | |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `used_at` | `timestamptz` | NULLABLE | |

**RLS:** Superadmins can read/write all rows. Unauthenticated users can validate a token hash (read-only, by hash only — no enumeration).

---

### `rag_contexts`

Named, saved filter configurations that scope `search_knowledge_base` results. Owned by individual users.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `user_id` | `uuid` | NOT NULL, FK → `auth.users.id` ON DELETE CASCADE | Owner |
| `name` | `text` | NOT NULL | Display name (e.g. "Project Alpha + Psychology KB") |
| `description` | `text` | NULLABLE | Optional description |
| `is_default` | `boolean` | NOT NULL, DEFAULT false | Applied automatically when no context_id is specified |
| `filter` | `jsonb` | NOT NULL, DEFAULT '{}' | See filter schema below |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Filter schema (`filter` jsonb):**
```jsonb
{
  "layers": ["knowledge_base", "project_workspace"],  // null = all layers
  "type_slugs": ["note", "book"],                     // null = all types
  "project_ids": ["uuid1", "uuid2"],                  // null = all projects
  "exclude_project_ids": ["uuid3"],                   // null = exclude nothing
  "tags": ["python", "ai"]                            // null = all tags
}
```
An empty `{}` filter (all null fields) means "search everything" — valid as a named global context.

**Constraints:**
- UNIQUE partial index: `(user_id) WHERE is_default = true` — only one default per user.

**RLS:** Users can only read, create, update, and delete their own contexts.

---

### `entry_versions`

Point-in-time snapshots of entry content. Written on every save that changes `title`, `metadata`, or `tags`. Provides history and restore capability.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `entry_id` | `uuid` | NOT NULL, FK → `entries.id` ON DELETE CASCADE | Parent entry |
| `version_number` | `integer` | NOT NULL | Sequential per entry, starting at 1 |
| `title` | `text` | NOT NULL | Snapshot of title at this version |
| `metadata` | `jsonb` | NOT NULL | Snapshot of all field values |
| `tags` | `text[]` | NOT NULL | Snapshot of tags |
| `saved_by` | `uuid` | NOT NULL, FK → `auth.users.id` | User who saved this version |
| `change_summary` | `text` | NULLABLE | Auto-generated diff summary (e.g. "Updated Key Insights; added tag 'ai'") |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | When this version was saved |

**Constraints:**
- UNIQUE `(entry_id, version_number)`

**Retention:** A trigger prunes versions older than the 50 most recent per entry after each insert.

**RLS:** Users can only read versions of their own entries. No user write access (versions are written by the `snapshot_version` trigger only).

---

### `audit_log`

Immutable event log for sensitive operations. Service role only — not user-readable via API.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `event_type` | `text` | NOT NULL | E.g. `pat_created`, `entry_hard_deleted`, `user_suspended` |
| `actor_id` | `uuid` | NULLABLE | The user performing the action (nullable for system events) |
| `target_id` | `uuid` | NULLABLE | The affected resource (user ID, entry ID, token ID, etc.) |
| `metadata` | `jsonb` | NOT NULL, DEFAULT '{}' | Additional context (token_prefix, entry type, etc.) — no PII, no content |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**RLS:** Superadmins can read. No user write access. Written exclusively by Edge Functions via service role.

---

## Database Functions

### `match_entries(query_embedding vector, match_count int, p_user_id uuid)`
`SECURITY DEFINER` for semantic search. Enforces user scoping via join on `entries.user_id`.

```sql
CREATE OR REPLACE FUNCTION match_entries(
  query_embedding vector,
  match_count     int,
  p_user_id       uuid
)
RETURNS TABLE (id uuid, title text, metadata jsonb, type_slug text, score float)
LANGUAGE plpgsql SECURITY DEFINER;
```

---

## Database Triggers

| Trigger | On | Action |
|---------|-----|--------|
| `handle_new_user` | INSERT `auth.users` | Creates `profiles` row |
| `set_updated_at` | UPDATE `entries`, `profiles`, `entry_type_definitions`, `embedding_queue`, `rag_contexts` | Sets `updated_at = now()` |
| `queue_embedding_on_change` | INSERT OR UPDATE `entries` (title or metadata changed) | Upserts `embedding_queue` row with `status='pending'` |
| `sync_project_id` | INSERT OR UPDATE `entries` | Copies `entry_reference` field value with `entry_reference_type='project'` into denormalized `project_id` |
| `update_search_text` | INSERT OR UPDATE `entries` | Rebuilds `search_text` from title + all text-type field values via type schema lookup |
| `snapshot_version` | INSERT OR UPDATE `entries` (when `title`, `metadata`, or `tags` changed) | Inserts a row into `entry_versions` with computed `change_summary`; prunes versions beyond 50 for this entry |
| `enforce_token_limit` | INSERT `api_tokens` | Raises exception if user has ≥ 10 active tokens |
| `prevent_field_type_change` | UPDATE `field_definitions` | Raises exception if `field_type` changes and any entry uses that field |
| `increment_schema_version` | INSERT OR UPDATE OR DELETE `field_definitions` | Increments `entry_type_definitions.schema_version` |
| `enforce_single_default_context` | INSERT OR UPDATE `rag_contexts` | If `is_default = true`, sets `is_default = false` on all other contexts for that user |

---

## Embedding Queue Worker

`process-embedding-queue` Edge Function runs on a `pg_cron` schedule (every 60 seconds):
1. Claims ≤ 10 `pending` rows atomically.
2. For each entry, concatenates `title + '\n\n' + search_text` for embedding input.
3. Calls the configured provider (`EMBEDDING_PROVIDER` secret).
4. Upserts into `entry_embeddings`.
5. Sets status to `done` or `failed`; retries up to 3 times before permanent failure.
