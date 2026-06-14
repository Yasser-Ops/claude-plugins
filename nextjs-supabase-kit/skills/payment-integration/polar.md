# Polar (merchant of record)

MoR — Polar is the seller of record (handles tax). Developer-focused, with a first-class Next.js adapter (`@polar-sh/nextjs`) that wraps both checkout and webhook verification. Verify against the installed package — Polar's API moves fast.

## Env vars

```
POLAR_ACCESS_TOKEN=...
POLAR_WEBHOOK_SECRET=...
POLAR_SERVER=sandbox                 # 'production' in prod
```

## Checkout (Next.js adapter)

```ts
// src/app/api/checkout/route.ts
import { Checkout } from '@polar-sh/nextjs';

export const GET = Checkout({
  accessToken: process.env.POLAR_ACCESS_TOKEN!,
  successUrl: `${process.env.NEXT_PUBLIC_SITE_URL}/billing?success=1`,
  server: process.env.POLAR_SERVER as 'sandbox' | 'production',
});
// link to /api/checkout?products=<priceId>&customerExternalId=<userId>
```

`customerExternalId` lets you tie the Polar customer to your Supabase user id.

## Webhook (adapter verifies the signature for you)

Polar uses the **Standard Webhooks** spec. The adapter validates the `webhook-signature` headers against `POLAR_WEBHOOK_SECRET`, then hands you a typed, verified payload:

```ts
// src/app/api/webhooks/polar/route.ts
import { Webhooks } from '@polar-sh/nextjs';
import { createAdminClient } from '@/lib/supabase/admin';

export const POST = Webhooks({
  webhookSecret: process.env.POLAR_WEBHOOK_SECRET!,
  onPayload: async (payload) => {
    const supabase = createAdminClient();
    const eventId = payload.data.id; // dedupe key
    const { data: seen } = await supabase
      .from('polar_events').select('event_id').eq('event_id', eventId).maybeSingle();
    if (seen) return;

    switch (payload.type) {
      case 'subscription.active':
      case 'subscription.updated':
      case 'subscription.canceled':
      case 'order.paid':
        // upsert status + polar_subscription_id + current_period_end onto the user row,
        // matched via customerExternalId
        break;
    }
    await supabase.from('polar_events').insert({ event_id: eventId, event_type: payload.type });
  },
});
```

If you don't use the adapter, verify manually with `validateEvent` from `@polar-sh/sdk/webhooks` (Standard Webhooks: `webhook-id`, `webhook-timestamp`, `webhook-signature` headers over the raw body).

## Key events

`order.paid`, `subscription.active`, `subscription.updated`, `subscription.canceled`, `subscription.revoked`. Polar also has a built-in **entitlements/benefits** system you can lean on instead of rolling your own gating.

## Gotchas

- `sandbox` and `production` are separate environments with separate tokens.
- Match events to users via `customerExternalId`, set at checkout.
- The adapter already enforces raw-body verification — don't re-parse before it.
