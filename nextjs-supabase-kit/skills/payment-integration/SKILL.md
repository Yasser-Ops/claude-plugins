---
name: payment-integration
description: Use when adding payments, checkout, subscriptions, or billing to a Next.js + Supabase site and the provider is NOT already decided — triggers on "add payments", "accept payments", "charge users", "subscriptions", "checkout", "billing", "Stripe", "Paddle", "Lemon Squeezy", "Polar", "Whop". Enforces choosing a provider before any code, then routes to provider-specific setup. Do NOT assume Paddle just because a paddle-webhook skill exists in this kit.
---

# Payment Integration (choose provider → survey → build)

## Overview

Picking a payment provider is the expensive, hard-to-reverse decision in a billing build — it dictates the data model, the tax/compliance story, the checkout flow, and the webhook contract. Agents rush past it: the biggest failure is **defaulting to Paddle just because a `paddle-webhook` skill happens to exist in this kit.** An existing Paddle skill is not a client decision.

**Core principle:** *Confirm the provider and billing model before installing an SDK or writing a line of code.* The provider is the client's choice, not the kit's default.

## The Gate

```
NO PAYMENT CODE — no SDK install, env var, migration, checkout, or webhook route —
before the provider AND billing model are explicitly confirmed.
```

Reading the repo, asking questions, and presenting the comparison table are allowed and required. Scaffolding a provider "to be safe" is not.

## Step 1 — Survey (read-only)

- Is any payment provider already wired? Grep for `stripe`, `paddle`, `lemonsqueezy`, `polar`, `whop`, existing `*/webhooks/*` routes, and billing env vars. Extend what exists; don't reinvent.
- Where does user/subscription state live in Supabase today (a `profiles`/`users` table)?
- Per `AGENTS.md`, this is a non-standard Next.js fork — read the relevant guide in `node_modules/next/dist/docs/` before writing route handlers or server actions.

## Step 2 — Decide the provider (don't default)

Ask, then map answers to the table:

1. **Merchant of record (MoR) or direct merchant?** MoR (Paddle, Lemon Squeezy, Polar, Whop) handles sales tax/VAT and is the seller of record — best for solo devs and global digital sales. Direct (Stripe) means *you/the client* own tax compliance but get the most control and lowest fees.
2. **What's being sold?** One-time, subscription, usage-based, or memberships/community access.
3. **Region & audience?** Global digital goods lean MoR (tax handled). Local/physical or US-centric often lean Stripe.
4. **Existing accounts/constraints?** A client already on Stripe or Whop usually settles it.

| Provider | MoR? | Best for | Notes |
|----------|------|----------|-------|
| **Stripe** | No | Max control, complex/usage billing, physical goods | Client owns tax/VAT; richest API; lowest fees |
| **Paddle** | Yes | Global SaaS subscriptions | MoR; already has `paddle-webhook` skill |
| **Lemon Squeezy** | Yes | Indie SaaS & digital downloads | MoR; simplest hosted checkout |
| **Polar** | Yes | Developer tools, digital products | MoR; first-class Next.js adapter |
| **Whop** | Yes | Memberships, communities, access passes | MoR; also a marketplace/storefront |

State the recommendation back and **wait for explicit confirmation** before building.

## Step 3 — Build (after confirmation) → route to the provider file

| Provider | Read |
|----------|------|
| Stripe | `stripe.md` |
| Paddle | the existing **`paddle-webhook`** skill (webhook + checkout setup) |
| Lemon Squeezy | `lemon-squeezy.md` |
| Polar | `polar.md` |
| Whop | `whop.md` |

> **Verify the SDK against current docs.** Payment SDKs change often and may be newer than training data. Read the installed package's types/README (and the provider's current docs) before finalizing — don't trust remembered method signatures.

## Shared architecture (every provider)

Regardless of provider, the shape is the same — only the SDK calls differ:

1. **Checkout entry** — a server action or route handler that creates a checkout/session and redirects. Never expose secret keys client-side.
2. **Webhook handler** at `src/app/api/webhooks/<provider>/route.ts`:
   - Read the **raw body** (`await req.text()`) before any parsing — parsing breaks signature verification.
   - **Verify the signature** before any side effect.
   - **Idempotency**: dedupe on the provider's event id via a `<provider>_events` table; providers retry.
   - Use the **Supabase admin/service-role client** for webhook writes (RLS bypass) — never the anon client.
   - **Persist provider IDs** (customer id, subscription id) onto the user row at creation time.
3. **Entitlement gating** — store `subscription_status` / `current_period_end` on the user row; gate features server-side off that, not off a live API call per request.
4. **Env separation** — distinct sandbox/test vs production keys; never mix.

## Idempotency table (adapt name per provider)

```sql
create table public.<provider>_events (
  event_id text primary key,
  event_type text not null,
  received_at timestamptz not null default now()
);
alter table public.<provider>_events enable row level security;
-- No policies; only the service-role client writes here.
```

## Red Flags — STOP, you're about to violate the gate

- "The plugin is `…-paddle`, so I'll set up Paddle." (Provider is the client's call.)
- "'Set it up' means just pick one and go." (Confirm provider + model first.)
- Running `pnpm add stripe` / scaffolding a webhook route before Step 2 is confirmed.
- Parsing the webhook body before verifying the signature.
- Writing webhook updates with the anon Supabase client instead of service-role.

**All of these mean: stop, return to the provider decision, and confirm before building.**
