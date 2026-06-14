# Stripe (direct merchant)

Direct merchant — the client owns sales-tax/VAT compliance (consider Stripe Tax). Most control, lowest fees. Read the installed `stripe` package types before finalizing API calls.

## Env vars

```
STRIPE_SECRET_KEY=sk_test_...        # sk_live_... in prod
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
```

## Checkout (server action → hosted Checkout)

```ts
// src/lib/stripe/checkout.ts
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function createCheckoutSession(opts: {
  priceId: string;
  userId: string;
  customerEmail: string;
}) {
  const session = await stripe.checkout.sessions.create({
    mode: 'subscription', // or 'payment' for one-time
    line_items: [{ price: opts.priceId, quantity: 1 }],
    customer_email: opts.customerEmail,
    client_reference_id: opts.userId,        // map back to the Supabase user
    success_url: `${process.env.NEXT_PUBLIC_SITE_URL}/billing?success=1`,
    cancel_url: `${process.env.NEXT_PUBLIC_SITE_URL}/billing?canceled=1`,
  });
  return session.url!;
}
```

A customer portal (`stripe.billingPortal.sessions.create`) lets users manage/cancel without custom UI.

## Webhook

`src/app/api/webhooks/stripe/route.ts` — verify with `constructEvent`:

```ts
import { NextRequest, NextResponse } from 'next/server';
import Stripe from 'stripe';
import { createAdminClient } from '@/lib/supabase/admin';

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

export async function POST(req: NextRequest) {
  const raw = await req.text();
  const sig = req.headers.get('stripe-signature')!;
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(raw, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch {
    return new NextResponse('Invalid signature', { status: 400 });
  }

  const supabase = createAdminClient();
  const { data: seen } = await supabase
    .from('stripe_events').select('event_id').eq('event_id', event.id).maybeSingle();
  if (seen) return NextResponse.json({ ok: true, deduped: true });

  switch (event.type) {
    case 'checkout.session.completed':
    case 'customer.subscription.updated':
    case 'customer.subscription.deleted':
      // upsert subscription_status, stripe_customer_id, stripe_subscription_id,
      // current_period_end onto the user row
      break;
    default:
      break;
  }

  await supabase.from('stripe_events').insert({ event_id: event.id, event_type: event.type });
  return NextResponse.json({ ok: true });
}
```

## Key events

- `checkout.session.completed` — first activation; read `client_reference_id` to find the user.
- `customer.subscription.updated` / `.deleted` — status changes, renewals, cancellations.
- `invoice.paid` / `invoice.payment_failed` — renewal success/dunning.

## Gotchas

- Verify with the **raw body**; never `JSON.parse` first.
- Test mode and live mode have **separate** keys, products, and webhook secrets.
- The Stripe Customer ID is the join key — persist it on the user row on first checkout.
