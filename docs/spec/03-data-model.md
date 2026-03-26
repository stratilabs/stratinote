# Data Model

---

## Entity Relationship Overview

```
profiles (1) ──── (N) entries
entries   (N) ──── (N) entries         [via entry_links]
entries   (N) ──── (1) projects        [meeting.project_id → entry.id where type='project']
entries   (1) ──── (1) entry_embeddings
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
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**RLS:** Users can only read and update their own profile.

---

### `entries`

The core table. All entry types are stored here; type-specific data lives in the `metadata` JSON column and is validated at the application layer.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `user_id` | `uuid` | NOT NULL, FK → `auth.users.id` | Owner |
| `type` | `entry_type` (enum) | NOT NULL | See entry types enum |
| `title` | `text` | NOT NULL | Entry title |
| `body` | `text` | NOT NULL, DEFAULT '' | Markdown body (excluding front-matter) |
| `metadata` | `jsonb` | NOT NULL, DEFAULT '{}' | Type-specific front-matter fields |
| `tags` | `text[]` | NOT NULL, DEFAULT '{}' | Searchable tags |
| `status` | `entry_status` (enum) | NOT NULL, DEFAULT 'draft' | |
| `project_id` | `uuid` | NULLABLE, FK → `entries.id` | Set for `meeting` and project-scoped entries |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `deleted_at` | `timestamptz` | NULLABLE | Soft delete timestamp |

**Indexes:**
- `(user_id, type)` — filtered list queries
- `(user_id, status)` — status filtering
- `(project_id)` — project workspace queries
- `(tags)` — GIN index for array containment queries
- Full-text: GIN index on `to_tsvector('english', title || ' ' || body)`

**RLS:** Users can only access rows where `user_id = auth.uid()` and `deleted_at IS NULL`.

---

### `entry_type` (enum)

```sql
CREATE TYPE entry_type AS ENUM (
  'note',
  'idea',
  'article',
  'book',
  'project',
  'meeting'
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

### `entry_links`

Explicit cross-links between entries. Supports the `related_entries` field and wiki-style `[[...]]` links resolved at save time.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `source_entry_id` | `uuid` | NOT NULL, FK → `entries.id` ON DELETE CASCADE | |
| `target_entry_id` | `uuid` | NOT NULL, FK → `entries.id` ON DELETE CASCADE | |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Constraints:**
- UNIQUE `(source_entry_id, target_entry_id)`
- CHECK `source_entry_id <> target_entry_id`

**RLS:** A user can only see/create links where both entries belong to them.

---

### `entry_embeddings`

Stores the vector embedding for each entry. Separated from `entries` to keep the main table lean.

The vector dimension is determined by the configured embedding model and is set at deployment time via the `EMBEDDING_DIMENSIONS` environment variable. The default model is `text-embedding-3-small` (1536 dimensions). Changing models requires a schema migration to update the column type and a full re-embedding of all entries.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `entry_id` | `uuid` | PK, FK → `entries.id` ON DELETE CASCADE | One-to-one with entries |
| `embedding` | `vector(EMBEDDING_DIMENSIONS)` | NOT NULL | pgvector embedding; dimension set at deploy time |
| `model` | `text` | NOT NULL | Embedding model used (e.g. `text-embedding-3-small`) |
| `generated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Indexes:**
- IVFFlat index on `embedding vector_cosine_ops` for ANN search. Built after initial data load; not maintained in real-time (pgvector handles incremental updates).

**RLS:** Direct user access is blocked (`USING (false)`). Embeddings are read exclusively via the `match_entries` Postgres function (defined below) which uses `SECURITY DEFINER` and enforces user scoping internally.

---

### `api_tokens`

Stores Personal API Tokens used to authenticate MCP tool calls from Claude.ai.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `user_id` | `uuid` | NOT NULL, FK → `auth.users.id` ON DELETE CASCADE | Token owner |
| `name` | `text` | NOT NULL | User-assigned label (e.g. "Claude.ai") |
| `token_hash` | `text` | NOT NULL, UNIQUE | SHA-256 hash of the plaintext token |
| `token_prefix` | `text` | NOT NULL | First 8 chars of the token for display (e.g. `sn_abc123`) |
| `last_used_at` | `timestamptz` | NULLABLE | Updated on each successful use |
| `revoked_at` | `timestamptz` | NULLABLE | Set when token is revoked |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Constraints:**
- CHECK: a user may not have more than 10 non-revoked tokens (enforced via trigger).

**RLS:** Users can only read and delete their own tokens. Insert is via a server-side function that generates the token, hashes it, and returns the plaintext once.

---

### `embedding_queue`

Tracks entries awaiting (re)embedding after creation or update. Enables async, retryable embedding generation.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | `uuid` | PK, DEFAULT gen_random_uuid() | |
| `entry_id` | `uuid` | NOT NULL, FK → `entries.id` ON DELETE CASCADE | |
| `status` | `text` | NOT NULL, DEFAULT 'pending' | `pending` \| `processing` \| `done` \| `failed` |
| `attempts` | `int` | NOT NULL, DEFAULT 0 | Retry counter |
| `last_error` | `text` | NULLABLE | Last failure message |
| `created_at` | `timestamptz` | NOT NULL, DEFAULT now() | |
| `updated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**RLS:** Service role only (no user-facing access).

---

## Metadata Schemas (per Entry Type)

These are enforced at the application layer. The `metadata` jsonb column stores type-specific fields beyond the base `entries` columns.

**Important:** `related_entries` is **not** stored in `metadata`. Links are stored exclusively in the `entry_links` table (see FR-024). When the API returns an entry, it populates `related_entries` dynamically from `entry_links` for convenience. On write, any `related_entries` values in incoming front-matter are resolved and written to `entry_links`, then removed from `metadata` before persistence.

### `note`
```jsonb
{}
```
*(no type-specific metadata beyond base columns)*

### `idea`
```jsonb
{}
```

### `article`
```jsonb
{
  "source_url": "string",            // optional
  "article_author": "string",        // optional
  "published_date": "date"           // optional ISO 8601
}
```

### `book`
```jsonb
{
  "book_author": "string",           // required
  "isbn": "string",                  // optional
  "rating": 1,                       // optional, integer 1–5
  "reading_status": "to-read"        // required: to-read | reading | completed
}
```

### `project`
```jsonb
{}
```

### `meeting`
```jsonb
{
  "meeting_date": "2026-01-15",      // required ISO 8601 date
  "attendees": ["Alice", "Bob"]      // optional
}
```

---

## Database Functions

### `match_entries(query_embedding vector, match_count int, p_user_id uuid)`
`SECURITY DEFINER` function used for semantic search. Bypasses RLS on `entry_embeddings` internally but enforces user scoping via the `p_user_id` parameter joined against `entries.user_id`. Returns ranked entries with cosine similarity scores.

```sql
-- Signature only; implementation in migration
CREATE OR REPLACE FUNCTION match_entries(
  query_embedding vector,
  match_count     int,
  p_user_id       uuid
)
RETURNS TABLE (
  id       uuid,
  title    text,
  body     text,
  type     entry_type,
  score    float
)
LANGUAGE plpgsql SECURITY DEFINER;
```

---

## Database Triggers

### `handle_new_user`
On `INSERT INTO auth.users` → creates a corresponding row in `profiles`.

### `set_updated_at`
On `UPDATE` of `entries`, `profiles`, `embedding_queue` → sets `updated_at = now()`.

### `queue_embedding_on_change`
On `INSERT OR UPDATE` of `entries` (when `title` or `body` changes) → upserts a row in `embedding_queue` with `status = 'pending'`.

### `enforce_token_limit`
On `INSERT INTO api_tokens` → raises an exception if the user already has 10 non-revoked tokens.

---

## Embedding Queue Worker

The `embedding_queue` table is processed by a **Supabase Edge Function** (`process-embedding-queue`) triggered on a cron schedule (every 60 seconds via `pg_cron`). The function:

1. Claims up to 10 `pending` rows (sets `status = 'processing'`).
2. Fetches each entry's `title || '\n\n' || body` text.
3. Calls the configured embedding provider API.
4. Upserts the result into `entry_embeddings`.
5. Sets the queue row to `done` or `failed` (with `last_error`).
6. Retries up to 3 times before marking `failed` permanently.

This design ensures entry saves are never blocked by embedding latency and embedding generation is retryable without data loss.
