# Lemon Squeezy (merchant of record)

MoR — Lemon Squeezy is the seller of record and handles global sales tax/VAT. Simplest hosted checkout; great for indie SaaS and digital downloads. Verify against the current `@lemonsqueezy/lemonsqueezy.js` package.

## Env vars

```
LEMONSQUEEZY_API_KEY=...
LEMONSQUEEZY_STORE_ID=...
LEMONSQUEEZY_WEBHOOK_SECRET=...      # the signing secret you set on the webhook
```

## Checkout

Two options: a static **checkout URL** per variant (from the dashboard, append `?checkout[custom][user_id]=...`), or create one via the API:

```ts
import { lemonSqueezySetup, createCheckout } from '@lemonsqueezy/lemonsqueezy.js';
lemonSqueezySetup({ apiKey: process.env.LEMONSQUEEZY_API_KEY! });

const checkout = await createCheckout(
  process.env.LEMONSQUEEZY_STORE_ID!,
  variantId,
  { checkoutData: { custom: { user_id: userId } } } // echoed back in webhook meta.custom_data
);
// redirect to checkout.data.data.attributes.url
```

## Webhook

`src/app/api/webhooks/lemonsqueezy/route.ts` — verify an **HMAC-SHA256** of the raw body against the `X-Signature` header:

```ts
import crypto from 'node:crypto';
import { NextRequest, NextResponse } from 'next/server';
import { createAdminClient } from '@/lib/supabase/admin';

export async function POST(req: NextRequest) {
  const raw = await req.text();
  const sig = req.headers.get('x-signature') ?? '';
  const expected = crypto
    .createHmac('sha256', process.env.LEMONSQUEEZY_WEBHOOK_SECRET!)
    .update(raw).digest('hex');
  if (!crypto.timingSafeEqual(Buffer.from(expected, 'hex'), Buffer.from(sig, 'hex'))) {
    return new NextResponse('Invalid signature', { status: 401 });
  }

  const payload = JSON.parse(raw);
  const eventName = payload.meta.event_name;          // e.g. 'subscription_created'
  const userId = payload.meta.custom_data?.user_id;   // from checkoutData.custom
  const eventId = req.headers.get('x-event-id') ?? `${eventName}:${payload.data.id}`;

  const supabase = createAdminClient();
  const { data: seen } = await supabase
    .from('lemonsqueezy_events').select('event_id').eq('event_id', eventId).maybeSingle();
  if (seen) return NextResponse.json({ ok: true, deduped: true });

  switch (eventName) {
    case 'subscription_created':
    case 'subscription_updated':
    case 'order_created':
      // upsert status + ls_subscription_id + renews_at onto the user row
      break;
  }

  await supabase.from('lemonsqueezy_events').insert({ event_id: eventId, event_type: eventName });
  return NextResponse.json({ ok: true });
}
```

## Key events

`order_created`, `subscription_created`, `subscription_updated`, `subscription_cancelled`, `subscription_expired`. Subscription status lives in `data.attributes.status` (`active`, `cancelled`, `expired`, `on_trial`, `past_due`).

## Gotchas

- The signing secret here is **your** webhook secret, not the API key.
- `custom_data` only round-trips if you pass it in `checkoutData.custom` — set it or you can't map the event to a user.
- HMAC over the **raw body** only.
