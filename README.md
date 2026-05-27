# Claude Plugins

Yasser's personal [Claude Code](https://claude.ai/code) plugin marketplace.

## Install

```bash
claude plugin marketplace add https://github.com/Yasser-Ops/claude-plugins.git
```

## Plugins

| Plugin | Description |
|---|---|
| [nextjs-supabase-paddle](./nextjs-supabase-paddle) | Next.js (App Router) + Supabase + Cloudflare Turnstile + Paddle Billing + Vercel workflow |

## Adding a new plugin

1. Create a folder: `plugins/<plugin-name>/`
2. Add `.claude-plugin/plugin.json` inside it
3. Add an entry to `.claude-plugin/marketplace.json`
4. Commit and push
5. Run `claude plugin marketplace update claude-plugins && claude plugin install <plugin-name>@claude-plugins`
