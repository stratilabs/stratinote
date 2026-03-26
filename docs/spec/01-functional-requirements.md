# Functional Requirements

Each requirement is identified by a unique ID (`FR-XXX`) for traceability. Each has a priority (`P1` = must have, `P2` = should have, `P3` = nice to have) and acceptance criteria.

---

## 1. Entry Management

### FR-001 — Dynamic Entry Type System
**Priority:** P1

Entry types are not hardcoded. Every type is defined by an **Entry Type Definition**: a named schema consisting of typed field definitions. Users see only types available to them (system types + their own types).

**Tier model:**

| Tier | Defined by | Available to | Mutable by |
|------|-----------|-------------|-----------|
| **System** | Superadmin | All users | Superadmin only |
| **User** | Individual user | That user only | That user |
| **Fork** | Individual user | That user only | That user (derived from a system type) |

**Built-in system types** (pre-seeded, equivalent to the previous hardcoded set):

| Type slug | Name | Layer |
|-----------|------|-------|
| `note` | Note | knowledge_base |
| `idea` | Idea | knowledge_base |
| `article` | Article | knowledge_base |
| `book` | Book Review | knowledge_base |
| `project` | Project | project_workspace |
| `meeting` | Meeting | project_workspace |

**Acceptance Criteria:**
- The type a user selects when creating an entry can be any system type or any type they own.
- Retrieving the list of available types returns system types + the requesting user's own types.
- Creating an entry against a deleted or another user's type is rejected.
- Entry type cannot be changed after creation.

---

### FR-002 — Field Type System
**Priority:** P1

Each Entry Type Definition contains an ordered list of **Field Definitions**. Every field has a type from the supported set below.

**Supported field types:**

| Field type | Storage | Description | Example use |
|-----------|---------|-------------|-------------|
| `text` | string | Single-line text | Author name, ISBN |
| `long_text` | string | Multi-line plain text | Short description |
| `markdown` | string | Full Markdown content block | Key insights, action items |
| `date` | ISO 8601 date | Date picker | Meeting date, published date |
| `datetime` | ISO 8601 datetime | Date + time picker | Event timestamp |
| `number` | number | Numeric value | Rating (1–5), page count |
| `url` | string | URL with format validation | Source link |
| `select` | string | Single choice from predefined options | Reading status |
| `multi_select` | string[] | Multiple choices from options | Categories |
| `boolean` | boolean | Yes/no toggle | Reviewed, archived |
| `entry_reference` | uuid | FK to another entry | Project link (for meetings) |

**System fields** are always present on every entry regardless of type and are **not** part of the field definition schema:
- `title` (text, required, always first)
- `tags` (multi_select, optional)
- `status` (select: draft/active/archived/deleted)
- `created_at`, `updated_at`, `owner` (auto-populated)

**Field Definition properties:**
- `field_key` — unique slug within the type (immutable after creation)
- `label` — display name shown in editor and Claude conversations
- `field_type` — one of the supported types above
- `required` — boolean; enforced at save time
- `default_value` — optional; used to pre-populate the editor
- `options` — for `select` / `multi_select`: ordered list of `{value, label}` pairs
- `hint` — optional helper text shown in the editor beneath the field
- `entry_reference_type` — for `entry_reference`: constrains the target to a specific type slug (e.g. `project`)
- `display_group` — `properties` (rendered in the properties strip) or `document` (rendered as a document section). `markdown` and `long_text` fields default to `document`; all others default to `properties`
- `order` — display position within its `display_group`

**Acceptance Criteria:**
- A type definition with no fields (beyond system fields) is valid.
- Field keys must be unique within a type and match `^[a-z][a-z0-9_]*$`.
- A `select` or `multi_select` field with no options is rejected.
- `entry_reference` fields with an `entry_reference_type` that does not match any known type slug are rejected.
- Field type cannot be changed after a field is created (to protect existing entry data).

---

### FR-003 — Entry CRUD
**Priority:** P1

The system shall support Create, Read, Update, and Delete operations for all entry types.

**Acceptance Criteria:**
- Create: a new entry is persisted with a unique ID, timestamps, and the requesting user's ID.
- Read: entries are retrievable by ID; only entries owned by the requesting user are returned.
- Update: the `updated_at` timestamp is refreshed on every update; entry type and field keys cannot be changed after creation.
- All field values in `metadata` are validated against the type's field definitions at write time; missing required fields cause a validation error.
- Delete: soft-delete by default (status set to `deleted`); hard-delete available as a separate explicit action.
- All operations enforce RLS (see Security Specification).

---

### FR-004 — Entry Cross-Linking
**Priority:** P2

Entries shall support explicit links to other entries via `entry_reference` fields and inline wiki-style links (`[[entry-title]]`) in `markdown` fields.

**Acceptance Criteria:**
- `entry_reference` field values are validated as existing entry IDs owned by the user.
- `[[entry-title]]` links in Markdown fields are resolved to entry IDs on save and stored in `entry_links`.
- Broken links are flagged as warnings, not errors.
- An entry's inbound links (backlinks) are queryable via API.
- Wiki-style links render as clickable hyperlinks in the web UI editor.

---

### FR-005 — Project Workspace
**Priority:** P1

Project-scoped entries (type layer = `project_workspace`) must include an `entry_reference` field targeting a `project` type entry. This replaces the hardcoded `project_id` column.

**Acceptance Criteria:**
- System type `meeting` has a required `entry_reference` field `project_id` with `entry_reference_type: project`.
- User-defined types with layer `project_workspace` must include at least one required `entry_reference` field.
- Creating a project-workspace entry without a valid project reference is rejected.
- All entries belonging to a project are listable via API filtered by `project_id` (resolved from the `entry_reference` field value).

---

### FR-025 — Entry Type Definition CRUD (User Types)
**Priority:** P1

Users shall be able to create, read, update, and delete their own entry type definitions via the web UI.

**Acceptance Criteria:**
- A user can create a new type definition with a name, optional description, icon, and ordered list of field definitions.
- A user can reorder, add, or update fields in their own types.
- A user cannot delete a field that has data in any existing entry (they may mark it deprecated instead; deprecated fields are hidden in the editor but their data is preserved).
- A user cannot delete a type that has existing non-deleted entries; they must archive it first.
- Changes to a type definition increment its `schema_version`; existing entries retain their original `schema_version` and remain valid.
- Retrieving a type definition includes the `schema_version` and a `field_count`.

---

### FR-026 — Type Forking
**Priority:** P2

Users shall be able to fork a system entry type to create a personalised variant.

**Acceptance Criteria:**
- Forking creates a new user-owned type pre-populated with the system type's field definitions.
- The fork is independent; subsequent changes to the system type do not propagate to the fork.
- The fork records `forked_from` (the source system type slug) for informational purposes.
- The user can add, remove, or reorder fields on the forked type freely.

---

---

## 2. Claude AI Integration (MCP)

The Claude AI integration is implemented as a **Model Context Protocol (MCP) server** registered as a Claude.ai integration. The user connects it once in their Claude.ai settings. Authentication between Claude.ai and the MCP server uses a **Personal API Token** (see FR-022). Claude calls MCP tools during conversation; the user never writes raw API calls.

---

### FR-006 — MCP Server and Tool Registration
**Priority:** P1

The system shall implement a Stratinote MCP server that registers the following tools with Claude.ai:

| Tool | Purpose |
|------|---------|
| `list_entry_types` | Lists all entry types available to the user (system + own) |
| `get_type_schema` | Returns the field definitions for a given type slug or ID |
| `create_entry` | Persists a new entry as structured field values |
| `update_entry` | Updates an existing entry by ID |
| `search_knowledge_base` | Semantic search over the user's knowledge base |
| `get_entry` | Retrieves a single entry by ID or title |
| `list_projects` | Lists available project entries for meeting linking |

**Acceptance Criteria:**
- The MCP server implements the MCP protocol spec and is registerable in Claude.ai as a remote integration.
- Each tool has a clear name, description, and typed input schema so Claude can invoke it correctly without user guidance.
- `get_type_schema` returns an ordered list of field definitions (key, label, type, required, options, hint) — not a Markdown template.
- `create_entry` accepts a structured map of `{ field_key: value }` pairs, validated server-side against the type schema.
- All tool calls are authenticated via the Personal API Token in the request header (see FR-022).
- A tool call with an invalid or expired token returns a structured MCP error; Claude surfaces this to the user.
- The MCP server is a Supabase Edge Function.

---

### FR-007 — Conversational Entry Creation via Claude.ai
**Priority:** P1

The Claude.ai integration shall enable the user to create a new entry through natural conversation, without manually filling in templates.

**Flow:**
1. User asks Claude to capture a new note, book summary, idea, etc.
2. Claude calls `list_entry_types` to confirm available types, then `get_type_schema` for the chosen type.
3. Claude guides the user conversationally, asking for each required field by its `label`.
4. Claude summarises the collected values and presents them to the user for review.
5. User approves (or asks for changes and iterates).
6. Claude calls `create_entry` with the structured `{ field_key: value }` map.
7. Claude confirms with the entry ID and a direct link to the web UI entry page.

**Acceptance Criteria:**
- The user never needs to write YAML front-matter manually; Claude fills it from the conversation.
- The user can request changes and re-review before committing (multi-turn iteration).
- `create_entry` validates the Markdown against the entry type schema server-side before persisting; validation errors are returned as tool errors and surfaced by Claude.
- On success, the MCP tool returns the entry ID and a direct URL to the web UI.
- On failure, Claude surfaces the error and offers to retry.
- Supported entry types: `note`, `idea`, `article`, `book`, `project`, `meeting`.

---

### FR-008 — Semantic Knowledge Retrieval (RAG) via Claude.ai
**Priority:** P1

During any Claude.ai session with the Stratinote integration active, Claude shall be able to retrieve semantically relevant entries from the user's knowledge base to answer questions or provide context.

**Flow:**
1. User asks a question or requests context ("what do I know about X?").
2. Claude calls `search_knowledge_base` with a semantic query.
3. The MCP server calls the backend search API using the Personal API Token.
4. Retrieved entries (title, excerpt, entry ID) are returned to Claude.
5. Claude answers using the retrieved content, citing the entry titles as sources.

**Acceptance Criteria:**
- `search_knowledge_base` accepts: `query` (string), `limit` (int, default 5, max 20), optional `type` and `project_id` filters.
- Only entries belonging to the token's owner are searched.
- Retrieved entries include full body content so Claude has complete context.
- Claude attributes answers to specific entries by title.
- RAG works in any conversation, not only entry-creation flows.

---

### FR-009 — Contextual Entry Pinning via Claude.ai
**Priority:** P2

The user shall be able to explicitly load specific entries into a Claude.ai conversation context by referencing them by title or ID.

**Flow:**
1. User tells Claude "use my notes on [[Topic X]] for this".
2. Claude calls `get_entry` with the title or ID.
3. The entry content is returned and held in the conversation context.
4. Claude uses it for the remainder of the session.

**Acceptance Criteria:**
- `get_entry` accepts either an entry UUID or an entry title (resolved server-side).
- Multiple entries can be fetched and used in the same session.
- If the entry is not found or doesn't belong to the user, a clear error is returned.

---

## 3. Web Application

### FR-010 — Knowledge Base Browser
**Priority:** P1

The web UI shall provide a browseable list of all entries with filtering and sorting.

**Acceptance Criteria:**
- Filter by: entry type, tags, status, project.
- Sort by: created date (desc/asc), updated date (desc/asc), title.
- Entries display: title, type badge, tags, status, last updated date.
- Pagination or infinite scroll with at least 20 entries per page.

---

### FR-011 — Full-Text and Semantic Search in Web UI
**Priority:** P1

The web UI shall provide a search interface supporting both full-text and semantic (AI-powered) search.

**Acceptance Criteria:**
- Full-text search queries entry titles and body content.
- Semantic search is triggered explicitly (e.g. toggle or separate button).
- Results are ranked by relevance.
- Search respects user-scoped RLS (no cross-user result leakage).

---

### FR-012 — Structured Document Editor
**Priority:** P1

The web UI shall provide a unified document editor where the entry type's field structure is seamlessly embedded into the writing surface. The user writes naturally without switching between form and document views.

**Layout:**

```
┌─────────────────────────────────────────────────────┐
│  [Title — H1, always first, always editable]        │
├─────────────────────────────────────────────────────┤
│  Properties strip (collapsible):                    │
│  Author [__________]  Rating [★★★★☆]               │
│  Status [completed ▾] Date   [2026-01-15]           │
├─────────────────────────────────────────────────────┤
│  Key Insights          ← non-editable section label │
│  ┌─────────────────────────────────────────────┐    │
│  │ This book argues that...                    │    │
│  │                                             │    │
│  └─────────────────────────────────────────────┘    │
│                                                     │
│  Action Items          ← non-editable section label │
│  ┌─────────────────────────────────────────────┐    │
│  │ - Apply slow thinking to ...                │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

**Editor implementation:** [Tiptap](https://tiptap.dev/) (ProseMirror-based). The document schema is derived from the entry type's field definitions at load time. Document-group fields become Tiptap node types with fixed, non-deletable section headings.

**Acceptance Criteria:**
- `document`-group fields (`markdown`, `long_text`) render as labelled sections in document flow order.
- Section label headings are visible and styled as H2 but are not editable or deletable by the user.
- The cursor moves naturally between sections (Tab / arrow keys cross section boundaries).
- `properties`-group fields render in a collapsible strip above the document body.
- Each property field renders its appropriate interactive widget inline (date picker, star rating, select dropdown, URL input, entry reference picker).
- Markdown syntax shortcuts work inside `markdown` fields (e.g. `**bold**`, `- list item`, `## heading`).
- Wiki-style links `[[entry title]]` in markdown fields are rendered as clickable chips with the linked entry's title; unresolved links render as plain text with a warning indicator.
- Saving re-queues the entry for embedding.
- Unsaved changes trigger a confirmation prompt before navigation.
- The editor is read-only for entries belonging to other users (future sharing feature) and for system entry type definitions.

---

### FR-013 — Entry Creation via Web UI
**Priority:** P2

The web UI shall allow creating new entries directly, as an alternative to the Claude MCP flow.

**Acceptance Criteria:**
- User selects an available entry type from a picker (system types shown first, then user types).
- The structured document editor opens with default values pre-populated and focus on the title field.
- Validation errors are shown inline (required fields highlighted) before the entry can be saved.
- After creation, the user is redirected to the new entry's page.

---

### FR-014 — Project Workspace View
**Priority:** P1

The web UI shall provide a dedicated view per project showing all associated entries.

**Acceptance Criteria:**
- Accessible from the entry list when a `project` entry is selected.
- Shows the project brief alongside all linked entries (meetings, notes, etc.).
- Entries can be created directly within the project view, pre-populating `project_id`.

---

## 4. Authentication and User Management

### FR-015 — User Authentication
**Priority:** P1

The system shall support user authentication via Supabase Auth.

**Acceptance Criteria:**
- Email/password authentication is supported.
- OAuth providers (Google, GitHub) are supported as optional extensions.
- Unauthenticated requests to any protected API endpoint return HTTP 401.
- Sessions expire after a configurable inactivity period (default: 24 hours).

---

### FR-016 — Multi-User Isolation
**Priority:** P1

All user data (entries, embeddings, projects) shall be strictly isolated between users.

**Acceptance Criteria:**
- RLS policies prevent any user from reading, writing, or deleting another user's entries at the database level.
- No API endpoint returns data belonging to a different user regardless of input parameters.
- Confirmed via cross-user test cases (see Test Specification).

---

### FR-017 — User Profile
**Priority:** P3

Users shall be able to view and update their profile (display name, avatar).

**Acceptance Criteria:**
- Profile data is stored in a `profiles` table linked to the Auth user.
- Profile updates do not affect entry ownership or access.

---

## 5. Embedding and Vector Search

### FR-018 — Automatic Embedding Generation
**Priority:** P1

Every time an entry is created or updated, its content embedding shall be (re)generated and stored.

**Acceptance Criteria:**
- Embedding is generated using the configured embedding model (default: `text-embedding-3-small` via OpenAI or equivalent via Anthropic).
- Embedding generation is performed server-side, not in the browser.
- Failures in embedding generation do not block entry storage; they are retried asynchronously.
- Entries without embeddings are excluded from semantic search results.

---

### FR-019 — Semantic Search API
**Priority:** P1

The system shall expose a semantic search endpoint that accepts a natural language query and returns ranked, user-scoped results.

**Acceptance Criteria:**
- Input: query string, optional filters (type, tags, project_id), top-N (default 5, max 20).
- Output: list of entries with relevance score, title, type, excerpt.
- Search is scoped to the authenticated user's entries only.
- Response time under 2 seconds for a corpus of up to 10,000 entries.

---

## 6. Data Portability

### FR-020 — Export Knowledge Base
**Priority:** P3

Users shall be able to export their entire knowledge base as a ZIP archive of Markdown files.

**Acceptance Criteria:**
- Export includes all non-deleted entries as `.md` files with front-matter intact.
- Files are organised into folders by entry type.
- Export is downloadable from the web UI.

---

### FR-021 — Import from Markdown Files
**Priority:** P3

Users shall be able to import Markdown files (with compatible front-matter) into their knowledge base.

**Acceptance Criteria:**
- System validates front-matter on import; invalid files are reported with details.
- Duplicate detection by title + type; user is prompted to skip or overwrite.
- Imported entries are embedded on ingest.

---

## 7. Personal API Token Management

### FR-022 — Personal API Token Generation
**Priority:** P1

Users shall be able to generate Personal API Tokens (PATs) from the web UI to authenticate the Claude.ai MCP integration.

**Acceptance Criteria:**
- A user can generate one or more named PATs (e.g. "Claude.ai integration", "Home laptop").
- The full token value is shown **once** at creation time and never again; only a masked prefix is stored.
- A PAT grants the same data access as the generating user, scoped by the same RLS policies.
- PATs do not expire by default but can be revoked individually at any time from the web UI.
- Revoking a PAT immediately invalidates all API calls using that token (no grace period).
- The `api_tokens` table stores only the hashed token value (SHA-256); the plaintext is never persisted.
- A user may have at most 10 active PATs simultaneously.

---

### FR-023 — PAT Authentication on API and MCP
**Priority:** P1

The backend API and MCP server shall accept Personal API Tokens as a valid authentication method, in addition to Supabase JWTs.

**Acceptance Criteria:**
- A request with `Authorization: Bearer <pat>` is authenticated by hashing the token and looking it up in `api_tokens`.
- If the token is valid and not revoked, the request proceeds as the owning user.
- Invalid or revoked tokens return HTTP 401.
- PAT usage is logged (last-used timestamp updated on each call) for audit purposes.
- PAT authentication is available on all `/api/v1/*` endpoints except `POST /auth/token` (which issues PATs and requires a Supabase JWT).

---

## 8. Entry Linking — Clarification of Mechanisms

### FR-024 — Single Source of Truth for Entry Links
**Priority:** P1

Entry cross-links are stored exclusively in the `entry_links` table. The `related_entries` field in Markdown front-matter is a **display convenience** rendered from `entry_links` on read and resolved to `entry_links` on write. There is no separate storage of link data in the `metadata` JSON column.

**Acceptance Criteria:**
- On entry save, any `related_entries` values in the front-matter are parsed, resolved to entry IDs, and written to `entry_links`. The field is then stripped from the stored `metadata`.
- On entry read, the API populates `related_entries` in the response by querying `entry_links`, so consumers always see a consistent view.
- Wiki-style `[[title]]` links found in the entry body are resolved and added to `entry_links` at save time.
- Unresolvable `[[title]]` references are preserved as-is in the body and flagged as warnings; no `entry_link` row is created for them.

---

## 9. Superadmin

A **superadmin** is a privileged user with access to the `/admin` section of the web UI. Superadmin status is set via the `is_superadmin` flag on the `profiles` table, enforced at both the API/Edge Function layer and by RLS.

### FR-027 — Superadmin User Management
**Priority:** P1

The superadmin shall have a user management interface for governing all registered users.

**Acceptance Criteria:**
- List all users with columns: email, display name, entry count, type definition count, last active, account status (active/suspended).
- Suspend or unsuspend a user account. Suspended users cannot authenticate; their data is preserved.
- View a user's entry type definitions (names and field counts only — no entry content, for privacy).
- Trigger permanent account deletion. This fires the data deletion cascade (entries, embeddings, tokens, profile).
- All superadmin actions are written to the `audit_log` table.

---

### FR-028 — Invite System
**Priority:** P1

The system shall support an invite-based registration mode controlled by the superadmin.

**Registration mode** is set via the `REGISTRATION_MODE` environment variable:
- `open` — anyone can register.
- `invite_only` — registration requires a valid invite token.

**Acceptance Criteria:**
- Superadmin can generate single-use invite links with an optional expiry date and optional email pre-fill.
- Invite links are listed in the admin UI with status: pending / used / expired / revoked.
- Superadmin can revoke a pending invite; revoked invites are immediately invalid.
- When `REGISTRATION_MODE=invite_only`, registration without a valid invite token returns an error.
- When `REGISTRATION_MODE=open`, the invite system remains available for superadmin use but is not enforced.
- Each invite is single-use: once a user completes registration with it, the invite is marked used and cannot be reused.
- Invite tokens are stored as opaque hashes; the plaintext URL is generated at creation time only.

---

### FR-029 — Superadmin System Type Management
**Priority:** P1

The superadmin shall be able to create, update, and deprecate system entry type definitions, which are available to all users.

**Acceptance Criteria:**
- Superadmin can create new system types with any field schema.
- Superadmin can add new optional fields to existing system types (backward-compatible; increments `schema_version`).
- Superadmin cannot remove a field from a system type that has existing entry data; they may mark the field deprecated (hidden in editor, data preserved).
- Superadmin can mark a system type as deprecated: it remains usable for existing entries but is hidden from the creation picker.
- Superadmin cannot delete a system type that has existing entries across any user.
- Changes to system types do not affect user forks of those types.

---

### FR-030 — Health Dashboard
**Priority:** P2

The superadmin shall have a dashboard showing system health and usage metrics.

**Metrics displayed:**

| Metric | Source |
|--------|--------|
| Total registered users | `auth.users` count |
| Active users (last 7 days / 30 days) | `profiles.last_active_at` |
| Total entries (by type, by status) | `entries` count |
| Embedding queue depth | `embedding_queue` pending count |
| Embedding failure rate (last 24h) | `embedding_queue` failed count |
| Total system types / user-defined types | `entry_type_definitions` counts |
| Storage used | Supabase Storage API |
| Edge Function error rate | Supabase logs API |

**Acceptance Criteria:**
- Dashboard refreshes on page load; no real-time streaming required.
- All metrics are scoped to the production project and visible only to superadmins.
- A "re-queue failed embeddings" action allows superadmin to reset all `failed` queue items to `pending`.
