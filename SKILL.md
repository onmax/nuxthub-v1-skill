---
name: nuxthub-migration
description: Migrate NuxtHub projects to v0.9.X self-hosting. Handles GitHub Actions removal, wrangler.jsonc creation, Workers CI setup. Optional Phase 2 for experimental v1/nightly with multi-cloud support (database/blob not production-ready).
---

# NuxtHub Migration

Two-phase migration path. Phase 1 is stable and recommended. Phase 2 is experimental.

## Phase 1: Self-Hosting (v0.9.X) - RECOMMENDED

Migrate from NuxtHub Admin / GitHub Actions to self-hosted Cloudflare Workers. No code changes required.

### 1.1 Remove Deprecated Deployment

Delete `.github/workflows/nuxthub.yml` or any NuxtHub-specific GitHub Action.

Remove deprecated env vars from CI/CD and `.env`:
- `NUXT_HUB_PROJECT_KEY`
- `NUXT_HUB_PROJECT_DEPLOY_TOKEN`

### 1.2 Create Cloudflare Resources

Create resources based on enabled features:

```bash
# D1 Database (if hub.database: true)
npx wrangler d1 create my-app-db

# KV Namespace (if hub.kv: true)
npx wrangler kv namespace create KV

# KV Namespace for cache (if hub.cache: true)
npx wrangler kv namespace create CACHE

# R2 Bucket (if hub.blob: true)
npx wrangler r2 bucket create my-app-bucket
```

### 1.3 Create wrangler.jsonc

Create `wrangler.jsonc` in project root. See `references/wrangler-templates.md` for full examples.

Minimal example with database:
```jsonc
{
  "$schema": "node_modules/wrangler/config-schema.json",
  "name": "my-app",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [{ "binding": "DB", "database_name": "my-app-db", "database_id": "<from-wrangler-output>" }]
}
```

Required binding names:
| Feature | Binding | Type |
|---------|---------|------|
| Database | `DB` | D1 |
| KV | `KV` | KV Namespace |
| Cache | `CACHE` | KV Namespace |
| Blob | `BLOB` | R2 Bucket |

### 1.4 Set Up Workers Builds CI/CD

In Cloudflare Dashboard:
1. Workers & Pages → Create → Import from Git
2. Connect GitHub/GitLab repository
3. Set build command: `npm run build` (or `pnpm build`)
4. Set output directory: `.output`

### 1.5 Configure Environment Variables (Optional)

For advanced features (blob presigned URLs, cache DevTools, AI):

```bash
NUXT_HUB_CLOUDFLARE_ACCOUNT_ID=<account-id>
NUXT_HUB_CLOUDFLARE_API_TOKEN=<token-with-appropriate-permissions>
# Feature-specific (as needed):
NUXT_HUB_CLOUDFLARE_BUCKET_ID=<bucket-id>
NUXT_HUB_CLOUDFLARE_CACHE_NAMESPACE_ID=<namespace-id>
```

### 1.6 Test Remote Development

```bash
npx nuxt dev --remote
```

### Phase 1 Complete

No changes to:
- `nuxt.config.ts` (`hub.database: true` stays as-is)
- `server/database/` directory structure
- `hubDatabase()` API usage
- Package (`@nuxthub/core`)

---

## Phase 2: v1/Nightly - EXPERIMENTAL

> **WARNING**: Database and blob features are not production-ready. Only proceed if explicitly requested.

Multi-cloud support (Cloudflare, Vercel, Deno, Netlify). Breaking changes from v0.9.X.

### 2.1 Update Package

```bash
pnpm remove @nuxthub/core
pnpm add @nuxthub/core-nightly
```

### 2.2 Update nuxt.config.ts

**Before (v0.9.X):**
```ts
hub: { database: true, kv: true, blob: true, cache: true }
```

**After (v1):**
```ts
hub: { db: 'sqlite', kv: true, blob: true, cache: true }
```

Key change: `database: true` → `db: '<dialect>'` (`sqlite` | `postgresql` | `mysql`)

### 2.3 Rename Database Directory

```bash
mv server/database server/db
```

Update imports: `~/server/database/` → `~/server/db/`

Migrations generated via `npx nuxt db generate` go to `server/db/migrations/{dialect}/`.

### 2.4 Migrate API (hubDatabase → Drizzle)

**Before:**
```ts
const db = hubDatabase()
const users = await db.prepare('SELECT * FROM users').all()
```

**After:**
```ts
const db = useDrizzle()
const users = await db.select().from(tables.users)
```

### 2.5 Provider-Specific Setup (Non-Cloudflare)

#### Vercel
- **KV**: Add Redis via dashboard, install `ioredis`
- **Blob**: Add Blob Store via dashboard, install `@vercel/blob`
- **Cache**: Auto-configured

#### Deno Deploy
- **KV**: Auto-configured (Deno KV)

#### Netlify
- **Blob**: Install `@netlify/blobs`, set `NETLIFY_BLOB_STORE_NAME`

### 2.6 New v1 Features

**Dialect-specific schemas:**
```ts
// server/db/schema.postgresql.ts - PostgreSQL only
// server/db/schema.ts - All dialects
```

**Database hooks:**
- `hub:db:migrations:dirs` - Add migration directories
- `hub:db:queries:paths` - Add post-migration queries
- `hub:db:migrations:done` - Callback after migrations

### Deprecated Features (v1)

Cloudflare-specific features removed:
- `hubAI()` - Use AI SDK with Workers AI Provider
- `hubBrowser()` - Puppeteer
- `hubVectorize()` - Vectorize
- `hubAutoRAG()` - AutoRAG

## Quick Reference

| Aspect | v0.9.X (Phase 1) | v1/Nightly (Phase 2) |
|--------|------------------|----------------------|
| Package | `@nuxthub/core` | `@nuxthub/core-nightly` |
| Database config | `hub.database: true` | `hub.db: 'sqlite'` |
| Directory | `server/database/` | `server/db/` |
| API | `hubDatabase()` | `useDrizzle()` |
| Cloud support | Cloudflare only | Multi-cloud |
| Stability | Stable | Experimental |

## Resources

- [Self-hosting changelog](https://hub.nuxt.com/changelog/self-hosting-first)
- [Deploy docs](https://hub.nuxt.com/docs/getting-started/deploy)
- [v1 Installation](https://v1.hub.nuxt.com/docs/getting-started/installation)
- [v1 Database](https://v1.hub.nuxt.com/docs/features/database)
- `references/wrangler-templates.md` - Cloudflare wrangler.jsonc templates
