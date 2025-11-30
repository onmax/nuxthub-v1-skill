# NuxtHub Migration Skill

Claude Code skill for migrating NuxtHub projects. Two-phase approach.

## Install

```bash
git clone https://github.com/onmax/nuxthub-v1-skill.git ~/.claude/skills/nuxthub-migration
```

## Phases

### Phase 1: v0.9.X Self-Hosting (Recommended)
- Remove GitHub Actions / NuxtHub Admin
- Create `wrangler.jsonc` with Cloudflare bindings
- Set up Workers Builds CI/CD
- No code changes required

### Phase 2: v1/Nightly (Experimental)
- `database: true` → `db: 'sqlite'`
- `server/database/` → `server/db/`
- `hubDatabase()` → `useDrizzle()`
- Multi-cloud support (Cloudflare, Vercel, Deno, Netlify)
- **Warning**: Database and blob not production-ready

## Docs

- [Self-hosting changelog](https://hub.nuxt.com/changelog/self-hosting-first)
- [Deploy docs](https://hub.nuxt.com/docs/getting-started/deploy)
- [v1 Installation](https://v1.hub.nuxt.com/docs/getting-started/installation)
- [v1 Database](https://v1.hub.nuxt.com/docs/features/database)
