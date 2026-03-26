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

Add a metadata validation schema to:
```
src/schemas/metadata/<new-type>.schema.ts
```

Export a Zod schema. The system automatically picks it up by convention:
```ts
export const newTypeMetadataSchema = z.object({
  // your fields
});
```

### Step 4 — (Optional) Register a Claude skill

See §3 below to add a `/new-<type>` Claude skill.

### No other changes required.

The entry CRUD endpoints, search, embedding, and export functions handle all types generically.

---

## 2. Adding a New Metadata Field to an Existing Type

1. Update the template file for the type.
2. Update the Zod metadata schema for the type to add the new field (as `optional()` to maintain backward compatibility).
3. Create a Supabase migration if any new database column or index is needed.
4. Add a test case to the unit tests for the updated schema.

---

## 3. Adding a New Claude Skill

Skills are registered in a skill registry file:
```
src/skills/registry.ts
```

Each skill is an object conforming to the `Skill` interface:

```ts
interface Skill {
  name: string;              // e.g. "new-note"
  command: string;           // e.g. "/new-note"
  description: string;
  entryType?: EntryType;     // if this skill creates an entry
  handler: SkillHandler;     // async function (session, args) => SkillResult
}
```

To add a new skill:
1. Create a handler in `src/skills/handlers/<skill-name>.ts`.
2. Register it in `src/skills/registry.ts`.
3. The dispatcher automatically routes `/new-skill-name` to the handler.

No changes to the core dispatcher are needed.

---

## 4. Swapping the Embedding Model

The embedding service is abstracted behind an interface:

```ts
interface EmbeddingProvider {
  embed(text: string): Promise<number[]>;
  dimensions: number;
  modelName: string;
}
```

To add a new provider:
1. Create a new class implementing `EmbeddingProvider` in `src/embedding/providers/<provider-name>.ts`.
2. Set the environment variable:
   ```
   EMBEDDING_PROVIDER=<provider-name>
   ```
3. The provider factory (`src/embedding/factory.ts`) instantiates the correct provider at startup.

**Important:** If you change the embedding model, all existing embeddings must be regenerated. Use the admin script:
```
npm run reembed-all
```
This re-queues all entries in `embedding_queue` for reprocessing.

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

The search service is structured around named search strategies:

```ts
interface SearchStrategy {
  name: string;                     // e.g. "semantic", "fulltext", "hybrid"
  search(query: SearchQuery): Promise<SearchResult[]>;
}
```

To add a new search strategy:
1. Implement the `SearchStrategy` interface in `src/search/strategies/<name>.ts`.
2. Register it in `src/search/registry.ts`.
3. The `POST /search` endpoint accepts any registered `mode` value.

---

## 7. Feature Flags

For larger features that should be rolled out gradually, the system supports environment-variable-based feature flags:

```
FEATURE_IMPORT_EXPORT=true
FEATURE_OAUTH=false
```

Feature flags are checked at runtime via a `isFeatureEnabled(flag)` utility. They are documented in `.env.example`.

---

## 8. Versioning the API

The API is versioned under `/api/v1`. When breaking changes are needed:
1. Create a new router at `/api/v2`.
2. Implement changed endpoints in `src/api/v2/`.
3. Maintain `/api/v1` until all clients have migrated.
4. Deprecation notices are added to `/api/v1` response headers: `Deprecation: true`, `Sunset: <date>`.
