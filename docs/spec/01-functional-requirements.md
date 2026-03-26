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

## 2. Claude AI Integration

### FR-006 — Claude Entry Creation Skill
**Priority:** P1

The system shall expose Claude Skills (prompt commands) that guide the user through creating a new entry interactively in a Claude session.

**Acceptance Criteria:**
- Skills available: `/new-note`, `/new-idea`, `/new-article`, `/new-book`, `/new-project`, `/new-meeting`.
- Each skill loads the appropriate template and prompts the user for content conversationally.
- The session maintains context of the in-progress entry across multiple turns.
- The user can iterate on the entry content within the same session before committing.
- The skill presents the final Markdown for user approval before storing.

---

### FR-007 — Entry Storage via Claude
**Priority:** P1

After the user approves the entry in a Claude session, the skill shall persist the entry to Supabase.

**Acceptance Criteria:**
- The entry is validated against the entry type schema before storage.
- The embedding is generated and stored at write time.
- On success, the skill responds with the entry ID and a link to the web UI.
- On failure, the error is surfaced in the session with a retry option.

---

### FR-008 — Semantic Retrieval (RAG) in Claude Sessions
**Priority:** P1

During a Claude session, the user shall be able to ask questions or request related context, and the system shall retrieve semantically relevant entries from the knowledge base.

**Acceptance Criteria:**
- A `/search <query>` skill performs semantic search and returns the top-N most relevant entries.
- Retrieved entries are injected into the Claude context window with source attribution.
- RAG is available in any Claude session, not only entry-creation flows.
- Only entries owned by the authenticated user are searched.
- The number of retrieved results (top-N) is configurable; default is 5.

---

### FR-009 — Contextual Knowledge Use in Creative Work
**Priority:** P2

The user shall be able to reference their knowledge base entries as context when working on creative or analytical tasks in a Claude session.

**Acceptance Criteria:**
- The user can pin specific entries into session context via `/use [[entry-title]]`.
- Pinned entries are summarised and injected into the system prompt for the session.
- Multiple entries can be pinned simultaneously.

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
