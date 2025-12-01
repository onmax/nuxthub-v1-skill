---
name: nuxthub-migration
description: Use when migrating NuxtHub projects or when user mentions NuxtHub Admin sunset, GitHub Actions deployment removal, self-hosting NuxtHub, or upgrading to v1/nightly. Covers v0.9.X self-hosting (stable) and v1/nightly multi-cloud (experimental, database/blob not ready).
---

# NuxtHub Migration

## When to Use

Activate this skill when:
- User mentions NuxtHub Admin deprecation or sunset (Dec 31, 2025)
- Project uses `.github/workflows/nuxthub.yml` or NuxtHub GitHub Action
- User wants to self-host NuxtHub on Cloudflare Workers
- User asks about migrating to v1 or nightly version
- Project has `NUXT_HUB_PROJECT_KEY` or `NUXT_HUB_PROJECT_DEPLOY_TOKEN` env vars

Two-phase migration. Phase 1 is stable and recommended. Phase 2 is experimental.

## Phase 1: Self-Hosting (v0.9.X) - RECOMMENDED

Migrate from NuxtHub Admin / GitHub Actions to self-hosted Cloudflare Workers. No code changes required.

### 1.1 Remove GitHub Action Deployment

Delete `.github/workflows/nuxthub.yml` or any NuxtHub-specific GitHub Action. Workers CI (step 1.4) replaces this.

Remove deprecated env vars from CI/CD and `.env`:
- `NUXT_HUB_PROJECT_KEY`
- `NUXT_HUB_PROJECT_DEPLOY_TOKEN`

Also remove any GitHub Actions secrets related to NuxtHub deployment.

Check and clean up Cloudflare Worker secrets:
```bash
npx wrangler secret list --name <worker-name>
npx wrangler secret delete NUXT_HUB_PROJECT_DEPLOY_TOKEN --name <worker-name>
```

### 1.2 Get or Create Cloudflare Resources

NuxtHub Admin already created resources in your Cloudflare account. **Reuse them to preserve existing data.**

List existing resources:
```bash
npx wrangler d1 list              # Find existing D1 databases
npx wrangler kv namespace list    # Find existing KV namespaces
npx wrangler r2 bucket list       # Find existing R2 buckets
```

Look for resources named after your project. Use their IDs in wrangler.jsonc.

Only create new resources if none exist:
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
  "main": "dist/server/index.mjs",
  "assets": { "directory": "dist/public" },
  "compatibility_date": "2025-12-01",
  "compatibility_flags": ["nodejs_compat"],
  "d1_databases": [{ "binding": "DB", "database_name": "my-app-db", "database_id": "<from-wrangler-output>" }]
}
```

> **Note**: Nuxt cloudflare-module preset outputs to `dist/`, not `.output/`.

Required binding names:
| Feature | Binding | Type |
|---------|---------|------|
| Database | `DB` | D1 |
| KV | `KV` | KV Namespace |
| Cache | `CACHE` | KV Namespace |
| Blob | `BLOB` | R2 Bucket |

### 1.4 Set Up Workers Builds CI/CD

Ensure `nuxt.config.ts` uses the `cloudflare_module` preset:
```ts
nitro: { preset: 'cloudflare_module' }
```

In Cloudflare Dashboard:
1. Workers & Pages → Create → Import from Git
2. Connect GitHub/GitLab repository
3. Configure build settings (**both fields required**):
   - **Build command**: `pnpm build` (or `npm run build`)
   - **Deploy command**: `npx wrangler deploy`
4. Add environment variables (e.g., secrets, API keys)

> **Common mistake**: Only setting deploy command. Build must run first to generate `.output/`.

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

### Phase 1 Checklist

- [ ] Delete `.github/workflows/nuxthub.yml`
- [ ] Remove `NUXT_HUB_PROJECT_KEY` and `NUXT_HUB_PROJECT_DEPLOY_TOKEN` env vars
- [ ] Clean up old Worker secrets (`wrangler secret list/delete`)
- [ ] Get existing or create new Cloudflare resources (D1, KV, R2 as needed)
- [ ] Create `wrangler.jsonc` with bindings
- [ ] Set `nitro.preset: 'cloudflare_module'` in nuxt.config.ts
- [ ] Connect repo to Cloudflare Workers Builds
- [ ] Test with `npx nuxt dev --remote`

No code changes required. Keep `hub.database: true`, `server/database/`, `hubDatabase()`, and `@nuxthub/core`.

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

### Phase 2 Checklist

- [ ] Complete Phase 1 first
- [ ] Replace `@nuxthub/core` with `@nuxthub/core-nightly`
- [ ] Change `hub.database: true` to `hub.db: 'sqlite'`
- [ ] Rename `server/database/` to `server/db/`
- [ ] Update imports from `~/server/database/` to `~/server/db/`
- [ ] Migrate `hubDatabase()` calls to `useDrizzle()`
- [ ] Test thoroughly (database/blob features unstable)

---

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
