---
name: paddle-webhook
description: Scaffold or audit a Paddle Billing (v2) webhook handler with signature verification and idempotent processing. Use when the user says "paddle webhook", "billing webhook", "subscription webhook", "payment webhook", or "/paddle-webhook".
---

# paddle-webhook

Build or review a Paddle Billing v2 webhook handler at `src/app/api/webhooks/paddle/route.ts`. Always verify the `Paddle-Signature` header before processing.

## Before writing

Confirm:

1. Which environment (sandbox vs production) — confirm env var pairs are wired correctly
2. Which event types matter (`subscription.created`, `subscription.updated`, `transaction.completed`, etc.)
3. Where Paddle IDs should be persisted in Supabase
4. Whether an idempotency table (`paddle_events`) already exists

## Route handler

```ts
// src/app/api/webhooks/paddle/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { verifyPaddleSignature } from '@/lib/paddle/verify';
import { createAdminClient } from '@/lib/supabase/admin';

export async function POST(req: NextRequest) {
  const rawBody = await req.text();
  const signature = req.headers.get('paddle-signature');

  if (!signature || !verifyPaddleSignature(rawBody, signature)) {
    return new NextResponse('Invalid signature', { status: 401 });
  }

  const event = JSON.parse(rawBody) as PaddleEvent;
  const supabase = createAdminClient(); // RLS bypass required for webhook writes

  const { data: seen } = await supabase
    .from('paddle_events')
    .select('event_id')
    .eq('event_id', event.event_id)
    .maybeSingle();

  if (seen) return NextResponse.json({ ok: true, deduped: true });

  switch (event.event_type) {
    case 'subscription.created':
    case 'subscription.updated':
    case 'transaction.completed':
      // handle each case
      break;
    default:
      console.log('Unhandled Paddle event', event.event_type);
  }

  await supabase
    .from('paddle_events')
    .insert({ event_id: event.event_id, event_type: event.event_type });

  return NextResponse.json({ ok: true });
}

type PaddleEvent = {
  event_id: string;
  event_type: string;
  data: Record<string, unknown>;
};
```

## Signature verification

```ts
// src/lib/paddle/verify.ts
import crypto from 'node:crypto';
import { env } from '@/lib/env';

export function verifyPaddleSignature(rawBody: string, header: string): boolean {
  const parts = Object.fromEntries(header.split(';').map((p) => p.split('=')));
  const ts = parts.ts;
  const h1 = parts.h1;
  if (!ts || !h1) return false;

  if (Math.abs(Date.now() / 1000 - Number(ts)) > 300) return false;

  const signedPayload = `${ts}:${rawBody}`;
  const expected = crypto
    .createHmac('sha256', env.PADDLE_NOTIFICATION_SECRET)
    .update(signedPayload)
    .digest('hex');

  return crypto.timingSafeEqual(Buffer.from(expected, 'hex'), Buffer.from(h1, 'hex'));
}
```

## Idempotency table

```sql
create table public.paddle_events (
  event_id text primary key,
  event_type text not null,
  received_at timestamptz not null default now()
);
alter table public.paddle_events enable row level security;
-- No policies; only the admin client writes here.
```

## Rules

- **Read the raw body** (`await req.text()`) before any parsing — JSON parsing changes whitespace and breaks HMAC
- **Verify timestamp** — reject events older than 5 minutes (replay protection)
- **Use `timingSafeEqual`** for HMAC comparison
- **Persist Paddle IDs at creation time** — don't try to look them up later
- **Idempotency is non-negotiable** — Paddle retries events
- **Sandbox vs prod**: separate env vars, never mix

## Audit checklist

1. Raw body read once, before parsing?
2. Signature verified before any side effects?
3. Timestamp validated against drift?
4. Comparison constant-time?
5. Dedup mechanism on `event_id`?
6. Paddle IDs persisted in the right rows?
7. Entitlement changes idempotent? (setting `status='active'` repeatedly is fine; sending an email is not)

## Common mistakes to avoid

- Parsing the body before verifying the signature
- Using `===` instead of `timingSafeEqual`
- Sending emails or running charges without idempotency
- Mixing sandbox and production secrets
