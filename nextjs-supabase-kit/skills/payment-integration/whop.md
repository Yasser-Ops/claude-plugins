# Whop (merchant of record + marketplace)

MoR — Whop handles tax and is also a **marketplace/storefront** for memberships, communities, and access passes. Best when the product is gated access (Discord, courses, software licenses) rather than a generic SaaS plan. Verify against the installed `@whop/api` (or `@whop-apps/sdk`) package — the SDK has changed names/shape across versions.

## Env vars

```
WHOP_API_KEY=...
WHOP_WEBHOOK_SECRET=...
NEXT_PUBLIC_WHOP_APP_ID=...
```

## Checkout / access

Whop sells via **access passes / plans**. Either send users to the hosted Whop checkout for a plan, or create a checkout session via the API and pass metadata to map back to your Supabase user:

```ts
import { WhopServerSdk } from '@whop/api';
const whop = WhopServerSdk({ appApiKey: process.env.WHOP_API_KEY! });

// Create a checkout session for a plan; attach your user id as metadata
const session = await whop.payments.createCheckoutSession({
  planId,
  metadata: { user_id: userId },
});
// redirect to the returned purchase URL
```

Whop's model is membership-centric: a user "has access" to a plan/product. Gate features off **membership validity**, not a one-time payment flag.

## Webhook

`src/app/api/webhooks/whop/route.ts` — verify the signature, then handle membership/payment events:

```ts
import { NextRequest, NextResponse } from 'next/server';
import { makeWebhookValidator } from '@whop/api';
import { createAdminClient } from '@/lib/supabase/admin';

const validateWebhook = makeWebhookValidator({ webhookSecret: process.env.WHOP_WEBHOOK_SECRET! });

export async function POST(req: NextRequest) {
  let event;
  try {
    event = await validateWebhook(req); // verifies signature over the raw body
  } catch {
    return new NextResponse('Invalid signature', { status: 401 });
  }

  const supabase = createAdminClient();
  const eventId = event.id;
  const { data: seen } = await supabase
    .from('whop_events').select('event_id').eq('event_id', eventId).maybeSingle();
  if (seen) return NextResponse.json({ ok: true, deduped: true });

  switch (event.action) {
    case 'payment.succeeded':
    case 'membership.went_valid':
    case 'membership.went_invalid':
      // upsert access status + whop_membership_id onto the user row (matched via metadata.user_id)
      break;
  }

  await supabase.from('whop_events').insert({ event_id: eventId, event_type: event.action });
  return NextResponse.json({ ok: true });
}
```

If your SDK version lacks `makeWebhookValidator`, fall back to manual HMAC-SHA256 of the raw body against the `X-Whop-Signature` header with `timingSafeEqual`.

## Key events

`payment.succeeded`, `payment.failed`, `membership.went_valid`, `membership.went_invalid`, `membership.cancel_at_period_end_changed`. The valid/invalid pair is your access on/off switch.

## Gotchas

- Think **memberships**, not subscriptions — access is driven by `membership.went_valid` / `went_invalid`.
- Attach `metadata.user_id` at checkout or you can't map the membership to a Supabase user.
- Whop App ID vs API key vs webhook secret are three distinct credentials — don't conflate.
