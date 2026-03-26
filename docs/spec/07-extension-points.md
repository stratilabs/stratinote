# Extension Points

This document describes the designed extension points of Stratinote. The goal is that common additions — new entry types, new Claude skills, new embedding models — require no changes to core logic, only the addition of new configuration or module files.

---

## 1. Adding a New Entry Type

### Step 1 — Define the template

Create a new Markdown template file at:
```
src/templates/<new-type>.md
```

The file must contain a YAML front-matter block with at minimum:
```yaml
---
type: <new-type>
title: ""
tags: []
status: draft
created_at: ""
updated_at: ""
author_id: ""
related_entries: []
---
```

Add any type-specific fields to the front-matter.

### Step 2 — Register the type in the schema

Add the new type to the `entry_type` enum in a new Supabase migration:
```sql
ALTER TYPE entry_type ADD VALUE '<new-type>';
```

### Step 3 — Define the metadata schema

Add a metadata validation schema to the `mcp-server` Edge Function:
```
supabase/functions/mcp-server/schemas/<new-type>.ts
```

Export a Zod schema:
```ts
export const newTypeMetadataSchema = z.object({
  // your type-specific fields
});
```

Register it in `supabase/functions/mcp-server/schemas/index.ts`:
```ts
export const metadataSchemas: Record<EntryType, ZodSchema> = {
  // ...existing
  'new-type': newTypeMetadataSchema,
};
```

### Step 4 — (Optional) Add an MCP tool

See §3 below to add a new MCP tool for the type (e.g. a specialised creation flow).

### No other changes required.

PostgREST handles CRUD generically. The `search`, `export`, and `import` Edge Functions handle all types without modification.

---

## 2. Adding a New Metadata Field to an Existing Type

1. Update the template row in the `templates` table via a Supabase migration.
2. Update the Zod schema for the type in the `mcp-server` Edge Function (add new field as `.optional()` for backward compatibility).
3. Create a Supabase migration if a new index is needed for the field.
4. Add a test case to the Edge Function unit tests.

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
