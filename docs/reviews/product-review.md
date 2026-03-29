# Stratinote — Senior Product Manager Review

**Date:** 2026-03-29
**Reviewer:** Senior PM Review (AI-assisted)
**Scope:** Complete specification review (docs/spec/00 through 07)
**Status:** Pre-implementation — specification phase only

---

## Executive Summary

Stratinote's specifications are impressively thorough and well-structured. The vision of an AI-native Zettelkasten system with Claude MCP integration is compelling and differentiated. However, after reviewing all 38 functional requirements, 21 NFRs, the data model, API spec, security model, and test plan, there are significant gaps and risks that should be addressed **before implementation begins**. These fall into five categories: (1) missing user-facing features critical for adoption, (2) unclear user journeys and onboarding, (3) prioritization concerns, (4) scalability and operational blind spots, and (5) specification ambiguities that will cause implementation friction.

---

## 1. Missing Features Critical for Adoption

### 1.1 No Onboarding or First-Run Experience

The spec defines 38 features but **zero onboarding flow**. A new user who signs up encounters:
- An empty knowledge base browser
- No guided setup for creating their first entry type or entry
- No explanation of what RAG contexts are or why they matter
- No walkthrough of Claude MCP integration setup (PAT generation, Claude.ai registration)

**Recommendation:** Add FR for a first-run experience: welcome wizard, sample entry types pre-populated, guided PAT setup flow with copy-to-clipboard and a link to Claude.ai settings, and contextual tooltips on first visit to each major view.

### 1.2 No Notification or Activity System

There is no mechanism for:
- Notifying a user when someone shares a space with them
- Alerting a superadmin when the embedding queue is failing
- Informing users when an import completes (or fails)
- Communicating system maintenance or degraded state

**Recommendation:** At minimum, add an in-app notification center for share invitations, import/export completion, and system alerts. Email notifications can be P3 but in-app is essential for the sharing feature to work well.

### 1.3 No Collaborative Features Beyond Read-Only Sharing

The sharing model (FR-037/038) is strictly read-only. There's no way for a grantee to:
- Comment on a shared entry
- Suggest edits
- Bookmark or annotate shared entries in their own context
- Get notified when shared content is updated

This makes sharing feel like a "view-only PDF" rather than a collaborative knowledge system. Users will quickly want to discuss shared knowledge.

**Recommendation:** Add at minimum a commenting/annotation system on shared entries (P2). Consider a "shared notes" feature where the grantee can attach their own private annotations to shared entries.

### 1.4 No Search History or Saved Searches

Users can configure RAG contexts, but there's no way to:
- See recent searches
- Save frequently-used search queries
- Access a "recently viewed" entries list

**Recommendation:** Add recent entries, recent searches, and quick-access bookmarks/favorites (P2). These are table-stakes for any knowledge management tool.

### 1.5 No Offline or Partial-Offline Support

The spec assumes always-online connectivity. For a personal knowledge management tool, this is a significant limitation. Users taking notes in meetings, on flights, or with poor connectivity will be blocked.

**Recommendation:** Acknowledge this limitation explicitly. Consider adding a PWA manifest and service worker for at least read-only offline access to recently viewed entries (P3 for v1, but should be on the roadmap).

### 1.6 No Dashboard or Home View

The spec mentions a "Knowledge Base Browser" (FR-010) as the main view, but there's no dashboard or home screen that gives users a quick pulse on their system:
- Recently updated entries
- Entries pending review (draft status)
- Embedding status
- Quick-create shortcuts

**Recommendation:** Add a home dashboard as the default landing page.

---

## 2. User Journey and UX Gaps

### 2.1 Claude MCP Setup is Under-Specified

The spec says "the user connects [the MCP integration] once in their Claude.ai settings" but doesn't define:
- What the setup flow looks like from the user's perspective
- How the user discovers the MCP server URL
- How error states are communicated when the connection fails
- What happens if the PAT is revoked while Claude is mid-conversation

This is the **core differentiator** of the product, and the setup experience is hand-waved.

**Recommendation:** Add a dedicated FR for "MCP Connection Setup Flow" with step-by-step screens: (1) Generate PAT with one click, (2) Show MCP server URL with copy button, (3) Link to Claude.ai integration page, (4) Test connection button, (5) Success/failure feedback.

### 2.2 Entry Type Creation UX is Complex but Underdefined

FR-025 says users can create custom types, but the UX for building a type schema (choosing field types, setting required/optional, defining select options, ordering fields) is not described. This is essentially a form builder and one of the hardest UX problems in software.

**Recommendation:** Add wireframes or at least detailed UX acceptance criteria for the type builder. Consider a "start from template" option where users can fork a system type rather than building from scratch (this aligns with FR-026 but should be surfaced prominently in the type creation flow).

### 2.3 No Guidance on RAG Context Configuration

RAG contexts are powerful but complex. The spec defines the data model but not how a user understands:
- When to create a RAG context vs. just searching
- How to verify their context includes the right entries
- What "include_shared" actually means in practice

**Recommendation:** Add a "preview" feature to RAG contexts that shows which entries match the current filter configuration. Add in-context help text. Consider a "smart defaults" approach where the system suggests RAG contexts based on usage patterns.

### 2.4 The Two-Track Entry Creation is Confusing

Entries can be created via Claude (FR-007) or via Web UI (FR-013), but the spec doesn't address:
- Are both paths equivalent in capability?
- Can Claude create entries with all field types, including `entry_reference`?
- How does the user know which path to use when?

**Recommendation:** Clearly document the parity (or intentional differences) between both paths. If the web UI path is a fallback for when Claude isn't available, say so explicitly.

---

## 3. Prioritization Concerns

### 3.1 P1 Scope is Too Large for v1

The spec marks **20+ features as P1** (must have). This includes the dynamic type system, field type system, all CRUD, all MCP tools, the structured editor, authentication, embeddings, semantic search, superadmin management, invite system, RAG contexts, PAT management, and entry linking. That's essentially the entire product.

**Recommendation:** Split P1 into P0 (absolute minimum for launch) and P1 (must-have for first public release). A viable P0 might be:
- Fixed system types only (no user-defined types initially)
- Basic CRUD + web editor
- Claude MCP with create/search/get
- Auth + single-user mode
- Embedding + semantic search

User-defined types, superadmin, invite system, RAG contexts, space sharing, and versioning can be P1 (first release after MVP validation).

### 3.2 Superadmin Features are Overweight for a PKM Tool

FR-027 through FR-030 define a full admin panel with user management, invite system, system type management, and a health dashboard. For what's described as a "personal knowledge management system," this is enterprise-grade admin tooling that will consume significant development time.

**Recommendation:** For v1, a simpler admin approach: superadmin managed via CLI/SQL scripts, not a full web UI. Invest that development time in the core user experience instead. Build the admin UI when there's actual multi-tenant demand.

### 3.3 Version History Before Basic Features

Version history (FR-033/034) is P2, but features like entry creation via web UI (FR-013) are also P2. The web UI creation flow is far more critical for adoption than versioning.

**Recommendation:** Reprioritize FR-013 (Entry Creation via Web UI) to P1. Users who don't use Claude need a first-class creation path.

---

## 4. Scalability and Operational Blind Spots

### 4.1 No Monitoring or Alerting Strategy

The spec defines an `audit_log` and a health dashboard (FR-030), but there's no specification for:
- Runtime monitoring (Edge Function errors, latency spikes, queue backlogs)
- Alerting (who gets notified when the embedding queue is stuck? when the DB is near capacity?)
- Incident response procedures

**Recommendation:** Add NFR for operational monitoring. At minimum: Supabase dashboard monitoring, email alerts for queue failures, and a documented runbook for common failure modes.

### 4.2 Embedding Queue is a Single Point of Fragility

The embedding queue (FR-018) runs as a cron-triggered Edge Function processing items one batch at a time. The spec says failures retry up to 3 times, but:
- What happens when the embedding provider is down for hours? The queue backs up indefinitely.
- What's the batch size? Processing time?
- What happens if two cron invocations overlap?
- Is there a dead-letter queue for permanently failed items?

**Recommendation:** Define queue processing parameters: batch size, concurrency guards, dead-letter handling, backpressure behavior, and alerting thresholds.

### 4.3 No Rate Limiting on the Web UI

Rate limiting is defined for Edge Functions (search: 30/min, export: 5/hour, etc.) but not for PostgREST endpoints. A malicious or buggy frontend could hammer the database directly.

**Recommendation:** Add rate limiting at the PostgREST/Supabase level or via an API gateway. Define abuse thresholds.

### 4.4 No Data Backup Strategy Beyond Supabase SLA

The spec trusts Supabase for durability (NFR-009) but doesn't define:
- Backup frequency and retention
- Point-in-time recovery capability
- Cross-region redundancy
- Disaster recovery procedures

**Recommendation:** Document the backup strategy explicitly, even if it's "rely on Supabase Pro plan's daily backups." Users entrusting their entire knowledge base to this system need confidence in data safety.

---

## 5. Specification Ambiguities That Will Block Implementation

### 5.1 Wiki-Link Resolution Timing is Unclear

FR-004 says `[[entry-title]]` links are resolved on save, but FR-024 says they're stored in `entry_links`. Questions:
- What if two entries share the same title? How is ambiguity resolved?
- What if the target entry is renamed? Do existing links break?
- Are links resolved case-sensitively?
- Is there an autocomplete for wiki-links in the editor, or must users type exact titles?

**Recommendation:** Define link resolution rules explicitly: case-insensitive matching, most-recently-updated wins on ambiguity, backlink update on rename (or not), and editor autocomplete behavior.

### 5.2 "Layer" Concept is Overloaded

The spec uses "layer" for both:
- Entry type classification (`knowledge_base` vs `project_workspace`)
- Share scope (`layer` share type)

This creates confusion: if I share my `knowledge_base` layer, does that include user-defined types I created with `layer: knowledge_base`? The spec implies yes, but this should be explicit.

**Recommendation:** Add a clarifying section on exactly what "layer" means in each context and how user-defined types interact with layer-based sharing.

### 5.3 Type Immutability vs. Evolving Needs

FR-001 says "Entry type cannot be changed after creation" and FR-002 says "Field type cannot be changed after a field is created." These are strong constraints that will frustrate users:
- A user creates entries with type "note" but later realizes they should be "article"
- A user defines a `text` field that should have been `markdown`

**Recommendation:** Add a "migrate entries" feature (even if P3) that allows converting entries from one type to another with a field mapping step. Without this, users will feel trapped.

### 5.4 Search Ranking Algorithm is Undefined

FR-011 and FR-019 say results are "ranked by relevance" but don't define:
- How semantic and full-text scores are combined in hybrid search
- Whether recency, entry status, or entry type affect ranking
- Whether the user can influence ranking (boost certain types or tags)

**Recommendation:** Define the ranking algorithm, even if it's simple (e.g., cosine similarity * 0.7 + BM25 * 0.3, with a recency decay factor).

### 5.5 Concurrent Edit Handling is Missing

The spec doesn't address what happens when:
- The same user has the entry open in two browser tabs
- A Claude MCP `update_entry` call races with a web UI save
- A version is restored while the user is editing

**Recommendation:** Define a conflict resolution strategy. At minimum: last-write-wins with a warning ("This entry was modified elsewhere. Reload to see changes?"). Optimistic locking via `updated_at` comparison would be better.

### 5.6 MCP Error Handling and Retry Semantics

The spec defines error formats but not:
- Which MCP errors are retryable vs. terminal
- Whether Claude should auto-retry on transient failures
- What the user experience is when the MCP server is unreachable
- Rate limit behavior when Claude hits the 120 req/min MCP limit

**Recommendation:** Define error categories (retryable, auth failure, validation error, server error) and expected Claude behavior for each.

---

## 6. Product Strategy Concerns

### 6.1 No Competitive Differentiation Beyond Claude Integration

If you strip away the Claude MCP integration, Stratinote is a Notion-like note-taking app with custom types and semantic search. The competitive landscape (Notion, Obsidian, Logseq, Capacities, Mem, Reflect) is extremely crowded.

**Recommendation:** Double down on what makes this unique — the AI-native knowledge capture loop. Consider:
- Proactive Claude suggestions ("Based on your recent entries, you might want to link X to Y")
- Automatic entry creation from Claude conversation transcripts
- AI-generated summaries across entries ("What are the key themes in my knowledge base this month?")
- Smart RAG context suggestions based on conversation topic

### 6.2 No Analytics or Insights for Users

There's no way for users to understand their own knowledge base:
- How many entries they've created over time
- Which topics are most covered
- Which entries are orphaned (no links)
- Knowledge gaps (topics referenced but never captured)

**Recommendation:** Add a "Knowledge Insights" view (P3) that helps users understand and improve their knowledge base.

### 6.3 No Mobile-First Consideration

NFR-016 says "responsive and usable on mobile" but the structured document editor (Tiptap with multiple field types, property strips, section labels) is inherently desktop-oriented. The mobile experience will likely be poor without dedicated mobile UX work.

**Recommendation:** Either invest in a proper mobile editor experience or explicitly scope v1 as desktop-first with a read-only mobile view. Don't promise mobile usability without designing for it.

### 6.4 No Pricing or Deployment Model

The spec doesn't address:
- Is this self-hosted, SaaS, or both?
- Who pays for Supabase, embedding API calls, and Claude API usage?
- What are the cost implications of 1,000 users with 10,000 entries each generating embeddings?

**Recommendation:** Add a deployment and cost model document. Embedding costs alone (10M entries * ~$0.0001/embedding for re-generation) could be significant. Define who bears these costs.

---

## 7. Specification Quality Issues

### 7.1 Duplicate NFR IDs

NFR-015 is defined twice: once for "Logging" and once for "Structured Editor Responsiveness" (labeled NFR-015a). This breaks traceability.

**Recommendation:** Renumber NFR-015a and ensure all IDs are unique.

### 7.2 Missing FRs in the Sequence

FR IDs jump from FR-005 to FR-006 (fine), but skip FR-030 (which is the Health Dashboard, placed under Superadmin section 9 instead of getting its own section). The numbering is inconsistent — FRs 025-029 are interspersed in section 1 and section 9 rather than being sequential.

**Recommendation:** Reorganize FR numbering to be sequential or group by feature area with clear sub-numbering (e.g., FR-001.1, FR-001.2).

### 7.3 No User Stories or Personas

The spec defines features but not who uses them. There are at least three distinct personas:
- **Solo knowledge worker** — uses Stratinote + Claude for personal PKM
- **Team lead** — shares project workspaces with team members
- **Superadmin** — manages the instance and user base

Each has different needs and priorities. The spec treats them uniformly.

**Recommendation:** Add a personas section and map features to personas. This will help prioritize and validate design decisions.

---

## Summary of Top 10 Actionable Recommendations

| # | Recommendation | Impact | Effort |
|---|---------------|--------|--------|
| 1 | **Split P1 into P0/P1** — reduce MVP scope to core CRUD + Claude MCP + basic search | Critical | Low (planning only) |
| 2 | **Add onboarding/first-run experience FR** | High | Medium |
| 3 | **Define MCP setup flow in detail** — this is the product's key differentiator | High | Low (spec work) |
| 4 | **Add concurrent edit handling** — last-write-wins at minimum | High | Medium |
| 5 | **Define wiki-link resolution rules** — ambiguity will cause bugs | High | Low (spec work) |
| 6 | **Add notification system for shares/imports** | Medium | Medium |
| 7 | **Defer superadmin web UI** — use CLI/SQL for v1 | Medium | Negative (saves effort) |
| 8 | **Promote FR-013 (Web UI entry creation) to P1** | High | Low (reprioritization) |
| 9 | **Define embedding queue operational parameters** | Medium | Low (spec work) |
| 10 | **Add user personas and map features** | Medium | Low (planning) |

---

## Conclusion

The Stratinote specification is one of the most thorough pre-implementation specs I've reviewed. The architecture is sound, the security model is robust, and the AI integration approach is genuinely innovative. However, the spec over-indexes on backend completeness and under-indexes on user experience, onboarding, and operational readiness. The P1 scope is too ambitious for an initial release and risks a long development cycle before users can validate the core value proposition.

**My strongest recommendation:** Ship a thin, vertical slice first — basic entry types, the Claude MCP integration with create/search, and a simple web editor. Validate that the Claude-powered knowledge capture loop is as magical as the vision promises. Everything else can be layered on once users confirm the core value.
