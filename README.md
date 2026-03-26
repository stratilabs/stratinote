# Stratinote

**AI-enhanced, Zettelkasten-inspired personal knowledge management system.**

Stratinote bridges a permanent knowledge base (notes, books, articles, ideas) with project workspaces (projects, meetings, ephemeral content), using Claude AI to make capturing, linking, and retrieving knowledge as frictionless as possible.

> The system adapts to the user, not the other way around.

---

## What it does

- **Structured knowledge capture** — Entry types with typed fields (text, markdown, date, select, references, …) define the shape of your knowledge. Create your own types or use the built-in ones.
- **Claude AI integration** — A Claude.ai MCP integration lets you create and search entries through natural conversation. Claude guides you through each field, no templates to fill in manually.
- **Semantic search (RAG)** — pgvector-powered semantic search over your knowledge base, scoped by optional RAG contexts to prevent knowledge from different projects bleeding together.
- **Structured document editor** — A Tiptap-based editor where the entry type's field structure is seamlessly embedded into the writing surface. No switching between form and document views.
- **Space sharing** — Share a knowledge base layer or a specific project with another user (read-only). Shared entries are clearly marked and never writable by the grantee.
- **Versioning and trash** — Every save creates a version snapshot. Soft-deleted entries live in a Trash view with auto-purge after a configurable retention period. Restore any entry or any past version.
- **Selective export** — Export entries as a ZIP of Markdown files, scoped by layer, type, project, RAG context, or explicit list.

---

## Architecture

No custom backend server. Everything runs inside **Supabase**:

| Concern | Solution |
|---------|---------|
| Database | Supabase PostgreSQL 15+ with pgvector and RLS |
| CRUD API | Supabase PostgREST (auto-generated, RLS-enforced) |
| Complex operations | Supabase Edge Functions (Deno / TypeScript) |
| Auth | Supabase Auth (email/password + optional OAuth) |
| MCP server | Supabase Edge Function registered in Claude.ai |
| Frontend | Next.js (React) — static/SSR, no custom API routes |
| Editor | Tiptap (ProseMirror) |
| Markdown | remark + rehype |

**Edge Functions:**
`mcp-server` · `search` · `process-embedding-queue` · `manage-tokens` · `manage-shares` · `export` · `import` · `restore-version` · `trash` · `admin` · `purge-trash`

---

## Key concepts

| Term | Description |
|------|-------------|
| **Entry** | A unit of knowledge with a type, title, typed fields, tags, and status |
| **Entry Type** | A named field schema — system types (superadmin-managed, available to all) or user types (private) |
| **Field Definition** | A typed slot in a type schema (e.g. "Author" of type `text`, "Key Insights" of type `markdown`) |
| **Knowledge Base** | Permanent entries: Notes, Ideas, Articles, Book Reviews, and user-defined variants |
| **Project Workspace** | A project entry and all entries linked to it via an `entry_reference` field |
| **RAG Context** | A named filter (layers, types, projects, tags) that scopes which entries Claude searches |
| **Space Share** | A read-only access grant from one user to another, covering a layer or project |
| **Personal API Token** | A long-lived bearer token for authenticating MCP tool calls from Claude.ai |
| **Entry Version** | A point-in-time snapshot of an entry's content stored on every save |

---

## MCP Tools (Claude.ai integration)

Once the Stratinote MCP server is connected in Claude.ai, Claude can call:

| Tool | What it does |
|------|-------------|
| `list_entry_types` | List all available entry types |
| `get_type_schema` | Get the field definitions for a type |
| `create_entry` | Create a new entry from structured field values |
| `update_entry` | Update an existing entry |
| `get_entry` | Retrieve an entry by ID or title |
| `search_knowledge_base` | Semantic + full-text search with optional RAG context scoping |
| `list_projects` | List active project entries |
| `list_rag_contexts` | List saved RAG context filter configurations |
| `list_shared_spaces` | List spaces other users have shared with you |

---

## Security model

- **RLS everywhere** — All user tables have Row Level Security. No user can read or write another user's data at the database level, regardless of API input.
- **Dual auth** — Supabase JWTs for web UI sessions; Personal API Tokens (SHA-256 hashed, shown once) for MCP calls.
- **Shared entries are read-only** — Space sharing grants read access only. Write, delete, and restore operations on shared entries are blocked at the RLS layer.
- **Superadmin** — Set only via DB migration; checked via DB read on every admin call (never via JWT claims).
- **Audit log** — PAT events, hard deletes, exports, imports, user management actions, and share grants/revocations are all logged.

---

## Documentation

Full specifications live in [`docs/spec/`](./docs/spec/):

| # | Document | Contents |
|---|---------|---------|
| 00 | [Overview](./docs/spec/00-overview.md) | Vision, architecture, tech stack, glossary |
| 01 | [Functional Requirements](./docs/spec/01-functional-requirements.md) | FR-001 – FR-038 |
| 02 | [Non-Functional Requirements](./docs/spec/02-non-functional-requirements.md) | Performance, reliability, scalability |
| 03 | [Data Model](./docs/spec/03-data-model.md) | Tables, indexes, RLS policies, triggers |
| 04 | [API Specification](./docs/spec/04-api-spec.md) | PostgREST endpoints, Edge Function contracts |
| 05 | [Security Specification](./docs/spec/05-security-spec.md) | Auth, RLS, threat model, audit logging |
| 06 | [Test Specification](./docs/spec/06-test-spec.md) | Unit, integration, RLS, E2E, security, perf |
| 07 | [Extension Points](./docs/spec/07-extension-points.md) | Adding types, MCP tools, embedding providers |

---

## Development status

This repository currently contains the **complete project specification**. Implementation follows spec-driven development: all requirements, data models, API contracts, security policies, and test cases are defined before any code is written.
