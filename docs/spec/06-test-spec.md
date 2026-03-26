# Test Specification

---

## 1. Test Strategy

### Layers

| Layer | Scope | Tooling |
|-------|-------|---------|
| **Unit** | Edge Function logic: validators, parsers, search strategies, PAT hashing | Deno test (`deno test`) or Vitest |
| **Integration** | Edge Functions + PostgREST against a local Supabase instance (`supabase start`) | Vitest + `fetch` |
| **RLS** | Database-level policy tests using `supabase test db` (pgTAP) | pgTAP |
| **E2E** | Full user flows in the browser | Playwright |
| **Security** | Auth bypass, cross-user RLS, injection, rate limiting | Vitest integration tests + manual |
| **Performance** | Load and latency tests | k6 |

### Principles
- Edge Function unit tests are co-located with the function code (`*.test.ts`).
- Integration and RLS tests run against a local Supabase instance started with `supabase start`. Data is seeded and reset between test suites via SQL fixtures.
- No production data is used in tests.
- Each test is independent and does not rely on execution order.
- CI pipeline: unit and integration tests run on every PR; E2E tests run on merge to main.
- Test IDs (`T-XXX`) correspond to functional requirements (`FR-XXX`) where applicable.

---

## 2. Unit Tests

### T-U001 — Entry Type Validation (FR-001, FR-002)
```
GIVEN an entry creation payload
WHEN the `type` field is missing or invalid
THEN a VALIDATION_ERROR is returned with field details
```

### T-U002 — Metadata Schema Validation per Type (FR-002)
```
GIVEN a `book` entry payload
WHEN `metadata.book_author` is missing
THEN a VALIDATION_ERROR is returned

GIVEN a `meeting` entry payload
WHEN `project_id` is null
THEN a VALIDATION_ERROR is returned
```

### T-U003 — Wiki-Link Resolution (FR-004)
```
GIVEN entry body containing [[Some Note Title]]
WHEN the title resolves to an existing entry ID
THEN the link is stored as a resolved entry_link

GIVEN entry body containing [[Nonexistent Title]]
WHEN no entry matches the title
THEN the link is flagged as unresolved (warning, not error)
```

### T-U004 — Soft Delete (FR-003)
```
GIVEN an existing entry
WHEN DELETE /entries/:id is called (no hard flag)
THEN entry.status = 'deleted' AND entry.deleted_at IS NOT NULL
AND the entry does not appear in GET /entries
```

### T-U005 — Hard Delete (FR-003)
```
GIVEN a soft-deleted entry
WHEN DELETE /entries/:id?hard=true is called
THEN the entry row is permanently removed from the database
AND all associated entry_links and entry_embeddings are removed
```

### T-U006 — Template Retrieval (FR-002)
```
GIVEN a request to GET /templates/note
THEN a markdown template string is returned
AND it contains a valid YAML front-matter block with type: note

GIVEN a request to GET /templates/unknown_type
THEN HTTP 404 is returned
```

### T-U007 — updated_at Trigger (Data Model)
```
GIVEN an existing entry
WHEN the body is updated via PATCH /entries/:id
THEN entry.updated_at > entry.created_at
```

### T-U008 — Entry Link Uniqueness (Data Model)
```
GIVEN an existing link from entry A to entry B
WHEN POST /entries/A/links with target B is called again
THEN HTTP 409 CONFLICT is returned
```

### T-U009 — Self-Link Prevention (Data Model)
```
GIVEN entry A
WHEN POST /entries/A/links with target A is called
THEN HTTP 422 VALIDATION_ERROR is returned
```

### T-U010 — Rate Limiting Headers (NFR / SEC-016)
```
GIVEN an authenticated user
WHEN they exceed 120 API requests per minute
THEN HTTP 429 is returned with a Retry-After header
```

---

## 3. Integration Tests

> Integration tests call PostgREST and Edge Functions running on `supabase start` (local) using a test user JWT.

### T-I001 — Full Entry Lifecycle (FR-003)
```
GIVEN an authenticated user
WHEN they create, read, update, and soft-delete an entry via PostgREST
THEN each operation returns the expected response and database state
```

### T-I002 — Pagination (FR-010)
```
GIVEN a user with 50 entries
WHEN GET /entries?limit=20 is called
THEN 20 entries are returned with a next_cursor
WHEN GET /entries?cursor=<cursor>&limit=20 is called
THEN the next 20 entries are returned (non-overlapping)
```

### T-I003 — Tag Filtering (FR-010)
```
GIVEN entries with tags ["ai", "notes"] and ["books", "ai"]
WHEN GET /entries?tags=ai is called
THEN both entries are returned

WHEN GET /entries?tags=ai,books is called
THEN only the entry with both tags is returned
```

### T-I004 — Project Workspace Filtering (FR-005, FR-014)
```
GIVEN a project entry P and a meeting entry M with project_id=P.id
WHEN GET /entries?project_id=P.id is called
THEN entry M is returned
```

### T-I005 — Backlinks (FR-004)
```
GIVEN entry A linked to entry B
WHEN GET /entries/B/backlinks is called
THEN entry A appears in the response
```

### T-I006 — Full-Text Search (FR-011, FR-019)
```
GIVEN entries with bodies containing "machine learning" and "deep learning"
WHEN POST /search with query "machine learning" and mode "fulltext"
THEN both entries are returned
AND the entry containing "machine learning" has a higher score
```

### T-I007 — Semantic Search (FR-011, FR-019)
```
GIVEN entries with embeddings stored
WHEN POST /search with query "neural networks" and mode "semantic"
THEN entries semantically related to "neural networks" are returned
AND entries are ordered by descending score
```

### T-I008 — Embedding Queue (FR-018)
```
GIVEN a new entry is created
THEN a row appears in embedding_queue with status='pending'
WHEN the embedding worker processes the queue
THEN an embedding row exists in entry_embeddings for the entry
AND the queue row status becomes 'done'
```

### T-I009 — Export (FR-020)
```
GIVEN a user with 3 entries of different types
WHEN GET /export is called
THEN a ZIP file is returned
AND the ZIP contains 3 .md files in folders named by type
AND each file contains valid front-matter
```

### T-I010a — PAT Create, Use, and Revoke (FR-022, FR-023)
```
GIVEN an authenticated user (Supabase JWT)
WHEN they call manage-tokens Edge Function with action=create
THEN a token is returned in plaintext (one time only)
AND the token_hash is stored in api_tokens

WHEN they call the search Edge Function with the PAT as Bearer token
THEN the call succeeds and returns results scoped to their entries

WHEN they revoke the PAT via action=revoke
THEN subsequent calls with that PAT return HTTP 401

WHEN they call manage-tokens with action=create using a PAT (not JWT)
THEN HTTP 401 is returned
```

---

### T-I010b — Embedding Queue Processing (FR-018)
```
GIVEN a new entry is created via PostgREST
THEN a row in embedding_queue has status='pending' within 1 second (trigger fires)

WHEN the process-embedding-queue Edge Function is invoked
THEN the queue row transitions to status='done'
AND an embedding row exists in entry_embeddings for the entry

GIVEN the embedding provider API returns an error
WHEN the Edge Function processes the queue item
THEN attempts is incremented and status remains 'pending' (up to 3 retries)
AFTER 3 failures THEN status='failed' and last_error is populated
```

---

### T-I010 — Import (FR-021)
```
GIVEN a valid .md file with correct front-matter
WHEN POST /import is called with the file
THEN HTTP 202 is returned with imported: 1

GIVEN a .md file missing required front-matter field `type`
WHEN POST /import is called
THEN the file is reported in the errors array with the reason
```

---

## 4. End-to-End Tests

### T-E001 — User Registration and Login (FR-015)
```
GIVEN an unregistered email
WHEN the user completes the registration form
THEN they are redirected to the main dashboard

WHEN the user logs out and logs back in
THEN they see their dashboard with existing entries
```

### T-E002 — Create Entry via Web UI (FR-013)
```
GIVEN an authenticated user on the dashboard
WHEN they select "New Note" and fill in the form
AND click Save
THEN the new entry appears in the entry list
AND the entry page shows the correct title and body
```

### T-E003 — Edit Entry in Web UI (FR-012)
```
GIVEN an existing entry
WHEN the user opens the editor, changes the body, and saves
THEN the updated body is reflected immediately
AND the updated_at timestamp is newer than before
```

### T-E004 — Unsaved Changes Warning (FR-012)
```
GIVEN an entry with unsaved edits in the editor
WHEN the user attempts to navigate away
THEN a confirmation dialog is shown
WHEN the user confirms navigation
THEN the page changes without saving
```

### T-E005 — Search via Web UI (FR-011)
```
GIVEN entries with diverse content
WHEN the user types a query in the search box
THEN relevant results appear with highlighted excerpts

WHEN the user toggles semantic search
THEN results re-order based on semantic relevance
```

### T-E006 — Project Workspace View (FR-014)
```
GIVEN a project entry with linked meetings
WHEN the user clicks the project entry
THEN the project workspace view shows all linked entries
WHEN the user creates a new meeting from this view
THEN the meeting's project_id is pre-filled with the project's ID
```

### T-E007 — Wiki-Link Rendering (FR-004)
```
GIVEN an entry body containing [[Another Entry Title]]
WHEN the entry is viewed in the web UI
THEN [[Another Entry Title]] renders as a clickable hyperlink
WHEN clicked
THEN the user is navigated to the linked entry
```

---

## 5. Security Tests

### T-S001 — Unauthenticated Access (SEC-002)
```
GIVEN no Authorization header
WHEN any /api/v1/* endpoint is called
THEN HTTP 401 is returned
```

### T-S002 — Cross-User Entry Access (FR-016, SEC-007)
```
GIVEN user A with entry A1 and user B with a valid JWT
WHEN user B calls GET /entries/A1.id
THEN HTTP 404 is returned (not 403, to avoid confirming existence)
```

### T-S003 — Cross-User Entry Update (FR-016, SEC-007)
```
GIVEN user A with entry A1 and user B with a valid JWT
WHEN user B calls PATCH /entries/A1.id
THEN HTTP 404 is returned
AND the database row is unchanged
```

### T-S004 — Cross-User Entry Delete (FR-016, SEC-007)
```
GIVEN user A with entry A1 and user B with a valid JWT
WHEN user B calls DELETE /entries/A1.id
THEN HTTP 404 is returned
AND the entry still exists for user A
```

### T-S005 — Cross-User Search Isolation (FR-016, SEC-007)
```
GIVEN user A with entries and user B with different entries
WHEN user B calls POST /search with a query matching user A's entries only
THEN no results are returned for user B
```

### T-S006 — Cross-User Link Creation (SEC-007)
```
GIVEN user A's entry A1 and user B's entry B1
WHEN user B calls POST /entries/B1.id/links with target A1.id
THEN HTTP 404 or 422 is returned
AND no link is created
```

### T-S007 — SQL Injection via Query Params (SEC-011)
```
GIVEN a malicious query string in ?type='; DROP TABLE entries; --
WHEN GET /entries is called
THEN HTTP 422 is returned (type fails enum validation)
AND no database error occurs
```

### T-S008 — XSS via Entry Body (SEC-012)
```
GIVEN entry body containing <script>alert('xss')</script>
WHEN the entry is rendered in the web UI
THEN the script tag is sanitised and not executed
```

### T-S009 — Expired Token (SEC-003)
```
GIVEN an expired JWT
WHEN any API endpoint is called
THEN HTTP 401 is returned
```

### T-S010 — Rate Limit Enforcement (SEC-016)
```
GIVEN an authenticated user
WHEN they send 31 POST /search requests in 60 seconds
THEN the 31st request returns HTTP 429 with Retry-After header
```

### T-S011 — Service Role Key Not Exposed (SEC-008)
```
GIVEN the frontend web application bundle
WHEN it is inspected for strings matching SUPABASE_SERVICE_ROLE_KEY pattern
THEN no match is found
```

### T-S012 — HTTPS Enforcement (SEC-017)
```
GIVEN an HTTP request to the API
WHEN the request is made
THEN it is redirected to HTTPS with HTTP 301
```

---

## 6. Failure and Degradation Tests

### T-F001 — Embedding Service Unavailable (NFR-010)
```
GIVEN the embedding provider API is unavailable (simulated via env var override)
WHEN a new entry is created via PostgREST
THEN the entry is persisted successfully (HTTP 201)
AND the embedding_queue row is marked status='pending'
AND the web UI displays the entry (without semantic search availability)
AND no error is surfaced to the user during creation
```

### T-F002 — MCP Server Unavailable
```
GIVEN the mcp-server Edge Function is down
WHEN the user calls a tool from Claude.ai
THEN Claude.ai surfaces a connection error (not a data corruption)
AND all existing entries remain intact in the database
```

---

## 7. Performance Tests

> Performance tests use **k6**. Scripts live in `tests/performance/`.

### T-P001 — PostgREST Response Time (NFR-001)
```
GIVEN 1,000 sequential requests to list entries (PostgREST)
THEN 95% of responses complete in under 500ms
```

### T-P002 — Semantic Search Latency (NFR-002)
```
GIVEN a user with 10,000 entries and embeddings
WHEN the search Edge Function is called with mode=semantic
THEN the response completes in under 2,000ms
```

### T-P003 — Concurrent Users (NFR-005)
```
GIVEN 100 simulated concurrent users each reading and writing entries
THEN no requests timeout and response times remain within NFR-001 bounds
```

### T-P004 — Embedding Queue Throughput
```
GIVEN 500 entries in embedding_queue with status='pending'
WHEN the process-embedding-queue Edge Function runs
THEN all items are processed within 10 minutes (accounting for provider rate limits)
```
