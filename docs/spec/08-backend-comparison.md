# Backend Comparison: QMD (Local Files + Git) vs Supabase

This document compares two backend approaches for Stratinote's knowledge storage layer:

1. **QMD** — Local Quarto Markdown (`.qmd`) files on disk, versioned with local Git
2. **Supabase** — Cloud-hosted PostgreSQL with PostgREST, Edge Functions, and RLS (current spec)

The comparison is evaluated against Stratinote's core goals: fast knowledge capture, structured storage, semantic retrieval, clean web UI, multi-user support, and extensibility.

---

## Architecture Overview

### QMD Approach

```
stratinote-vault/
├── .git/                          ← Local git repo (versioning)
├── .stratinote/
│   ├── config.yaml                ← Type definitions, field schemas
│   ├── embeddings.db              ← SQLite/DuckDB for vector index
│   └── index.db                   ← SQLite for full-text search + metadata index
├── knowledge-base/
│   ├── notes/
│   │   ├── my-first-note.qmd
│   │   └── quantum-computing.qmd
│   ├── books/
│   │   └── designing-data-intensive-apps.qmd
│   └── ideas/
│       └── ai-knowledge-graphs.qmd
└── projects/
    ├── project-alpha/
    │   ├── _project.qmd
    │   └── meetings/
    │       └── 2026-04-10-kickoff.qmd
    └── project-beta/
        └── _project.qmd
```

Each `.qmd` file contains YAML front-matter (metadata, tags, type info) and Markdown body:

```yaml
---
title: "Designing Data-Intensive Applications"
type: book
tags: [distributed-systems, databases]
status: active
fields:
  book_author: "Martin Kleppmann"
  rating: 5
  key_insights: |
    - Data models shape how we think about problems
    - Replication and partitioning are fundamental trade-offs
created: 2026-03-15T10:30:00Z
updated: 2026-04-02T14:22:00Z
---

## Summary

A comprehensive guide to the principles behind reliable, scalable,
and maintainable data systems...

## Notes

[[quantum-computing]] has interesting parallels with distributed consensus...
```

The **local server** (e.g., a lightweight Rust/Go/Node daemon, or a Next.js API route) reads/writes these files and maintains the search indexes.

### Supabase Approach (Current Spec)

```
┌─────────────────┐     ┌──────────────────────────────────┐
│  Next.js Client │────▶│  Supabase                        │
│  Claude MCP     │     │  PostgreSQL + pgvector + RLS     │
│                 │     │  Edge Functions (Deno)            │
│                 │     │  Auth (JWT + PAT)                 │
└─────────────────┘     └──────────────────────────────────┘
```

All data lives in PostgreSQL. Entries are rows with JSONB metadata. Versioning is via `entry_versions` table. Auth and RLS enforce multi-user isolation.

---

## Feature-by-Feature Comparison

### 1. Data Storage

| Aspect | QMD (Local Files + Git) | Supabase (PostgreSQL) |
|--------|------------------------|----------------------|
| **Format** | Human-readable `.qmd` files with YAML front-matter + Markdown body | Rows in PostgreSQL; metadata in JSONB column |
| **Readability** | Files are directly readable/editable in any text editor, VS Code, Obsidian, etc. | Data only accessible via API or database client |
| **Portability** | Inherently portable — it's just files on disk | Requires export to move data; import/export Edge Functions needed |
| **Durability** | As durable as the filesystem + git remotes | Managed by Supabase; automatic backups on paid plans |
| **Schema enforcement** | Validated at write time by local server against `.stratinote/config.yaml` | Enforced by PostgreSQL constraints, triggers, and Edge Functions |
| **Structured metadata** | YAML front-matter (typed fields, tags, status) | JSONB `metadata` column with trigger-based validation |

**Verdict:** QMD wins on readability, portability, and "own your data" philosophy. Supabase wins on schema enforcement rigor and query capabilities.

---

### 2. Versioning

| Aspect | QMD (Local Files + Git) | Supabase (`entry_versions` table) |
|--------|------------------------|----------------------------------|
| **Mechanism** | Real git commits — full history, branching, merging | Database rows: snapshots of title/metadata/tags per save |
| **Granularity** | Per-file diffs with line-level precision | Whole-entry JSONB snapshots; diff computed at application layer |
| **History depth** | Unlimited (full git log) | Capped at 50 versions per entry (pruned by trigger) |
| **Tooling** | `git log`, `git diff`, `git blame`, GitHub/GitLab UI, any git client | Custom version history UI in Stratinote web app |
| **Branching** | Native git branches — experiment with entry changes, merge back | Not supported |
| **Conflict resolution** | Git merge tooling (manual or automated) | Last-write-wins (single-user per entry) |
| **Restore** | `git checkout <commit> -- path/to/file.qmd` | Edge Function applies version snapshot, creates new version |
| **Remote backup** | `git push` to GitHub/GitLab — free, encrypted, familiar | Supabase manages backups; point-in-time recovery on Pro plan |

**Verdict:** Git versioning is strictly superior for a single-user or small-team knowledge base. Unlimited history, line-level diffs, branching, and ecosystem tooling far exceed database-level snapshots.

---

### 3. Search

| Aspect | QMD (Local Files + Git) | Supabase |
|--------|------------------------|---------|
| **Full-text** | SQLite FTS5 index (rebuilt from file contents) or ripgrep for raw search | PostgreSQL GIN index on `to_tsvector('english', search_text)` |
| **Semantic** | Local vector DB (SQLite-vss, LanceDB, Chroma, or DuckDB) | pgvector with IVFFlat index; `match_entries()` function |
| **Performance** | Fast for <100K entries; index lives on local SSD | Scales to millions of rows; managed infrastructure |
| **Embedding generation** | Local or remote API call; stored in local vector DB | Async queue worker (Edge Function + pg_cron) |
| **Query flexibility** | Custom queries via SQLite; can add faceted search easily | Full SQL power; PostgREST filters; custom DB functions |

**Verdict:** Comparable for the expected scale (single user, thousands of entries). Supabase has an edge at very large scale. QMD is simpler to operate since there's no cloud dependency for search.

---

### 4. Authentication & Multi-User

| Aspect | QMD (Local Files + Git) | Supabase |
|--------|------------------------|---------|
| **Single user** | Natural fit — files on your machine, your git repo | Works but is over-engineered for single-user |
| **Multi-user isolation** | Requires separate vaults per user or OS-level permissions; sharing = git remotes or shared folders | Native: RLS policies enforce row-level isolation; `space_shares` for controlled access |
| **Auth** | Can skip entirely for local use; add API keys for MCP server | Full auth stack: JWT, email/password, OAuth, PATs |
| **Sharing** | Share via git remote, symlinks, or read-only git subtree | `space_shares` table with granular layer/project grants |
| **Admin capabilities** | File-system permissions; no built-in admin UI | Superadmin role with user management, invites, health dashboard |

**Verdict:** Supabase is far better for multi-user scenarios. QMD is simpler and more natural for single-user or personal use.

---

### 5. MCP / Claude Integration

| Aspect | QMD (Local Files + Git) | Supabase |
|--------|------------------------|---------|
| **MCP server** | Local process or Claude Desktop MCP server reading files directly | Edge Function hosted on Supabase |
| **Entry creation** | MCP tool writes `.qmd` file to disk; auto-commits via git | MCP tool calls PostgREST INSERT via Edge Function |
| **Search** | MCP tool queries local vector DB + FTS index | MCP tool calls `search` Edge Function |
| **Auth for MCP** | Filesystem access (local) or simple API key (remote) | PAT-based authentication |
| **Latency** | Near-zero for local; network hop for remote git | Network hop to Supabase Edge Function |
| **Offline capability** | Full functionality offline; sync when online | Requires internet connection |

**Verdict:** QMD with a local MCP server is simpler and works offline. Supabase MCP is more robust for remote/multi-device access.

---

### 6. Development & Operations

| Aspect | QMD (Local Files + Git) | Supabase |
|--------|------------------------|---------|
| **Setup complexity** | Clone repo + run local server; no cloud accounts needed | Supabase project + migrations + secrets + Edge Function deployments |
| **Infrastructure cost** | $0 (local compute) + optional git hosting (free tier) | Free tier limited; Pro plan ~$25/mo for production |
| **Deployment** | Run locally or self-host the server anywhere | Managed by Supabase; Edge Functions deploy via CLI |
| **Vendor lock-in** | None — standard files, standard git | Moderate — PostgREST API, Edge Functions, RLS policies are Supabase-specific |
| **Debugging** | Read files, inspect git log, query SQLite directly | Supabase dashboard, Edge Function logs, SQL queries |
| **CI/CD potential** | Git hooks for validation, linting, auto-tagging | Database migrations via Supabase CLI |
| **Backup** | `git push` (already done) | Supabase automatic backups; manual pg_dump |

**Verdict:** QMD is dramatically simpler to develop, deploy, and operate. Zero cloud dependencies. Supabase trades simplicity for managed scalability.

---

### 7. Data Integrity & Consistency

| Aspect | QMD (Local Files + Git) | Supabase |
|--------|------------------------|---------|
| **Transactions** | Filesystem writes are not transactional (but git commits are atomic) | Full ACID transactions |
| **Referential integrity** | Enforced at application layer; broken links detected by index rebuild | PostgreSQL foreign keys; cascade deletes |
| **Concurrent writes** | Git handles via merge; local server can use file locks | PostgreSQL MVCC; row-level locking |
| **Schema migration** | Update `config.yaml`; revalidate files with a script | SQL migrations; trigger-enforced constraints |

**Verdict:** Supabase has stronger guarantees for complex multi-table consistency. QMD is adequate for single-user knowledge management where conflicts are rare.

---

## Hybrid Architecture (Recommended)

Given Stratinote's goals — especially "the system adapts to the user, not the other way around" — a **QMD-primary with optional Supabase sync** architecture captures the best of both:

```
┌──────────────────────────────────────────────────────┐
│  Local Machine                                        │
│                                                       │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ .qmd files  │  │ Local Server │  │ Git repo     │ │
│  │ (vault)     │◀▶│ (API + MCP)  │  │ (versioning) │ │
│  └─────────────┘  └──────┬───────┘  └──────────────┘ │
│                          │                            │
│  ┌───────────────────────▼──────────────────────────┐ │
│  │ .stratinote/                                      │ │
│  │  embeddings.db (vector search)                    │ │
│  │  index.db (full-text search + metadata cache)     │ │
│  │  config.yaml (type definitions)                   │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────┬───────────────────────────┘
                           │ Optional: git push / Supabase sync
┌──────────────────────────▼───────────────────────────┐
│  Remote (Optional)                                    │
│  • GitHub/GitLab for git backup                       │
│  • Supabase for web UI, multi-device, sharing         │
└──────────────────────────────────────────────────────┘
```

### Why This Works

1. **Local-first, own your data** — Knowledge lives as readable files on your machine
2. **Git-native versioning** — Unlimited history, diffs, branching, blame — no cap at 50 versions
3. **Offline-capable** — Full functionality without internet
4. **Zero cost** — No cloud services required for core functionality
5. **Claude integration** — Local MCP server reads/writes files directly; near-zero latency
6. **Escape hatch** — If Supabase/any service shuts down, your data is safe in git
7. **Gradual complexity** — Start local-only; add sync/sharing/web UI when needed

### What You Lose vs Pure Supabase

1. **Multi-user RLS** — Must be rebuilt at the API layer or deferred to Supabase sync
2. **Real-time collaboration** — Not practical with file-based storage
3. **Managed infrastructure** — You run the local server yourself
4. **Space sharing** — Requires git remote sharing or Supabase sync layer
5. **Superadmin features** — Less relevant for local-first single-user

---

## Decision Matrix

| Criterion | Weight | QMD Score (1-5) | Supabase Score (1-5) | QMD Weighted | Supabase Weighted |
|-----------|--------|-----------------|---------------------|-------------|-------------------|
| Data ownership & portability | 5 | 5 | 2 | 25 | 10 |
| Versioning quality | 5 | 5 | 3 | 25 | 15 |
| Offline capability | 4 | 5 | 1 | 20 | 4 |
| Setup & operational simplicity | 4 | 5 | 2 | 20 | 8 |
| Cost | 3 | 5 | 3 | 15 | 9 |
| Claude/MCP integration | 5 | 4 | 4 | 20 | 20 |
| Search (FTS + semantic) | 4 | 4 | 5 | 16 | 20 |
| Multi-user support | 2 | 2 | 5 | 4 | 10 |
| Sharing & collaboration | 2 | 2 | 5 | 4 | 10 |
| Schema enforcement | 3 | 3 | 5 | 9 | 15 |
| Web UI capability | 3 | 3 | 4 | 9 | 12 |
| Scalability (>10K entries) | 2 | 3 | 5 | 6 | 10 |
| **Total** | | | | **173** | **143** |

---

## Recommendation

**Go with QMD (local files + git) as the primary storage layer.** The weighted analysis strongly favors it for Stratinote's use case.

### Key reasons:

1. **Alignment with vision** — "Store all knowledge in well-structured, human-readable Markdown with rich metadata" is literally the QMD file format
2. **Git versioning is a superpower** — The current spec limits versions to 50 per entry; git has no limit and provides superior diffing, blame, and branching
3. **Local-first = resilient** — No dependency on cloud services for core knowledge management
4. **Simpler to build** — No Supabase project setup, no Edge Function deployments, no RLS policy debugging, no migration management
5. **MCP integration is cleaner** — A local MCP server reading files is simpler than an Edge Function proxy chain
6. **Cost** — $0 vs $25+/mo, with no capability trade-off for single-user

### Defer Supabase for:

- When you need multi-device web access (add as a sync target)
- When you need to share spaces with other users
- When you need the web UI to work without a local server running

This means the initial build focuses on the local server + MCP + file-based storage, and the web UI can read from the same local vault (Next.js dev server) or from an optional Supabase sync.

---

## Impact on Current Spec

If adopting the QMD approach, the following spec documents would change:

| Document | Impact |
|----------|--------|
| **00-overview.md** | Architecture diagram replaces Supabase with local server + git |
| **01-functional-requirements.md** | FR-015–017 (auth) simplified; FR-033–034 (versioning) replaced by git; FR-037–038 (sharing) deferred |
| **02-non-functional-requirements.md** | Remove Supabase-specific SLAs; add local performance targets |
| **03-data-model.md** | Replace PostgreSQL schema with `.qmd` file format spec + SQLite index schema |
| **04-api-spec.md** | Replace PostgREST + Edge Functions with local REST API spec |
| **05-security-spec.md** | Simplify dramatically; RLS replaced by filesystem permissions + API key for MCP |
| **06-test-spec.md** | Replace pgTAP with file-based test fixtures; keep Vitest + Playwright |
| **07-extension-points.md** | Update to reference `config.yaml` for type definitions instead of DB migrations |
