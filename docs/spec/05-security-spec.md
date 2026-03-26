# Security Specification

---

## 1. Threat Model

### Assets to Protect
1. **User knowledge entries** — private intellectual content, potentially sensitive. Sharing is explicit and consent-based; no entry is accessible to another user without the owner's explicit share grant.
2. **User credentials** — passwords, session tokens, refresh tokens.
3. **Personal API Tokens** — grant full user-scoped access; treated as secrets.
4. **Service API keys** — Anthropic API key, embedding provider key, Supabase service role key.
5. **User profile data** — display name, email.
6. **Share grants** — control which spaces are readable by whom; must not be creatable or revocable by anyone other than the grantor.

### Threat Actors
| Actor | Description |
|-------|-------------|
| **Unauthenticated external attacker** | Attempts to access the API without credentials |
| **Authenticated malicious user** | Attempts to access or corrupt another user's data |
| **Compromised PAT holder** | Has a leaked Personal API Token (e.g. from Claude.ai integration settings) |
| **Leaked service key holder** | Has a leaked Anthropic or Supabase service role key |
| **Server-side injection attacker** | Attempts SQL/command/YAML injection via input fields |
| **Brute-force attacker** | Attempts to guess passwords or enumerate valid user emails |

### Out of Scope
- Physical infrastructure attacks (mitigated by Supabase managed service).
- Anthropic/OpenAI service-side breaches.

---

## 2. Authentication

### SEC-001 — Supabase Auth as Identity Provider
All user authentication is delegated to Supabase Auth. The system does not implement its own password hashing or session management.

### SEC-002 — Dual Authentication Methods
The API accepts two authentication methods, validated in this order:

1. **Supabase JWT** — issued by Supabase Auth for web UI sessions. Validated by checking the JWT signature against the Supabase JWT secret.
2. **Personal API Token (PAT)** — a bearer token prefixed `sn_`. Validated by SHA-256 hashing the token and looking up the hash in the `api_tokens` table. The resolved user identity is used for all subsequent RLS checks.

A request missing both returns HTTP 401. PAT creation (`POST /auth/tokens`) requires a Supabase JWT specifically (cannot be done with a PAT).

### SEC-003 — Session Expiry
JWTs expire after **1 hour** (Supabase default). Refresh tokens are used to silently re-authenticate. Refresh tokens expire after **24 hours** of inactivity.

### SEC-004 — Secure Token Storage
- Web UI: JWT stored in memory only; refresh token stored in a `httpOnly`, `Secure`, `SameSite=Strict` cookie.
- No JWTs stored in `localStorage` or `sessionStorage`.
- Personal API Tokens: shown once in the web UI immediately after creation. The plaintext is never stored server-side; only the SHA-256 hash is persisted.

### SEC-004a — CSRF Protection
Cookie-based auth (refresh token) is protected against CSRF as follows:
- `SameSite=Strict` prevents the cookie from being sent on cross-origin requests in modern browsers.
- All state-mutating API requests (`POST`, `PATCH`, `DELETE`) additionally require a `X-Requested-With: XMLHttpRequest` header, which browsers cannot set cross-origin without a CORS preflight (which the strict CORS policy will block).
- The Supabase JWT itself (stored in memory, not cookies) is required for API calls, which cannot be stolen via CSRF.

### SEC-005 — Password Policy
Enforced by Supabase Auth:
- Minimum 8 characters.
- At least one uppercase, one lowercase, one digit.

### SEC-005a — Brute-Force and Account Lockout
- Supabase Auth rate-limits sign-in attempts: maximum **5 failed attempts per 15 minutes** per email address; configured in the Supabase Auth settings.
- After 5 failures, the account is temporarily locked and the user receives an email to unlock.
- Auth API endpoints (`/auth/v1/token`, `/auth/v1/signup`) are additionally rate-limited at the edge (30 requests/minute per IP).
- Sign-in responses for invalid credentials return a generic message ("Invalid email or password") — no enumeration of whether the email exists.

### SEC-006 — OAuth Security
When OAuth providers (Google, GitHub) are enabled:
- Redirect URIs are whitelisted and validated.
- PKCE flow is used (Supabase default for browser clients).
- State parameter is validated to prevent CSRF.

---

## 3. Authorisation and Row Level Security

### SEC-007 — Superadmin Role
Superadmin status is stored in `profiles.is_superadmin`. This flag:
- Can only be set via a direct database migration or a CLI script run by an operator — never via any API endpoint.
- Is checked in every Edge Function that handles admin operations by reading the `profiles` row for the authenticated user.
- Is additionally enforced by RLS policies on `entry_type_definitions` (system types), `invites`, and `audit_log`.

The JWT for a superadmin session is indistinguishable from a regular user JWT; the superadmin check is always a database read, never a claim in the token.

### SEC-007a — RLS on All User Tables
RLS is **enabled** on `entries`, `entry_links`, `entry_embeddings`, `entry_type_definitions`, `field_definitions`, `api_tokens`, `profiles`, `invites`, `audit_log`, `rag_contexts`, `entry_versions`, and `space_shares`. Every policy references `auth.uid()`.

**`entries` RLS policies:**
```sql
-- SELECT: user sees their own non-deleted entries, plus entries in shared spaces
CREATE POLICY "entries_select" ON entries
  FOR SELECT USING (
    deleted_at IS NULL AND (
      user_id = auth.uid()
      OR EXISTS (
        SELECT 1 FROM space_shares
        WHERE grantee_id = auth.uid()
          AND grantor_id = entries.user_id
          AND (
            (share_type = 'layer'   AND space_key = entries.layer)
            OR (share_type = 'project' AND space_key::uuid = entries.project_id)
          )
      )
    )
  );

-- INSERT: user can only insert entries for themselves
CREATE POLICY "entries_insert" ON entries
  FOR INSERT WITH CHECK (user_id = auth.uid());

-- UPDATE: user can only update their own entries (shared entries are read-only)
CREATE POLICY "entries_update" ON entries
  FOR UPDATE USING (user_id = auth.uid());

-- DELETE: user can only delete their own entries
CREATE POLICY "entries_delete" ON entries
  FOR DELETE USING (user_id = auth.uid());
```

**`space_shares` RLS policies:**
```sql
-- SELECT: both parties can see the share
CREATE POLICY "shares_select" ON space_shares
  FOR SELECT USING (grantor_id = auth.uid() OR grantee_id = auth.uid());

-- INSERT: only the grantor can create shares (re-share validation done in Edge Function)
CREATE POLICY "shares_insert" ON space_shares
  FOR INSERT WITH CHECK (grantor_id = auth.uid());

-- DELETE: only the grantor can revoke shares
CREATE POLICY "shares_delete" ON space_shares
  FOR DELETE USING (grantor_id = auth.uid());
```

**`entry_links` RLS policies:**
```sql
-- SELECT: user can see links where they own both entries
CREATE POLICY "links_select" ON entry_links
  FOR SELECT USING (
    EXISTS (SELECT 1 FROM entries WHERE id = source_entry_id AND user_id = auth.uid())
  );

-- INSERT: user can only create links between their own entries
CREATE POLICY "links_insert" ON entry_links
  FOR INSERT WITH CHECK (
    EXISTS (SELECT 1 FROM entries WHERE id = source_entry_id AND user_id = auth.uid()) AND
    EXISTS (SELECT 1 FROM entries WHERE id = target_entry_id AND user_id = auth.uid())
  );

-- DELETE: user can only remove links they own
CREATE POLICY "links_delete" ON entry_links
  FOR DELETE USING (
    EXISTS (SELECT 1 FROM entries WHERE id = source_entry_id AND user_id = auth.uid())
  );
```

**`entry_embeddings` RLS policies:**
```sql
-- Embeddings are accessed via service role only from backend;
-- direct user access is blocked entirely.
CREATE POLICY "embeddings_no_direct_access" ON entry_embeddings
  FOR ALL USING (false);
```

### SEC-008 — Service Role Key
The service role key (bypasses RLS) is used **only** server-side for:
- Embedding generation and storage.
- Embedding queue processing.

It is **never** exposed to the browser or included in any client-side bundle.

### SEC-009 — API Layer Authorisation
Even though RLS provides database-level enforcement, the API layer performs an additional ownership check before returning data. Defence in depth.

---

## 4. Input Validation and Injection Prevention

### SEC-010 — Schema Validation
All API request bodies are validated against a strict JSON schema (e.g. using Zod) before any database interaction. Unknown fields are stripped.

### SEC-011 — SQL Injection
The system uses parameterised queries exclusively via the Supabase client SDK. Raw SQL strings with user input are prohibited.

### SEC-012 — Markdown Sanitisation
Markdown body content rendered in the browser is sanitised before rendering to prevent XSS (e.g. using `DOMPurify` or a safe Markdown renderer that disallows raw HTML). Raw HTML in Markdown is stripped by default.

### SEC-013 — YAML Front-Matter Injection
YAML parsing uses a safe-load parser (e.g. `js-yaml` in `SAFE_LOAD` mode) to prevent code execution via YAML deserialization.

### SEC-014 — File Upload Validation (Import)
Imported files must:
- Have a `.md` extension.
- Not exceed 1 MB per file.
- Be validated as valid UTF-8 text before parsing.

---

## 5. API Security

### SEC-015 — CORS Policy
Supabase Edge Functions set a strict CORS policy allowing only the configured frontend origin. `Access-Control-Allow-Origin: <FRONTEND_URL>`. Wildcard `*` is forbidden. The Supabase project's allowed origins are configured in the Supabase dashboard.

### SEC-016 — Rate Limiting
Edge Functions enforce rate limiting per authenticated user (see API Specification §4). PostgREST rate limits are configured at the Supabase project level.

Exceeding limits returns HTTP 429 with a `Retry-After` header.

### SEC-017 — HTTPS Only
All Supabase endpoints (PostgREST, Auth, Edge Functions) are HTTPS-only by default. The Next.js frontend must be deployed with HTTPS enforced. HTTP requests are redirected to HTTPS.

### SEC-018 — Security Headers
The web application sets the following HTTP headers:
```
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

---

## 6. Secrets Management

### SEC-019 — Secrets Storage
- **Edge Function secrets** (Anthropic API key, embedding provider key, `EMBEDDING_DIMENSIONS`, etc.) are stored as Supabase project secrets, accessible only by Edge Functions at runtime. They are never visible in the dashboard after creation.
- **Frontend environment variables** (`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`) are safe to expose publicly; they are scoped by RLS.
- The Supabase **service role key** is stored only as a Supabase project secret for use by Edge Functions. It is never included in the frontend bundle.

### SEC-020 — Secret Rotation
Supabase project secrets can be updated without redeploying Edge Functions (they are read at cold start). Rotating the Anthropic API key or embedding key requires updating the Supabase secret — Edge Functions will pick it up on next invocation.

### SEC-021 — `.env` Protection
A `.gitignore` rule ensures `.env`, `.env.local`, `.env.production` are never committed. A pre-commit hook validates this. Only `.env.example` (with placeholder values) is committed.

---

## 7. Audit Logging

### SEC-022a — Audit Log for Sensitive Operations
The following operations are recorded in an `audit_log` table (service role only; not user-readable via API):

| Event | Logged Data |
|-------|-------------|
| PAT created | user_id (hashed), token_prefix, timestamp |
| PAT revoked | user_id (hashed), token_prefix, timestamp |
| Entry hard-deleted | user_id (hashed), entry_id, timestamp |
| Knowledge base exported | user_id (hashed), entry count, timestamp |
| Knowledge base imported | user_id (hashed), file count, timestamp |
| Failed PAT authentication | token_prefix (if recognisable), IP (hashed), timestamp |
| Space share created | grantor_id (hashed), grantee_id (hashed), share_type, space_key, timestamp |
| Space share revoked | grantor_id (hashed), share_id, share_type, space_key, timestamp |

No entry content or PII is written to the audit log.

---

## 8. Dependency Security

### SEC-022b — Dependency Vulnerability Scanning
- Dependabot is enabled on the repository to alert on known CVEs in npm dependencies.
- `npm audit` runs as a required CI step; builds fail on high or critical severity findings.
- Supabase client and Edge Function dependencies are reviewed on each version bump.

---

## 9. Data Privacy

### SEC-022 — Data Minimisation
The system does not collect user data beyond what is functionally required (email, display name, entries).

### SEC-023 — No PII in Logs
Logs must not contain user email, entry content, or any personally identifiable information. User references in logs use hashed user IDs.

### SEC-024 — Third-Party Data Sharing
Entry content is sent to the Anthropic API for embedding and AI processing. Users must be informed of this in the privacy policy. No other third parties receive entry content.

### SEC-025 — Data Deletion
On account deletion, all user data (entries, embeddings, links, profile) is permanently deleted within 30 days. The process is automated via a database function triggered on `auth.users` deletion.

---

## 8. Security Testing Requirements

See [Test Specification §5](./06-test-spec.md#5-security-tests) for security-specific test cases.
