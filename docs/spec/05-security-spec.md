# Security Specification

---

## 1. Threat Model

### Assets to Protect
1. **User knowledge entries** — private intellectual content, potentially sensitive.
2. **User credentials** — passwords, session tokens.
3. **API keys** — Claude/Anthropic and embedding model API keys.
4. **User profile data** — display name, email.

### Threat Actors
| Actor | Description |
|-------|-------------|
| **Unauthenticated external attacker** | Attempts to access the API without credentials |
| **Authenticated malicious user** | Attempts to access or corrupt another user's data |
| **Compromised API key holder** | Has a leaked/stolen API key |
| **Server-side injection attacker** | Attempts SQL/command injection via input fields |

### Out of Scope
- Physical infrastructure attacks (mitigated by Supabase managed service).
- Anthropic/OpenAI service-side breaches.

---

## 2. Authentication

### SEC-001 — Supabase Auth as Identity Provider
All user authentication is delegated to Supabase Auth. The system does not implement its own password hashing or session management.

### SEC-002 — JWT Validation
Every API request must include a valid Supabase JWT in the `Authorization: Bearer` header. The backend validates the JWT signature using the Supabase JWT secret before processing any request.

### SEC-003 — Session Expiry
JWTs expire after **1 hour** (Supabase default). Refresh tokens are used to silently re-authenticate. Refresh tokens expire after **24 hours** of inactivity.

### SEC-004 — Secure Token Storage
- Web UI: JWT stored in memory only; refresh token stored in a `httpOnly`, `Secure`, `SameSite=Strict` cookie.
- No JWTs stored in `localStorage` or `sessionStorage`.

### SEC-005 — Password Policy
Enforced by Supabase Auth:
- Minimum 8 characters.
- At least one uppercase, one lowercase, one digit.

### SEC-006 — OAuth Security
When OAuth providers (Google, GitHub) are enabled:
- Redirect URIs are whitelisted and validated.
- PKCE flow is used (Supabase default for browser clients).
- State parameter is validated to prevent CSRF.

---

## 3. Authorisation and Row Level Security

### SEC-007 — RLS on All User Tables
RLS is **enabled** on `entries`, `entry_links`, `entry_embeddings`, and `profiles`. Every policy references `auth.uid()`.

**`entries` RLS policies:**
```sql
-- SELECT: user can only see their own non-deleted entries
CREATE POLICY "entries_select" ON entries
  FOR SELECT USING (user_id = auth.uid() AND deleted_at IS NULL);

-- INSERT: user can only insert entries for themselves
CREATE POLICY "entries_insert" ON entries
  FOR INSERT WITH CHECK (user_id = auth.uid());

-- UPDATE: user can only update their own entries
CREATE POLICY "entries_update" ON entries
  FOR UPDATE USING (user_id = auth.uid());

-- DELETE: user can only delete their own entries
CREATE POLICY "entries_delete" ON entries
  FOR DELETE USING (user_id = auth.uid());
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
The API sets a strict CORS policy allowing only the configured frontend origin. `Access-Control-Allow-Origin: <FRONTEND_URL>`. Wildcard `*` is forbidden.

### SEC-016 — Rate Limiting
API endpoints are rate-limited per authenticated user:
- General API: **120 requests/minute**.
- Search endpoints: **30 requests/minute**.
- Export endpoint: **5 requests/hour**.
- Import endpoint: **10 requests/hour**.

Exceeding limits returns HTTP 429 with a `Retry-After` header.

### SEC-017 — HTTPS Only
All traffic between clients and the backend, and between the backend and Supabase, must use TLS 1.2 or higher. HTTP is redirected to HTTPS.

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

### SEC-019 — Environment Variables
All secrets (Supabase URL, service role key, Anthropic API key, JWT secret) are stored as environment variables. They are never committed to source control.

### SEC-020 — Secret Rotation
API keys must be rotatable without downtime. The system supports reading the key from environment variables at runtime (no restart required for key rotation on serverless deployments).

### SEC-021 — `.env` Protection
A `.gitignore` rule ensures `.env`, `.env.local`, `.env.production` are never committed. A pre-commit hook validates this.

---

## 7. Data Privacy

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
