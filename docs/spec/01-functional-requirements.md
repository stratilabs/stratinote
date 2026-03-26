# Functional Requirements

Each requirement is identified by a unique ID (`FR-XXX`) for traceability. Each has a priority (`P1` = must have, `P2` = should have, `P3` = nice to have) and acceptance criteria.

---

## 1. Entry Management

### FR-001 — Entry Types
**Priority:** P1

The system shall support the following built-in entry types, each with its own Markdown template:

| Type | Layer | Purpose |
|------|-------|---------|
| `note` | Knowledge Base | Free-form permanent note |
| `idea` | Knowledge Base | Captured idea or hypothesis |
| `article` | Knowledge Base | Summary/annotation of an external article |
| `book` | Knowledge Base | Book summary with key insights |
| `project` | Project Workspace | Project brief and context container |
| `meeting` | Project Workspace | Meeting minutes linked to a project |

**Acceptance Criteria:**
- Each entry type has a corresponding Markdown template with a YAML front-matter block.
- Creating an entry without a valid type is rejected with a descriptive error.
- The list of supported types is retrievable via API.

---

### FR-002 — Markdown Templates with Metadata
**Priority:** P1

Each entry type shall have a standardised Markdown template containing:
- A YAML front-matter block with type-specific required and optional fields.
- Section headings appropriate to the entry type.
- Inline guidance comments that are removed before storing.

**Acceptance Criteria:**
- Templates are retrievable by entry type via API.
- Stored entries must pass front-matter schema validation for their type.
- Missing required metadata fields cause a validation error before storage.

**Example — `note` template front-matter:**
```yaml
---
type: note
title: ""
tags: []
status: draft          # draft | active | archived
created_at: ""         # auto-filled
updated_at: ""         # auto-filled
author_id: ""          # auto-filled
related_entries: []    # list of entry IDs
---
```

**Example — `book` template front-matter:**
```yaml
---
type: book
title: ""
author: ""
isbn: ""
tags: []
status: reading        # to-read | reading | completed
rating: null           # 1–5
created_at: ""
updated_at: ""
author_id: ""
related_entries: []
---
```

**Example — `meeting` template front-matter:**
```yaml
---
type: meeting
title: ""
project_id: ""         # required — links to a project entry
date: ""
attendees: []
tags: []
status: draft
created_at: ""
updated_at: ""
author_id: ""
related_entries: []
---
```

---

### FR-003 — Entry CRUD
**Priority:** P1

The system shall support Create, Read, Update, and Delete operations for all entry types.

**Acceptance Criteria:**
- Create: a new entry is persisted with a unique ID, timestamps, and the requesting user's ID.
- Read: entries are retrievable by ID; only entries owned by (or shared with) the requesting user are returned.
- Update: the `updated_at` timestamp is refreshed on every update; entry type cannot be changed after creation.
- Delete: soft-delete by default (status set to `deleted`); hard-delete available as a separate explicit action.
- All operations enforce RLS (see Security Specification).

---

### FR-004 — Entry Cross-Linking
**Priority:** P2

Entries shall support explicit links to other entries via the `related_entries` front-matter field and inline wiki-style links (`[[entry-id]]` or `[[entry-title]]`).

**Acceptance Criteria:**
- The system resolves `[[entry-title]]` links to the correct entry ID on save.
- Broken links (pointing to deleted or non-existent entries) are flagged as warnings, not errors.
- An entry's inbound links (backlinks) are queryable via API.
- Wiki-style links render as clickable hyperlinks in the web UI.

---

### FR-005 — Project Workspace
**Priority:** P1

A `project` entry acts as a container. `meeting` entries and other project-scoped entries must reference a valid `project_id`.

**Acceptance Criteria:**
- Creating a `meeting` entry without a valid `project_id` is rejected.
- All entries belonging to a project are listable via API filtered by `project_id`.
- Deleting a project soft-deletes the project entry; associated entries remain but lose their project reference warning.

---

## 2. Claude AI Integration (MCP)

The Claude AI integration is implemented as a **Model Context Protocol (MCP) server** registered as a Claude.ai integration. The user connects it once in their Claude.ai settings. Authentication between Claude.ai and the MCP server uses a **Personal API Token** (see FR-022). Claude calls MCP tools during conversation; the user never writes raw API calls.

---

### FR-006 — MCP Server and Tool Registration
**Priority:** P1

The system shall implement a Stratinote MCP server that registers the following tools with Claude.ai:

| Tool | Purpose |
|------|---------|
| `get_template` | Returns the Markdown template for a given entry type |
| `create_entry` | Persists a completed, user-approved entry |
| `update_entry` | Updates an existing entry by ID |
| `search_knowledge_base` | Semantic search over the user's knowledge base |
| `get_entry` | Retrieves a single entry by ID or title |
| `list_projects` | Lists available project entries for meeting linking |

**Acceptance Criteria:**
- The MCP server implements the MCP protocol spec and is registerable in Claude.ai as a remote integration.
- Each tool has a clear name, description, and typed input schema so Claude can invoke it correctly without user guidance.
- All tool calls are authenticated via the Personal API Token in the request header (see FR-022).
- A tool call with an invalid or expired token returns a structured MCP error; Claude surfaces this to the user.
- The MCP server is a standalone deployable service (or Next.js API route group) separate from the web-facing REST API.

---

### FR-007 — Conversational Entry Creation via Claude.ai
**Priority:** P1

The Claude.ai integration shall enable the user to create a new entry through natural conversation, without manually filling in templates.

**Flow:**
1. User asks Claude to capture a new note, book summary, idea, etc.
2. Claude calls `get_template` for the appropriate entry type.
3. Claude guides the user conversationally to provide the required content.
4. Claude assembles the final Markdown with front-matter and presents it to the user for review.
5. User approves (or asks for changes and iterates).
6. Claude calls `create_entry` with the final Markdown.
7. Claude confirms with the entry ID and a link to the web UI entry page.

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

### FR-012 — Markdown Entry Editor
**Priority:** P1

The web UI shall provide an inline Markdown editor for viewing and editing entries.

**Acceptance Criteria:**
- Editor supports Markdown syntax highlighting.
- Front-matter metadata is editable via a structured form (not raw YAML).
- Saving triggers re-embedding of the entry.
- Unsaved changes prompt the user before navigation.
- Wiki-style links `[[...]]` render as clickable links.

---

### FR-013 — Entry Creation via Web UI
**Priority:** P2

The web UI shall allow creating new entries directly, as an alternative to the Claude skill flow.

**Acceptance Criteria:**
- User selects entry type, and the template is pre-populated.
- Validation errors are shown inline before save.
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
