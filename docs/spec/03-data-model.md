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

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `entry_id` | `uuid` | PK, FK → `entries.id` ON DELETE CASCADE | One-to-one with entries |
| `embedding` | `vector(1536)` | NOT NULL | pgvector embedding |
| `model` | `text` | NOT NULL | Embedding model used (e.g. `text-embedding-3-small`) |
| `generated_at` | `timestamptz` | NOT NULL, DEFAULT now() | |

**Indexes:**
- IVFFlat index on `embedding vector_cosine_ops` for ANN search.

**RLS:** Users can only access embeddings for their own entries (via join on `entries.user_id`).

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

These are enforced at the application layer. The `metadata` jsonb column stores any fields beyond the base `entries` columns.

### `note`
```jsonb
{
  "related_entries": ["uuid", ...]   // optional
}
```

### `idea`
```jsonb
{
  "related_entries": ["uuid", ...]   // optional
}
```

### `article`
```jsonb
{
  "source_url": "string",            // optional
  "author": "string",                // optional
  "published_date": "date",          // optional
  "related_entries": ["uuid", ...]
}
```

### `book`
```jsonb
{
  "book_author": "string",           // required
  "isbn": "string",                  // optional
  "rating": 1-5,                     // optional
  "reading_status": "to-read|reading|completed",  // required
  "related_entries": ["uuid", ...]
}
```

### `project`
```jsonb
{
  "related_entries": ["uuid", ...]
}
```

### `meeting`
```jsonb
{
  "meeting_date": "date",            // required
  "attendees": ["string", ...],      // optional
  "related_entries": ["uuid", ...]
}
```

---

## Database Triggers

### `handle_new_user`
On `INSERT INTO auth.users` → creates a corresponding row in `profiles`.

### `set_updated_at`
On `UPDATE` of `entries` or `profiles` → sets `updated_at = now()`.

### `queue_embedding_on_change`
On `INSERT OR UPDATE` of `entries` → inserts or updates a row in `embedding_queue` with `status = 'pending'`.
