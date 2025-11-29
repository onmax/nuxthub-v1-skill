---
name: nuxthub-migration
description: Use when migrating NuxtHub projects from v0 to v1. Handles config changes (database→db with dialect), directory restructuring (server/database→server/db), multi-cloud provider setup (Cloudflare, Vercel, Deno, Netlify), and wrangler.jsonc creation for Cloudflare deployments.
---

# NuxtHub v0 → v1 Migration

Migrate NuxtHub projects from v0 to v1. Key change: v1 is **cloud-agnostic** supporting multiple providers, not just Cloudflare.

## Migration Checklist

### 1. Update Package

```bash
pnpm remove @nuxthub/core
pnpm add @nuxthub/core-nightly
```

### 2. Update nuxt.config.ts

**Before (v0):**
```ts
hub: {
  database: true,
  kv: true,
  blob: true,
  cache: true
}
```

**After (v1):**
```ts
hub: {
  db: 'sqlite',  // 'sqlite' | 'postgresql' | 'mysql'
  kv: true,
  blob: true,
  cache: true
}
```

Key change: `database: true` → `db: '<dialect>'`

### 3. Rename Database Directory

```bash
mv server/database server/db
```

Files affected:
- `server/database/schema.ts` → `server/db/schema.ts`
- `server/database/migrations/` → `server/db/migrations/`

New migrations generated via `npx nuxt db generate` go to `server/db/migrations/{dialect}/`.

Update any imports referencing `~/server/database/`.

### 4. Provider-Specific Setup

v1 auto-configures based on deployment target:

#### Cloudflare Workers
Create `wrangler.jsonc`. See `references/wrangler-templates.md`.

Required bindings:
| Feature | Binding | Type |
|---------|---------|------|
| Database | `DB` | D1 |
| KV | `KV` | KV Namespace |
| Cache | `CACHE` | KV Namespace |
| Blob | `BLOB` | R2 Bucket |

#### Vercel
- **KV**: Add Redis via dashboard, install `ioredis`
- **Blob**: Add Blob Store via dashboard, install `@vercel/blob`
- **Cache**: Auto-configured (Vercel Runtime Cache)

#### Deno Deploy
- **KV**: Auto-configured (Deno KV)

#### Netlify
- **Blob**: Install `@netlify/blobs`, set `NETLIFY_BLOB_STORE_NAME`

### 5. Environment Variables (Cloudflare)

**Remove (NuxtHub Admin specific):**
- `NUXT_HUB_PROJECT_KEY`
- `NUXT_HUB_PROJECT_DEPLOY_TOKEN`

**Add (for advanced features):**
```bash
NUXT_HUB_CLOUDFLARE_ACCOUNT_ID=<account-id>
NUXT_HUB_CLOUDFLARE_API_TOKEN=<token>
# Feature-specific IDs as needed
```

### 6. Test Remote Development

```bash
npx nuxt dev --remote
```

## New Features in v1

### Realtime (WebSockets)
```ts
// nuxt.config.ts
nitro: {
  experimental: { websocket: true }
}
```

### Database Hooks
- `hub:db:migrations:dirs` - Add migration directories
- `hub:db:queries:paths` - Add post-migration queries
- `hub:db:migrations:done` - Callback after migrations

### Multi-dialect Schema
```ts
// server/db/schema.postgresql.ts - PostgreSQL only
// server/db/schema.ts - All dialects
```

## Deprecated Features

Cloudflare-specific features removed in v1:
- `hubAI()` - Workers AI
- `hubBrowser()` - Puppeteer
- `hubVectorize()` - Vectorize
- `hubAutoRAG()` - AutoRAG

## Quick Reference

| v0 | v1 |
|---|---|
| `hub.database: true` | `hub.db: 'sqlite'` |
| `server/database/` | `server/db/` |
| `server/database/migrations/` | `server/db/migrations/` |
| Cloudflare only | Multi-cloud (CF, Vercel, Deno, Netlify...) |
| `@nuxthub/core` | `@nuxthub/core-nightly` |

## Resources

- `references/wrangler-templates.md` - Cloudflare wrangler.jsonc templates
