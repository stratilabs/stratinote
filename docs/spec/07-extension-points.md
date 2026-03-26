# Extension Points

This document describes the designed extension points of Stratinote. The goal is that common additions — new entry types, new MCP tools, new embedding models — require no code deployment for UI-driven operations, and minimal scoped changes for code-level extensions.

---

## 1. Adding a New Entry Type

### Option A — Via the Web UI (no code required)

Users add their own types through the web UI:
1. Navigate to Settings → My Entry Types → New Type.
2. Set name, slug, description, icon, colour, and layer.
3. Add field definitions using the field builder.
4. Save. The type is immediately available for entry creation and in the MCP integration.

Superadmins add system types (available to all users) via Admin → Entry Types → New System Type. Same flow, no code deployment needed.

### Option B — Via Supabase migration (for programmatic seeding)

For bootstrapping or bulk system type creation:

```sql
-- 1. Insert the type definition
INSERT INTO entry_type_definitions (slug, name, owner_type, layer)
VALUES ('research-note', 'Research Note', 'system', 'knowledge_base');

-- 2. Insert field definitions
INSERT INTO field_definitions (type_definition_id, field_key, label, field_type, display_group, "order", required)
SELECT id, 'hypothesis', 'Hypothesis', 'markdown', 'document', 1, true
FROM entry_type_definitions WHERE slug = 'research-note';
```

No application code changes are needed. The MCP `get_type_schema` tool, the structured editor, search, and export all handle new types generically.

---

## 2. Adding a New Field to an Existing Type

**Via UI:** Settings → My Entry Types → [Type] → Add Field. Choose type, label, display group, and order. New fields are always optional for existing entries.

**Via migration (system types):**
```sql
INSERT INTO field_definitions (type_definition_id, field_key, label, field_type, display_group, "order", required)
SELECT id, 'new_field', 'New Field', 'text', 'properties', 10, false
FROM entry_type_definitions WHERE slug = 'note';
```

Adding a required field to a type that already has entries: the field is treated as optional for existing entries (those entries have `schema_version` < current). Only new entries must supply it.

---

## 3. Adding a New MCP Tool

MCP tools are registered in the `mcp-server` Edge Function:
```
supabase/functions/mcp-server/tools/index.ts
```

Each tool is an object conforming to the `MCPTool` interface:

```ts
interface MCPTool {
  name: string;              // e.g. "create_note"
  description: string;       // shown to Claude to decide when to use the tool
  inputSchema: ZodSchema;    // typed input; Claude uses this to format calls
  handler: (input: unknown, userId: string) => Promise<MCPToolResult>;
}
```

To add a new tool:
1. Create a handler in `supabase/functions/mcp-server/tools/<tool-name>.ts`.
2. Register it in `supabase/functions/mcp-server/tools/index.ts`.
3. The MCP dispatcher routes tool calls by name automatically.
4. Deploy the updated Edge Function: `supabase functions deploy mcp-server`.

No other changes needed.

---

## 4. Swapping the Embedding Model

The embedding logic lives in `supabase/functions/process-embedding-queue/providers/`. It is abstracted behind an interface:

```ts
interface EmbeddingProvider {
  embed(text: string): Promise<number[]>;
  readonly dimensions: number;
  readonly modelName: string;
}
```

To add a new provider:
1. Create a new file implementing `EmbeddingProvider` in `supabase/functions/process-embedding-queue/providers/<name>.ts`.
2. Register it in the provider factory in the same directory.
3. Update Supabase project secrets:
   ```
   EMBEDDING_PROVIDER=<name>
   EMBEDDING_MODEL=<model-id>
   EMBEDDING_DIMENSIONS=<int>
   ```

**Important — model change requires a migration:**
If you change the embedding dimensions, the `entry_embeddings.embedding` column type must be updated. Run:
```sql
-- In a new Supabase migration
ALTER TABLE entry_embeddings ALTER COLUMN embedding TYPE vector(<new-dimensions>);
```
Then re-queue all entries for re-embedding:
```sql
INSERT INTO embedding_queue (entry_id, status)
SELECT id, 'pending' FROM entries WHERE deleted_at IS NULL
ON CONFLICT (entry_id) DO UPDATE SET status = 'pending', attempts = 0;
```

---

## 5. Adding a New Authentication Provider

Supabase Auth supports additional OAuth providers (GitHub, Google, Twitter, etc.) and enterprise SSO (SAML).

To add a new provider:
1. Enable it in the Supabase dashboard under Authentication → Providers.
2. Add the provider's client ID and secret as environment variables.
3. Update the web UI login page to show the new provider button.

No backend API changes are needed; Supabase handles the OAuth flow.

---

## 6. Adding a New Search Mode

The `search` Edge Function is structured around named search strategies:

```ts
interface SearchStrategy {
  name: string;
  search(query: SearchQuery, userId: string): Promise<SearchResult[]>;
}
```

To add a new search strategy:
1. Create a file in `supabase/functions/search/strategies/<name>.ts`.
2. Register it in `supabase/functions/search/strategies/index.ts`.
3. The `mode` field in the search request will now accept the new name.
4. Deploy: `supabase functions deploy search`.

---

## 7. Feature Flags

For larger features that should be rolled out gradually, Supabase project secrets act as feature flags:

```
FEATURE_IMPORT_EXPORT=true
FEATURE_OAUTH=false
```

Edge Functions read these at runtime via `Deno.env.get('FEATURE_X')`. Frontend flags are prefixed `NEXT_PUBLIC_FEATURE_X` in `.env.local`. All flags are documented in `.env.example`.

---

## 8. Versioning Edge Functions

Edge Functions are versioned by deploying updated code:
- Breaking changes to an Edge Function's API should be introduced as a new function name (e.g. `search-v2`).
- The old function remains deployed until all clients migrate.
- Deprecation is communicated via a `Deprecation: true` response header and a documented sunset date.
- PostgREST schema changes follow the Supabase migration versioning system (`supabase/migrations/`).
