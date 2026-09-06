---
name: stripe
description: Add payments to a Next.js app with Stripe — env keys, a Checkout Session route (one-time or subscription), a signature-verified webhook that grants access, a customer portal link, and local testing through the Stripe CLI with test cards. Use when a product needs to charge money. Grants access from the webhook, never from the success redirect.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
argument-hint: [project-path] [one-time|subscription]
---

# Stripe — minimum viable payments

Checkout Session for the payment, webhook for the truth, customer portal for everything after. That's the whole integration for most indie products.

**Argument**: `$ARGUMENTS` is the project path and the pricing model. Default to the current directory; ask one-time vs subscription if unstated. Assumes Next.js App Router; for other stacks the same three endpoints exist, just in that framework's routing.

## Process

### 1. Detect state

```bash
cat package.json | grep -E '"(next|stripe|@stripe/stripe-js)"'
grep -l STRIPE .env.local .env.example 2>/dev/null
ls src/app/api/checkout src/app/api/webhooks 2>/dev/null
which stripe || echo "no stripe cli"
grep -rl "supabase" src/lib 2>/dev/null   # decides where "grant access" writes
```

### 2. Keys and product (the human part)

1. stripe.com → stay in **Test mode** (toggle top right).
2. **Developers → API keys**: copy the Secret key (`sk_test_`) and Publishable key (`pk_test_`).
3. **Products → Add product**: name, price, and for subscriptions a recurring interval. Copy the **Price ID** (`price_…`).
4. `.env.local`, plus blanked copies in `.env.example`:
```
STRIPE_SECRET_KEY=sk_test_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
STRIPE_PRICE_ID=price_...
STRIPE_WEBHOOK_SECRET=whsec_...   # from step 5
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### 3. Install and scaffold

```bash
npm install stripe
```

`src/lib/stripe.ts`:
```typescript
import Stripe from 'stripe';
export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
```

**Checkout** `src/app/api/checkout/route.ts`:
```typescript
import { stripe } from '@/lib/stripe';

export async function POST(req: Request) {
  const { userId, email } = await req.json();       // from your auth session, not trusted client input
  const base = process.env.NEXT_PUBLIC_APP_URL!;
  const session = await stripe.checkout.sessions.create({
    mode: 'subscription',                            // or 'payment' for one-time
    line_items: [{ price: process.env.STRIPE_PRICE_ID!, quantity: 1 }],
    customer_email: email,
    client_reference_id: userId,
    metadata: { userId },
    success_url: `${base}/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${base}/pricing`,
    allow_promotion_codes: true,
  });
  return Response.json({ url: session.url });
}
```

Frontend: `fetch('/api/checkout', { method: 'POST', body: JSON.stringify({ userId, email }) })` then `window.location.href = url`. Read `userId` from the server session (Supabase `getUser()`, NextAuth, whatever the app has). If there is no auth yet, say so and suggest `/supabase` first; a payment with no user to attach it to is a support ticket waiting to happen.

**Webhook** `src/app/api/webhooks/stripe/route.ts`:
```typescript
import { headers } from 'next/headers';
import { stripe } from '@/lib/stripe';
import type Stripe from 'stripe';

export async function POST(req: Request) {
  const body = await req.text();                     // raw body; JSON parsing breaks the signature
  const sig = (await headers()).get('stripe-signature')!;
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(body, sig, process.env.STRIPE_WEBHOOK_SECRET!);
  } catch (err) {
    return new Response(`Webhook error: ${(err as Error).message}`, { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed': {
      const s = event.data.object as Stripe.Checkout.Session;
      await grantAccess(s.metadata!.userId, s.customer as string, s.subscription as string | null);
      break;
    }
    case 'invoice.payment_succeeded':               // renewals
    case 'customer.subscription.updated': {          // plan change, past_due
      const sub = event.data.object as Stripe.Subscription;
      await syncSubscription(sub);
      break;
    }
    case 'customer.subscription.deleted': {
      await revokeAccess((event.data.object as Stripe.Subscription).customer as string);
      break;
    }
  }
  return Response.json({ received: true });
}
```

Implement `grantAccess` / `syncSubscription` / `revokeAccess` against the app's real data store. With Supabase: a `subscriptions` table (or columns on `profiles`) holding `stripe_customer_id`, `stripe_subscription_id`, `status`, `current_period_end`, written with the **service-role** server client because the webhook has no user session. Make the handler idempotent: Stripe retries on any non-2xx and can deliver an event twice.

**Customer portal** `src/app/api/portal/route.ts`: `stripe.billingPortal.sessions.create({ customer, return_url })` and redirect to `.url`. Enable it once in **Settings → Billing → Customer portal**. This gives users card updates, cancellation, and invoices for free; don't build those screens.

### 4. Success page

`/success` reads `session_id`, calls `stripe.checkout.sessions.retrieve` server-side, and shows a thank-you. It must not grant access. Anyone can load `/success?session_id=anything`; only the webhook is trustworthy.

### 5. Test locally

```bash
brew install stripe/stripe-cli/stripe    # or the installer for the OS
stripe login
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

The CLI prints a `whsec_…` secret. Put it in `.env.local` as `STRIPE_WEBHOOK_SECRET` and restart the dev server. Then, in a second terminal:
```bash
stripe trigger checkout.session.completed
```
Expect a `200` in the `listen` output and a row written in the database. Then do a real run through the UI with card `4242 4242 4242 4242`, any future expiry, any CVC. Also try `4000 0000 0000 9995` (declined) and `4000 0027 6000 3184` (3D Secure challenge) so the failure paths have been seen once.

### 6. Going live (checklist for the user)

1. Toggle **Live mode**. Products and prices don't carry over; recreate them and update `STRIPE_PRICE_ID`.
2. Swap to `sk_live_` / `pk_live_` in Vercel's environment variables.
3. **Developers → Webhooks → Add endpoint**: `https://<domain>/api/webhooks/stripe`, subscribe to the four events above, copy the new signing secret into Vercel. Test and live secrets are different; the CLI secret is different again.
4. Real $1 purchase, then refund it from the dashboard.

### 7. Report

```
Stripe (test mode)
- Routes: /api/checkout · /api/webhooks/stripe · /api/portal
- Mode: subscription · price_abc (Pro, $10/mo)
- Access: written to supabase subscriptions table via service role
- Webhook: stripe listen forwarding · trigger checkout.session.completed → 200, row created
- Cards tested: 4242 success · 9995 decline
- Live checklist: given above · env vars still test keys
```

## Gotchas

- **Renewals are `invoice.payment_succeeded`, not `checkout.session.completed`.** Handle both or month two silently revokes everyone.
- **Failed renewals don't cancel immediately.** Stripe retries for about a week; the subscription goes `past_due` via `customer.subscription.updated`. Decide whether past_due keeps access.
- **Raw body.** Any middleware that parses JSON before the webhook route breaks signature verification. In App Router `req.text()` is correct.
- **Tax.** US-only sales are simple. EU/UK sales need VAT handling: Stripe Tax (per-transaction fee) or a merchant of record like Lemon Squeezy (5% + 50¢, they file it). Decide before the first European customer, not after.
- **Fees**: 2.9% + 30¢ per successful card charge in the US, higher for international cards and currency conversion.
