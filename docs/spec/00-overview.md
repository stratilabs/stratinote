# Stratinote — Project Overview

## Vision

Stratinote is an AI-enhanced, Zettelkasten-inspired personal knowledge management system. It bridges a permanent knowledge base (articles, books, notes, ideas) with project-specific workspaces (projects, meeting minutes, ephemeral content), using Claude AI to make capturing, linking, and retrieving knowledge as frictionless as possible.

The guiding principle: **the system adapts to the user, not the other way around**.

---

## Goals

1. Make knowledge capture fast and structured using Claude-driven entry creation flows.
2. Store all knowledge in well-structured, human-readable Markdown with rich metadata.
3. Enable semantic retrieval of past knowledge inside Claude sessions (RAG).
4. Provide a clean web UI for browsing, searching, and editing entries.
5. Support multiple isolated users with strong security guarantees.
6. Be extensible: new entry types, templates, and AI skills can be added with minimal friction.

---

## Glossary

| Term | Definition |
|------|-----------|
| **Entry** | A single unit of knowledge stored in the system. Always a typed Markdown document with a metadata block. |
| **Entry Type** | The category of an entry (e.g. Article, Book, Note, Idea, Project, Meeting). Each type has its own template. |
| **Knowledge Base** | The collection of permanent entries: Articles, Books, Notes, Ideas. |
| **Project Workspace** | A named container grouping project-related entries (Project brief, Meetings, Tasks). |
| **Template** | A Markdown scaffold with a YAML front-matter metadata block for a given Entry Type. |
| **Skill** | A Claude prompt command that initiates or continues a specific workflow (e.g. `/new-note`, `/new-project`). |
| **Embedding** | A vector representation of an entry's content stored in Supabase for semantic search. |
| **RAG** | Retrieval-Augmented Generation — pulling relevant entries from the knowledge base to augment a Claude session. |
| **RLS** | Row Level Security — Supabase/PostgreSQL policy system that enforces per-user data isolation at the database level. |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Claude Skills                     │
│   (Entry creation, in-chat refinement, RAG queries) │
└────────────────────┬────────────────────────────────┘
                     │ Anthropic API
┌────────────────────▼────────────────────────────────┐
│                  Backend API                         │
│         (REST + server-side logic layer)             │
└──────┬─────────────────────────────────┬────────────┘
       │                                 │
┌──────▼──────┐                 ┌────────▼────────────┐
│  Supabase   │                 │    Supabase Auth     │
│  PostgreSQL │                 │  (JWT + RLS)         │
│  + pgvector │                 └─────────────────────┘
└──────┬──────┘
       │
┌──────▼──────────────────────────────────────────────┐
│                  Web Application                     │
│        (Browse, search, create, edit entries)        │
└─────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Database | Supabase (PostgreSQL 15+) | Managed, supports pgvector, built-in auth and RLS |
| Vector search | pgvector (Supabase) | Co-located with data, no separate vector DB needed |
| Auth | Supabase Auth | JWT-based, social + email, integrates with RLS |
| Storage | Supabase Storage | File attachments, scoped to user via RLS |
| AI | Anthropic Claude API | Entry creation, refinement, semantic Q&A, RAG |
| Backend API | Node.js / TypeScript (tbd) | Type-safe, integrates with Supabase client and Anthropic SDK |
| Frontend | React / Next.js (tbd) | SSR-capable, easy Supabase and Markdown integration |
| Markdown rendering | MDX or remark | Render and edit Markdown with extensions |

---

## Document Index

| # | Document | Description |
|---|---------|-------------|
| 00 | [Overview](./00-overview.md) | This document |
| 01 | [Functional Requirements](./01-functional-requirements.md) | What the system must do |
| 02 | [Non-Functional Requirements](./02-non-functional-requirements.md) | Performance, reliability, scalability |
| 03 | [Data Model](./03-data-model.md) | Entities, schemas, relationships |
| 04 | [API Specification](./04-api-spec.md) | Endpoint contracts |
| 05 | [Security Specification](./05-security-spec.md) | Auth, RLS, threat model |
| 06 | [Test Specification](./06-test-spec.md) | Test strategy and cases |
| 07 | [Extension Points](./07-extension-points.md) | How to add entry types, templates, skills |
