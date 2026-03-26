# Non-Functional Requirements

Each requirement is identified by `NFR-XXX`.

---

## 1. Performance

### NFR-001 — API Response Time
The API shall respond to 95% of requests within **500ms** under normal load (single user, corpus ≤ 10,000 entries), excluding embedding generation calls.

### NFR-002 — Semantic Search Latency
Semantic search queries shall return results within **2 seconds** for a corpus of up to 10,000 entries per user.

### NFR-003 — Web UI Load Time
The web application initial page load shall complete within **3 seconds** on a standard broadband connection (10 Mbps+).

### NFR-004 — Embedding Generation
Embedding generation shall not block entry save responses. The entry is persisted immediately; embedding is generated asynchronously within **30 seconds** of creation.

---

## 2. Scalability

### NFR-005 — User Scalability
The system architecture shall support up to **1,000 concurrent users** without degradation, achieved primarily through Supabase's managed scaling.

### NFR-006 — Entry Volume
The system shall perform within NFR-001 and NFR-002 bounds for users with up to **10,000 entries** each.

### NFR-007 — Stateless Edge Functions
All Supabase Edge Functions are stateless by design. Scaling is handled by Supabase's managed infrastructure.

---

## 3. Reliability

### NFR-008 — Availability
The system shall target **99.5% uptime**, consistent with Supabase's managed service SLA.

### NFR-009 — Data Durability
All entries shall be persisted to the database before a success response is returned to the client. Embedding failures shall not cause data loss.

### NFR-010 — Graceful Degradation
If the Claude API or embedding service is unavailable, the system shall:
- Still allow entry creation, update, and read via the web UI.
- Queue embedding re-generation for when the service recovers.
- Display a clear notice in the UI and Claude session about reduced AI functionality.

---

## 4. Maintainability

### NFR-011 — Code Coverage
All non-trivial business logic in Edge Functions shall have unit test coverage of at least **80%**. Critical paths (auth/PAT validation, RLS, entry CRUD, search) shall have integration test coverage.

### NFR-012 — Type Safety
All Edge Function and frontend code shall be written in TypeScript with strict mode enabled. Any `any` types shall require explicit justification via a comment.

### NFR-013 — Linting and Formatting
All code shall pass ESLint and Prettier checks as part of the CI pipeline. Commits that fail these checks shall not be merged.

### NFR-014 — Environment Configuration
All environment-specific values (Supabase URL, service role key, Anthropic API key, embedding model key) shall be managed as Supabase project secrets (for Edge Functions) and as `.env.local` variables (for the Next.js frontend). No secrets shall be committed to the repository.

### NFR-015 — Logging
Edge Functions shall emit structured JSON logs for all requests and errors, including: timestamp, request ID, user ID (hashed), function name, status code, and duration. No PII shall appear in logs. Logs are accessible via the Supabase dashboard.

---

## 5. Usability

### NFR-016 — Mobile Compatibility
The web UI shall be responsive and usable on mobile screens (≥ 375px width).

### NFR-017 — Keyboard Navigation
Core web UI actions (search, navigate entries, open editor, save) shall be accessible via keyboard shortcuts.

### NFR-018 — Accessibility
The web UI shall conform to WCAG 2.1 Level AA standards.

---

## 6. Extensibility

### NFR-019 — Entry Type Registry
New entry types shall be addable by defining a template file and schema — no changes to core API logic required. See [Extension Points](./07-extension-points.md).

### NFR-020 — Claude Skills Registry
New Claude skills shall be addable by registering a new skill definition — no changes to the core skill dispatcher required.

### NFR-021 — Embedding Model Abstraction
The embedding model shall be configurable via a Supabase project secret (`EMBEDDING_PROVIDER`, `EMBEDDING_MODEL`, `EMBEDDING_DIMENSIONS`), with an adapter interface in the `process-embedding-queue` Edge Function enabling provider swap without changing business logic. Changing the model requires a re-embedding migration (see Extension Points).
