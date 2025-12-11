> **See also [onmax/nuxt-skills](https://github.com/onmax/nuxt-skills)** - Vue, Nuxt, and NuxtHub skills for AI coding assistants

# NuxtHub Migration Skill

Claude Code skill for migrating NuxtHub projects. Two-phase approach.

## Install

```bash
git clone https://github.com/onmax/nuxthub-v1-skill.git ~/.claude/skills/nuxthub-migration
```

## Phases

### Phase 1: v0.9.X Self-Hosting (Legacy)
- Remove GitHub Actions / NuxtHub Admin
- Create `wrangler.jsonc` with Cloudflare bindings
- Set up Workers Builds CI/CD
- No code changes required

### Phase 2: v0.10 (Recommended)
- `database: true` → `db: 'sqlite'`
- `server/database/` → `server/db/`
- `hubDatabase()` → `db` from `hub:db`
- Multi-cloud support (Cloudflare, Vercel, Deno, Netlify)

## Docs

- [Self-hosting changelog](https://hub.nuxt.com/changelog/self-hosting-first)
- [Deploy docs](https://hub.nuxt.com/docs/getting-started/deploy)
- [v0.10 Installation](https://hub.nuxt.com/docs/getting-started/installation)
- [v0.10 Database](https://hub.nuxt.com/docs/features/database)
- [Legacy v0.9 docs](https://legacy.hub.nuxt.com)
