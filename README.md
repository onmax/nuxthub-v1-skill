# NuxtHub v0 → v1 Migration Skill

Claude Code skill for migrating NuxtHub projects from v0 to v1.

## Install

```bash
git clone https://github.com/onmax/nuxthub-v1-skill.git ~/.claude/skills/nuxthub-migration
```

## What it covers

- `database: true` → `db: 'sqlite'` config change
- `server/database/` → `server/db/` directory rename
- `wrangler.jsonc` creation with Cloudflare bindings
- NuxtHub Admin → self-hosted Cloudflare Workers transition
- Environment variables cleanup

## Docs

- [v1 Installation](https://v1.hub.nuxt.com/docs/getting-started/installation)
- [v1 Deploy](https://v1.hub.nuxt.com/docs/getting-started/deploy)
- [v1 Database](https://v1.hub.nuxt.com/docs/features/database)
- [Self-hosting first changelog](https://hub.nuxt.com/changelog/self-hosting-first)
