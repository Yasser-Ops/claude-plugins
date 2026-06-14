# nextjs-supabase-kit

Personal Claude plugin encoding a single web-dev workflow across every Next.js project:

- **Framework**: Next.js (App Router) + TypeScript strict + pnpm
- **Database & Auth**: Supabase (queried via `@supabase/supabase-js` directly)
- **Bot protection**: Cloudflare Turnstile
- **Payments**: provider-agnostic — Stripe, Paddle, Lemon Squeezy, Polar, or Whop (chosen per project)
- **UI**: Tailwind + shadcn/ui
- **Testing**: Vitest + Playwright
- **Deployment**: Vercel

## Skills (9)

| Skill | Purpose |
|---|---|
| `scaffold-route` | New App Router segment (page / loading / error) |
| `server-action` | Server Action with zod validation + auth |
| `supabase-migration` | New migration + type regeneration |
| `rls-policy` | Write or audit RLS policies |
| `turnstile-form` | Form + Turnstile + server-side verify |
| `optimize-images` | Convert images to WebP + unify aspect ratios |
| `cms-retrofit` | Survey-first CMS/admin layer onto an existing site |
| `payment-integration` | Pick a payment provider, then scaffold checkout + webhook |
| `paddle-webhook` | Paddle webhook handler w/ signature verify |

## MCP integrations (4)

| Server | Purpose | Required env |
|---|---|---|
| `github` | Issues, PRs, CI status | `GITHUB_PERSONAL_ACCESS_TOKEN` |
| `supabase` | Inspect schema, run queries | `SUPABASE_ACCESS_TOKEN` |
| `vercel` | Deploys, logs, projects | OAuth on first use |
| `chrome-devtools` | Drive a real browser for E2E | none |

## Setup

Set these in your shell profile before using:

```bash
export GITHUB_PERSONAL_ACCESS_TOKEN=ghp_xxx
export SUPABASE_ACCESS_TOKEN=sbp_xxx
```

Vercel uses OAuth on the first call. Chrome DevTools works out of the box.

## Relationship to CLAUDE.md

This plugin assumes the conventions in your top-level `CLAUDE.md`. Per-project `CLAUDE.md` overrides take precedence over the plugin's defaults.

## Versioning

Bump `version` in `.claude-plugin/plugin.json` on every meaningful change.
