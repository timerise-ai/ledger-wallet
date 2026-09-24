# Stripe top-up and the webhook

Topping up is two steps that never overlap. **The route opens a Stripe Checkout and credits nothing.** **The
webhook credits the wallet**, once, when Stripe says the money is collected. The success page only reports
what the webhook has already done.

```
 POST /api/wallet/top-up { currency, amount }
   validate against the policy, create Checkout (metadata: type, tenantId, customerId, currency, amount)
   return { url }  and the browser goes to Stripe
 Stripe: checkout.session.completed (payment_status 'paid')
      or checkout.session.async_payment_succeeded (bank debits, some wallets)
   POST /api/wallet/webhook/<tenantId>   signature checked with that tenant's secret
   handleWalletEvent: TOP_UP with ref stripe:cs:<session id>, amount = what Stripe collected
 Browser returns to /wallet?topUp=success&session=cs_...
   GET /api/wallet/top-up/status polls until the entry exists, then the balance reloads
```

## Opening Checkout

```ts
// lib/wallet/top-up.ts: open a Stripe Checkout that, once paid, credits the wallet.
// Nothing is credited here: only the webhook (webhook.ts) moves money into the wallet.
import type Stripe from 'stripe';
import {
  assertTopUpAmount, requireAllowedCurrency, stripeCurrency, toStripeAmount, type CurrencyPolicy,
} from './money';

export const TOP_UP_METADATA_TYPE = 'wallet_top_up';

export interface TopUpInput {
  stripe: Stripe;
  policy: CurrencyPolicy;
  tenantId: string;              // from the session, never from the body
  customerId: string;            // from the session, never from the body
  currency: unknown;             // from the body, validated here
  amount: unknown;               // minor units, from the body, validated here
  productName: string;           // translated by the host, e.g. t('wallet.topUp.productName')
  successUrl: string;            // should contain {CHECKOUT_SESSION_ID}
  cancelUrl: string;
  customerEmail?: string | null;
  stripeCustomerId?: string | null;
}

export async function createTopUpCheckout(input: TopUpInput): Promise<{ url: string; sessionId: string }> {
  // The allow-list is enforced HERE, when new money is requested, not in the webhook.
  const currency = requireAllowedCurrency(input.policy, input.currency);
  const amount = assertTopUpAmount(input.policy, currency, input.amount);

  const metadata: Record<string, string> = {
    type: TOP_UP_METADATA_TYPE,
    tenantId: input.tenantId,
    customerId: input.customerId,
    currency,
    amount: String(amount),
  };

  const session = await input.stripe.checkout.sessions.create({
    mode: 'payment',
    // No payment_method_types: Stripe offers the methods enabled in the Dashboard that
    // support this currency. A hardcoded list (e.g. ['card', 'blik']) breaks every
    // currency one of those methods does not support.
    line_items: [{
      quantity: 1,
      price_data: {
        currency: stripeCurrency(currency),
        unit_amount: toStripeAmount(amount, currency),
        product_data: { name: input.productName },
      },
    }],
    client_reference_id: input.customerId,
    ...(input.stripeCustomerId
      ? { customer: input.stripeCustomerId }
      : input.customerEmail ? { customer_email: input.customerEmail } : {}),
    metadata,
    payment_intent_data: { metadata },
    success_url: input.successUrl,
    cancel_url: input.cancelUrl,
  });
  if (!session.url) throw new Error('Stripe returned a Checkout session without a URL');
  return { url: session.url, sessionId: session.id };
}
```

| Choice | Reason |
|---|---|
| No `payment_method_types` | Stripe offers the methods enabled in the Dashboard that support the currency; a hardcoded list breaks every currency one of its methods does not support |
| Metadata on the session and the payment intent | the session drives the webhook; the payment intent's copy makes the payment searchable in the Dashboard |
| `amount` in metadata | audit only; the credit uses `amount_total`, what Stripe actually collected |
| `{CHECKOUT_SESSION_ID}` in `success_url` | Stripe substitutes the id, so the return page can ask whether that exact session was credited |
| Customer or email passed when known | receipts and the Dashboard link the payment to a person |

## The route

```ts
// app/api/wallet/top-up/route.ts: POST { currency, amount }: opens Stripe Checkout.
// The wallet is credited by the webhook, never here.
import { walletPolicy } from '@/lib/wallet/config';
import { absoluteUrl, getCustomerSession, getStripe, translate } from '@/lib/wallet/host';
import { topUpBody } from '@/lib/wallet/schemas';
import { assertWalletEnabled, errorResponse, invalid, json, readJson, unauthorized } from '@/lib/wallet/server';
import { createTopUpCheckout } from '@/lib/wallet/top-up';

export async function POST(req: Request) {
  try {
    const session = await getCustomerSession(req);
    if (!session) return unauthorized();
    await assertWalletEnabled(session.tenantId);
    const parsed = topUpBody.safeParse(await readJson(req));
    if (!parsed.success) return invalid(parsed.error);

    const { url } = await createTopUpCheckout({
      stripe: getStripe(session.tenantId),
      policy: walletPolicy,
      tenantId: session.tenantId,
      customerId: session.customerId,
      currency: parsed.data.currency,
      amount: parsed.data.amount,
      productName: translate(session.locale, 'wallet.topUp.productName'),
      customerEmail: session.email,
      stripeCustomerId: session.stripeCustomerId,
      // Locale-aware return URLs; the page reads ?topUp= and polls the status route.
      successUrl: absoluteUrl(session.tenantId, session.locale, '/wallet?topUp=success&session={CHECKOUT_SESSION_ID}'),
      cancelUrl: absoluteUrl(session.tenantId, session.locale, '/wallet?topUp=cancelled'),
    });
    return json({ url });
  } catch (err) {
    return errorResponse(err);
  }
}
```

## The webhook handler

```ts
// lib/wallet/webhook.ts: the wallet's half of the Stripe webhook. The route verifies
// the signature; this decides what the event means for the wallet. Every branch is
// idempotent, because Stripe delivers at least once and a Dashboard "resend" is a replay.
import type Stripe from 'stripe';
import { isWalletError } from './errors';
import { fromStripeAmount, normalizeCurrency, type CurrencyCode } from './money';
import { SPLIT_METADATA_TYPE } from './split';
import { TOP_UP_METADATA_TYPE } from './top-up';
import type { Actor } from './types';
import type { Wallet } from './wallet';

/** Subscribe the endpoint to exactly these. */
export const WALLET_WEBHOOK_EVENTS = [
  'checkout.session.completed',
  'checkout.session.async_payment_succeeded',
  'checkout.session.async_payment_failed',
  'checkout.session.expired',
] as const satisfies readonly Stripe.Event.Type[];

export interface SplitSettlement {
  tenantId: string;
  customerId: string;
  orderRef: string;
  holdRef: string;
  sessionId: string;
  currency: CurrencyCode;
  walletAmount: number;
  cardAmount: number;
}

export interface SplitCallbacks {
  /**
   * The card part is paid. Confirm the order, idempotently, this can run more than once.
   * `walletSettled: false` means the hold had already been released and the balance no
   * longer covers the wallet part: refund the card or collect the difference.
   */
  onSplitPaid?(s: SplitSettlement & { paymentIntentId: string | null; walletSettled: boolean }): Promise<void>;
  /** The card part will not be paid and the wallet part is back in the wallet. Free the order. */
  onSplitFailed?(s: SplitSettlement & { cause: 'expired' | 'payment_failed' }): Promise<void>;
}

export interface WebhookDeps extends SplitCallbacks {
  wallet: Wallet;
  /** The tenant this endpoint's signing secret belongs to. */
  tenantId: string;
}

export type WebhookOutcome =
  | 'ignored'            // not a wallet event
  | 'tenant_mismatch'    // metadata names another tenant, never credit across tenants
  | 'awaiting_payment'   // completed, but an async method (bank debit) has not settled yet
  | 'credited'
  | 'replayed'
  | 'captured'
  | 'captured_late'      // hold was released first; wallet part taken as a fresh spend
  | 'wallet_short'       // hold released and balance too low; onSplitPaid got walletSettled:false
  | 'released';

const actor: Actor = { type: 'system', id: 'stripe-webhook' };

export async function handleWalletEvent(event: Stripe.Event, deps: WebhookDeps): Promise<WebhookOutcome> {
  if (!(WALLET_WEBHOOK_EVENTS as readonly string[]).includes(event.type)) return 'ignored';
  const session = event.data.object as Stripe.Checkout.Session;
  const md = session.metadata ?? {};
  if (md.type !== TOP_UP_METADATA_TYPE && md.type !== SPLIT_METADATA_TYPE) return 'ignored';
  if (md.tenantId !== deps.tenantId) return 'tenant_mismatch';

  const currency = normalizeCurrency(session.currency);
  const customerId = md.customerId;
  if (!currency || !customerId) throw new Error(`Session ${session.id} is missing currency or customerId`);
  const paymentIntentId =
    typeof session.payment_intent === 'string' ? session.payment_intent : session.payment_intent?.id ?? null;

  const paid =
    event.type === 'checkout.session.async_payment_succeeded' ||
    (event.type === 'checkout.session.completed' && session.payment_status === 'paid');
  const failed = event.type === 'checkout.session.expired' || event.type === 'checkout.session.async_payment_failed';
  if (!paid && !failed) return 'awaiting_payment';

  // ---- top-up
  if (md.type === TOP_UP_METADATA_TYPE) {
    if (!paid) return 'ignored'; // an unpaid top-up has nothing to undo
    // Credit what Stripe collected, not what the metadata asked for.
    const amount = fromStripeAmount(session.amount_total ?? 0, currency);
    if (amount <= 0) return 'ignored';
    const { replayed } = await deps.wallet.topUp({
      tenantId: deps.tenantId,
      customerId,
      currency,
      amount,
      ref: `stripe:cs:${session.id}`,
      externalRef: paymentIntentId,
      reason: { key: 'wallet.reason.topUp' },
      actor,
    });
    return replayed ? 'replayed' : 'credited';
  }

  // ---- wallet + card split
  const settlement: SplitSettlement = {
    tenantId: deps.tenantId,
    customerId,
    orderRef: md.orderRef ?? '',
    holdRef: md.holdRef ?? '',
    sessionId: session.id,
    currency,
    walletAmount: Number(md.walletAmount ?? 0),
    cardAmount: fromStripeAmount(session.amount_total ?? 0, currency),
  };
  const settle = { tenantId: deps.tenantId, customerId, holdRef: settlement.holdRef, externalRef: session.id, actor };

  if (failed) {
    try {
      await deps.wallet.release({ ...settle, reason: { key: 'wallet.reason.orderReleased', params: { orderRef: settlement.orderRef } } });
    } catch (err) {
      // Never held (startSplitPayment expired the session after a failed hold) or already
      // captured by an earlier "paid" event, either way nothing to give back.
      if (!isWalletError(err) || (err.code !== 'HOLD_NOT_FOUND' && err.code !== 'HOLD_NOT_OPEN')) throw err;
      return 'ignored';
    }
    await deps.onSplitFailed?.({ ...settlement, cause: event.type === 'checkout.session.expired' ? 'expired' : 'payment_failed' });
    return 'released';
  }

  let outcome: WebhookOutcome = 'captured';
  let walletSettled = true;
  try {
    await deps.wallet.capture({ ...settle, reason: { key: 'wallet.reason.orderPaid', params: { orderRef: settlement.orderRef } } });
  } catch (err) {
    if (!isWalletError(err) || err.code !== 'HOLD_NOT_OPEN') throw err;
    // The sweeper released the hold while the customer was still paying. The card money
    // is real, so try to take the wallet part again as a plain spend.
    try {
      await deps.wallet.spend({
        tenantId: deps.tenantId, customerId, currency, amount: settlement.walletAmount,
        ref: `${settlement.holdRef}:late`, orderRef: settlement.orderRef, externalRef: session.id,
        reason: { key: 'wallet.reason.orderPaid', params: { orderRef: settlement.orderRef } }, actor,
      });
      outcome = 'captured_late';
    } catch (spendErr) {
      if (!isWalletError(spendErr) || spendErr.code !== 'INSUFFICIENT_FUNDS') throw spendErr;
      outcome = 'wallet_short';
      walletSettled = false;
    }
  }
  await deps.onSplitPaid?.({ ...settlement, paymentIntentId, walletSettled });
  return outcome;
}
```

### Events

| Event | Top-up | Split payment ([split-payment.md](split-payment.md)) |
|---|---|---|
| `checkout.session.completed`, `payment_status: 'paid'` | credit | capture the hold, then `onSplitPaid` |
| `checkout.session.completed`, `payment_status: 'unpaid'` | nothing yet (`awaiting_payment`) | nothing yet; the hold stays |
| `checkout.session.async_payment_succeeded` | credit | capture, then `onSplitPaid` |
| `checkout.session.async_payment_failed` | nothing | release, then `onSplitFailed` |
| `checkout.session.expired` | nothing | release, then `onSplitFailed` |

Subscribe the endpoint to exactly `WALLET_WEBHOOK_EVENTS`. A `completed` session with an async method (SEPA
Direct Debit, some bank transfers) is not money yet: crediting on it would hand out credit that may bounce
days later.

### Why every branch is safe to repeat

Stripe delivers at least once, retries on any non-2xx, and a Dashboard "Resend" is a replay by design. The
credit's ref is the Checkout session id, so the second delivery finds the entry and answers `replayed`. The
capture and release refs derive from the hold, so they replay the same way. Nothing in the handler needs an
event-id table.

### Tenant check

The route picks the signing secret from the path (`/api/wallet/webhook/<tenantId>`), so a valid signature
proves which tenant's Stripe account sent the event. The handler then refuses any session whose metadata
names another tenant (`tenant_mismatch`, answered 200 so Stripe stops retrying). Metadata is written by the
server when Checkout is created, never by the customer, so this check catches configuration mistakes, such
as two tenants sharing an endpoint, rather than attacks.

## The webhook route

```ts
// app/api/wallet/webhook/[tenantId]/route.ts: one Stripe endpoint per tenant's Stripe
// account. The path picks the secret; a valid signature then proves the tenant.
// If the app already has a Stripe webhook route, call handleWalletEvent from it instead.
import type Stripe from 'stripe';
import { getStripe, getWebhookSecret, splitCallbacks } from '@/lib/wallet/host';
import { getWallet, json } from '@/lib/wallet/server';
import { handleWalletEvent } from '@/lib/wallet/webhook';

export async function POST(req: Request, ctx: { params: Promise<{ tenantId: string }> }) {
  const { tenantId } = await ctx.params;
  const secret = getWebhookSecret(tenantId);
  const signature = req.headers.get('stripe-signature');
  if (!secret || !signature) return json({ error: 'unknown endpoint or missing signature' }, 400);

  let event: Stripe.Event;
  try {
    // The raw body, byte for byte, parsing it first breaks the signature.
    event = getStripe(tenantId).webhooks.constructEvent(await req.text(), signature, secret);
  } catch {
    return json({ error: 'invalid signature' }, 400);
  }

  try {
    const outcome = await handleWalletEvent(event, { wallet: getWallet(), tenantId, ...splitCallbacks });
    return json({ received: true, outcome });
  } catch (err) {
    // 500 makes Stripe retry, which is safe: every branch is idempotent.
    console.error('[wallet webhook]', event.id, event.type, err);
    return json({ error: 'processing failed' }, 500);
  }
}
```

**If the app already has a Stripe webhook route**, do not add a second endpoint. Call `handleWalletEvent`
from the existing route after its signature check, pass the tenant that route already knows, and let its
`'ignored'` outcome fall through to the app's other handlers.

**Answer 500 on any thrown error.** Stripe retries with backoff for up to three days; a swallowed error is a
payment the wallet never records.

## The return page

```ts
// app/api/wallet/top-up/status/route.ts: GET ?session=cs_...: has this top-up been credited yet?
// The success page polls this for a few seconds, because the redirect usually beats the webhook.
import { getCustomerSession } from '@/lib/wallet/host';
import { topUpStatusQuery } from '@/lib/wallet/schemas';
import { errorResponse, getWallet, invalid, json, unauthorized } from '@/lib/wallet/server';

export async function GET(req: Request) {
  try {
    const session = await getCustomerSession(req);
    if (!session) return unauthorized();
    const parsed = topUpStatusQuery.safeParse(Object.fromEntries(new URL(req.url).searchParams));
    if (!parsed.success) return invalid(parsed.error);
    const entry = await getWallet().getEntry(session.tenantId, `stripe:cs:${parsed.data.session}`);
    // Someone else's session id reads as "pending", exactly like one not yet credited.
    if (!entry || entry.customerId !== session.customerId) return json({ status: 'pending' });
    return json({ status: 'credited', amount: entry.amount, currency: entry.currency });
  } catch (err) {
    return errorResponse(err);
  }
}
```

The redirect usually arrives before the webhook. The customer UI polls this route every 2 seconds, up to 10
times, then says the payment is being processed and will appear shortly ([ui.md](ui.md)). A session id that
belongs to someone else reads as `pending`, the same as one not credited yet, so the route reveals nothing.

## Setup checklist

- [ ] One webhook endpoint per Stripe account: `https://<host>/api/wallet/webhook/<tenantId>`, events
      `WALLET_WEBHOOK_EVENTS`, secret stored where `getWebhookSecret(tenantId)` reads it
- [ ] Local development: `stripe listen --forward-to localhost:3000/api/wallet/webhook/<tenantId>`
- [ ] Payment methods enabled in the Dashboard for every listed currency
- [ ] Test: pay with `4242 4242 4242 4242`, watch the balance appear; resend the event from the Dashboard
      and confirm the balance does not change
