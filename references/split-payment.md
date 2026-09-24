# Paying with the wallet, a card, or both

An order is paid in one of three ways, and each has exactly one place where money moves.

| The customer chose | Plan | Money moves |
|---|---|---|
| Wallet, and the balance covers it | `{ walletAmount: total, cardAmount: 0 }` | one `SPEND` in the **same transaction** that creates the order ([integration.md](integration.md)) |
| Card | `{ walletAmount: 0, cardAmount: total }` | the host's normal Checkout; the wallet is not involved |
| Wallet first, card for the rest | both parts non-zero | `HOLD` now, `CAPTURE` when the card pays, `RELEASE` if it does not |
| Anything, total 0 | `{ 0, 0 }` | nothing; confirm the order directly |

**"Wallet" never turns into a card payment silently.** If the balance is short, `planPayment` throws
`INSUFFICIENT_FUNDS` with both numbers and the UI offers "use my balance and pay the rest by card" as an
explicit choice.

## Why a hold, not a debit

Debiting the wallet part before the card pays leaves money gone with no order if the card never pays, and
something has to notice and give it back. A hold keeps the money in the customer's wallet, visible as
"reserved", until Stripe answers:

```
 startSplitPayment
   create Checkout for cardAmount (expires in 35 min, metadata: type wallet_split, holdRef, orderRef, ...)
   HOLD walletAmount, ref order:<id>:hold, externalRef = session id
     (if the hold fails: expire the session, rethrow)
 webhook, paid                 CAPTURE  then onSplitPaid     (confirm the order)
 webhook, expired or failed    RELEASE  then onSplitFailed   (free the order)
 sweeper, webhook never came   asks Stripe for the session and takes the same path
```

The session is created **before** the hold: a session costs nothing and can be expired, while a hold with no
session behind it could never settle. If the hold then fails because the balance moved since the plan, the
session is expired and the caller re-plans.

## Planning and starting

```ts
// lib/wallet/split.ts: paying one order from the wallet, by card, or both.
//
//   wallet only:  one SPEND in the SAME transaction that creates the order
//   card only:    the host's normal Checkout; the wallet is not involved
//   wallet+card:  HOLD the wallet part, Checkout for the rest,
//                  CAPTURE when Stripe says paid, RELEASE when it expires or fails
import type Stripe from 'stripe';
import { WalletError } from './errors';
import {
  assertChargeableAmount, minorUnitExponent, stripeCurrency, stripeMinimumCharge, toStripeAmount,
  type CurrencyCode,
} from './money';
import type { Actor } from './types';
import type { Wallet } from './wallet';

export const SPLIT_METADATA_TYPE = 'wallet_split';

/** Refs for one order. Deterministic, so every retry of every step is idempotent. */
export function orderRefs(orderRef: string) {
  return {
    spend: `order:${orderRef}`,
    hold: `order:${orderRef}:hold`,
    refund: `order:${orderRef}:refund`,
  };
}

export type PaymentChoice = 'wallet' | 'card' | 'wallet_then_card';

export interface PaymentPlan {
  walletAmount: number;
  cardAmount: number;
}

/**
 * Pure. Decides how much each source pays.
 * - 'wallet' never falls back to a card silently: short funds are an error the UI shows.
 * - A card remainder below Stripe's minimum is raised to the minimum (the wallet pays less),
 *   because Stripe rejects e.g. a $0.12 Checkout outright.
 * - Three-decimal currencies: the card part is rounded up to end in 0.
 */
export function planPayment(args: {
  total: number;
  available: number;
  currency: CurrencyCode;
  choice: PaymentChoice;
}): PaymentPlan {
  const { total, available, currency, choice } = args;
  if (!Number.isSafeInteger(total) || total < 0) {
    throw new WalletError('INVALID_AMOUNT', 'Total must be a non-negative integer', { total });
  }
  if (total === 0) return { walletAmount: 0, cardAmount: 0 }; // nothing to charge; confirm directly

  if (choice === 'card') {
    assertChargeableAmount(total, currency);
    return { walletAmount: 0, cardAmount: total };
  }
  if (choice === 'wallet') {
    if (available < total) {
      throw new WalletError('INSUFFICIENT_FUNDS', 'Insufficient balance', { available, required: total });
    }
    return { walletAmount: total, cardAmount: 0 };
  }

  let card = total - Math.min(Math.max(available, 0), total);
  if (card > 0) {
    card = Math.max(card, stripeMinimumCharge(currency) ?? 0);
    if (minorUnitExponent(currency) === 3) card = Math.ceil(card / 10) * 10;
    card = Math.min(card, total);
    assertChargeableAmount(card, currency);
  }
  return { walletAmount: total - card, cardAmount: card };
}

export interface StartSplitInput {
  wallet: Wallet;
  stripe: Stripe;
  tenantId: string;
  customerId: string;
  orderRef: string;
  currency: CurrencyCode;
  plan: PaymentPlan;             // from planPayment with choice 'wallet_then_card'
  productName: string;
  successUrl: string;
  cancelUrl: string;
  actor: Actor;
  /** Checkout lifetime, 31 to 1439 minutes (Stripe allows 30 min to 24 h). The hold lasts as long. */
  expiresInMinutes?: number;
  now?: () => Date;
}

/**
 * Creates the Checkout for the card part, then holds the wallet part against it.
 * Session first: it is free to create and can be expired, while a hold without a
 * session would have nothing to settle it. If the hold fails (the balance moved since
 * planning), the session is expired and the error is rethrown for the caller to re-plan.
 */
export async function startSplitPayment(input: StartSplitInput): Promise<{ url: string; sessionId: string; holdRef: string }> {
  const { plan, currency } = input;
  if (plan.walletAmount <= 0 || plan.cardAmount <= 0) {
    throw new Error('startSplitPayment is for wallet+card plans; pay wallet-only or card-only orders directly');
  }
  const refs = orderRefs(input.orderRef);
  const now = input.now ?? (() => new Date());
  // A margin on both ends: expires_at is measured against Stripe's clock, not ours, and a value
  // that lands a second outside the 30-minute-to-24-hour window is rejected.
  const minutes = Math.min(24 * 60 - 1, Math.max(31, input.expiresInMinutes ?? 35));

  const metadata: Record<string, string> = {
    type: SPLIT_METADATA_TYPE,
    tenantId: input.tenantId,
    customerId: input.customerId,
    orderRef: input.orderRef,
    holdRef: refs.hold,
    currency,
    walletAmount: String(plan.walletAmount),
  };
  const session = await input.stripe.checkout.sessions.create({
    mode: 'payment',
    line_items: [{
      quantity: 1,
      price_data: {
        currency: stripeCurrency(currency),
        unit_amount: toStripeAmount(plan.cardAmount, currency),
        product_data: { name: input.productName },
      },
    }],
    client_reference_id: input.customerId,
    metadata,
    payment_intent_data: { metadata },
    expires_at: Math.floor(now().getTime() / 1000) + minutes * 60,
    success_url: input.successUrl,
    cancel_url: input.cancelUrl,
  });

  try {
    await input.wallet.hold({
      tenantId: input.tenantId,
      customerId: input.customerId,
      currency,
      amount: plan.walletAmount,
      ref: refs.hold,
      orderRef: input.orderRef,
      externalRef: session.id,
      reason: { key: 'wallet.reason.orderHold', params: { orderRef: input.orderRef } },
      actor: input.actor,
    });
  } catch (err) {
    await input.stripe.checkout.sessions.expire(session.id).catch(() => undefined);
    throw err;
  }
  if (!session.url) throw new Error('Stripe returned a Checkout session without a URL');
  return { url: session.url, sessionId: session.id, holdRef: refs.hold };
}
```

`planPayment` handles the amounts Stripe would reject:

- **A card remainder below the minimum charge** is raised to the minimum and the wallet pays less. A $0.12
  Checkout is refused outright; $0.50 by card and $9.50 from the wallet is not.
- **Three-decimal currencies** round the card part up to end in 0 ([money.md](money.md)).

## When the card pays after the hold was released

The sweeper releases holds whose Checkout is past its expiry. If the customer completes payment in the
instant between the sweeper's check and the expiry, the webhook finds the hold `RELEASED`. The card money is
real, so the handler takes the wallet part again as a plain `SPEND` (ref `<holdRef>:late`). If the balance
no longer covers it, the handler calls `onSplitPaid` with `walletSettled: false`, and the host either refunds
the card part or asks the customer for the difference. `test/stripe.test.ts` covers both branches.

## Host callbacks

`SplitCallbacks` in `webhook.ts` is how the wallet tells the host about its order. Both callbacks may run more
than once for the same order (webhook retries, the sweeper), so make them idempotent: confirming an order
that is already confirmed is a no-op.

```ts
// lib/orders/split-callbacks.ts: EXAMPLE host wiring, exported from lib/wallet/host.ts as splitCallbacks.
import type { SplitCallbacks } from '@/lib/wallet/webhook';

export interface OrderPort {
  confirm(orderId: string, payment: { walletPaid: number; cardPaid: number; paymentIntentId: string | null }): Promise<void>;
  release(orderId: string): Promise<void>;
  flagForRefund(orderId: string, reason: string): Promise<void>;
}

export function makeSplitCallbacks(orders: OrderPort): SplitCallbacks {
  return {
    async onSplitPaid(s) {
      if (!s.walletSettled) {
        // Card paid, wallet part gone: do not confirm on half the money.
        await orders.flagForRefund(s.orderRef, 'wallet part unavailable after hold release');
        return;
      }
      await orders.confirm(s.orderRef, { walletPaid: s.walletAmount, cardPaid: s.cardAmount, paymentIntentId: s.paymentIntentId });
    },
    async onSplitFailed(s) {
      await orders.release(s.orderRef);
    },
  };
}
```

If the order can no longer be fulfilled when `onSplitPaid` runs (the slot sold out meanwhile), refund it
with `refundOrder` below instead of confirming it.

## Refunds go back to their source

```ts
// lib/wallet/refunds.ts: give back what was actually paid, to where it came from.
//
// The caller passes the amounts recorded on the order WHEN IT WAS PAID
// (walletPaid, cardPaid), never the order's current total, which can be edited
// after payment, and never anything for an order that was not paid.
import type Stripe from 'stripe';
import { WalletError } from './errors';
import { toStripeAmount, type CurrencyCode } from './money';
import { orderRefs } from './split';
import type { Actor, Reason } from './types';
import type { Wallet } from './wallet';

/** 'source': wallet part to the wallet, card part to the card. 'wallet': everything to the wallet (store credit). */
export type RefundDestination = 'source' | 'wallet';

export interface RefundInput {
  wallet: Wallet;
  stripe: Stripe | null;
  tenantId: string;
  customerId: string;
  orderRef: string;
  currency: CurrencyCode;
  walletPaid: number;
  cardPaid: number;
  paymentIntentId: string | null;
  destination?: RefundDestination;
  reason: Reason;
  actor: Actor;
}

export interface RefundResult {
  walletRefunded: number;
  cardRefunded: number;
  stripeRefundId: string | null;
}

/**
 * Safe to call again after any failure: the wallet part is idempotent on its ref, and the
 * Stripe part on its idempotency key (and, past the key's 24h life, on Stripe refusing to
 * refund a charge twice). Retry until it returns.
 */
export async function refundOrder(input: RefundInput): Promise<RefundResult> {
  const { walletPaid, cardPaid } = input;
  for (const [name, value] of [['walletPaid', walletPaid], ['cardPaid', cardPaid]] as const) {
    if (!Number.isSafeInteger(value) || value < 0) {
      throw new WalletError('INVALID_AMOUNT', `${name} must be a non-negative integer`, { [name]: value });
    }
  }
  const destination = input.destination ?? 'source';
  const toWallet = destination === 'wallet' ? walletPaid + cardPaid : walletPaid;
  const toCard = destination === 'wallet' ? 0 : cardPaid;
  const refs = orderRefs(input.orderRef);

  if (toWallet > 0) {
    await input.wallet.refund({
      tenantId: input.tenantId,
      customerId: input.customerId,
      currency: input.currency,
      amount: toWallet,
      ref: refs.refund,
      orderRef: input.orderRef,
      externalRef: input.paymentIntentId,
      reason: input.reason,
      actor: input.actor,
    });
  }

  let stripeRefundId: string | null = null;
  if (toCard > 0) {
    if (!input.stripe || !input.paymentIntentId) {
      throw new Error(`Order ${input.orderRef} has a card part but no payment intent to refund`);
    }
    try {
      const refund = await input.stripe.refunds.create(
        {
          payment_intent: input.paymentIntentId,
          amount: toStripeAmount(toCard, input.currency),
          metadata: { tenantId: input.tenantId, orderRef: input.orderRef, ref: refs.refund },
        },
        { idempotencyKey: `${input.tenantId}:${refs.refund}` },
      );
      stripeRefundId = refund.id;
    } catch (err) {
      // A retry after the idempotency key expired: Stripe refuses the second refund.
      if ((err as { code?: string }).code !== 'charge_already_refunded') throw err;
    }
  }
  return { walletRefunded: toWallet, cardRefunded: toCard, stripeRefundId };
}
```

| Input | Rule |
|---|---|
| `walletPaid`, `cardPaid` | the amounts recorded on the order **when it was paid**; never its current total, never anything for an unpaid order |
| `destination: 'source'` (default) | wallet part as a wallet `REFUND`, card part as a Stripe refund of the payment intent |
| `destination: 'wallet'` | everything as store credit; a host policy choice, and in some jurisdictions it needs the customer's consent |
| Retries | wallet part idempotent on `order:<id>:refund`; Stripe part on its idempotency key; after the key's 24 h life, Stripe's `charge_already_refunded` counts as done |

Partial refunds (one ticket of three) need one ref per refund, for example `order:<id>:refund:<n>`, and the
order must track how much of each source is already refunded. That is an extension; the template refunds
whole orders.

## The sweeper

```ts
// lib/wallet/sweeper.ts: settles holds whose webhook never arrived (endpoint down,
// event filtered out, secret rotated). Without it, a missed "expired" event freezes the
// customer's money in `held` forever. Run every 15 minutes.
import type Stripe from 'stripe';
import { handleWalletEvent, type SplitCallbacks, type WebhookOutcome } from './webhook';
import type { WalletStore } from './ports';
import type { Wallet } from './wallet';

export interface SweepDeps extends SplitCallbacks {
  store: WalletStore;
  wallet: Wallet;
  stripeFor(tenantId: string): Stripe;
  /** Holds younger than this are left to the webhook. Must exceed the Checkout lifetime. */
  olderThanMinutes?: number;
  limit?: number;
  now?: () => Date;
}

export async function sweepStaleHolds(deps: SweepDeps) {
  const now = deps.now ?? (() => new Date());
  const cutoff = new Date(now().getTime() - (deps.olderThanMinutes ?? 90) * 60_000);
  const holds = await deps.store.listOpenHolds(cutoff, deps.limit ?? 100);
  const outcomes: Array<{ holdRef: string; outcome: WebhookOutcome | 'released_orphan' | 'error'; error?: string }> = [];

  for (const hold of holds) {
    try {
      if (!hold.externalRef) {
        // A hold with no Checkout behind it can never be paid.
        await deps.wallet.release({
          tenantId: hold.tenantId, customerId: hold.customerId, holdRef: hold.holdRef,
          reason: { key: 'wallet.reason.orderReleased', params: { orderRef: hold.orderRef ?? '' } },
          actor: { type: 'system', id: 'hold-sweeper' },
        });
        outcomes.push({ holdRef: hold.holdRef, outcome: 'released_orphan' });
        continue;
      }
      const stripe = deps.stripeFor(hold.tenantId);
      let session = await stripe.checkout.sessions.retrieve(hold.externalRef);
      if (session.status === 'open') {
        // Past its own expiry but still open, close it so it cannot be paid after release.
        // If the customer pays in this instant, expire fails and the next run sees 'complete'.
        session = await stripe.checkout.sessions.expire(session.id);
      }
      // Replay the decision the webhook would have made, through the same code path.
      // 'complete' + unpaid is an async payment still settling: the handler answers
      // awaiting_payment and the hold stays until the bank decides.
      const type: Stripe.Event.Type =
        session.status === 'expired' ? 'checkout.session.expired' : 'checkout.session.completed';
      const synthetic = { id: `sweep_${hold.holdRef}`, type, data: { object: session } } as unknown as Stripe.Event;
      const outcome = await handleWalletEvent(synthetic, { ...deps, tenantId: hold.tenantId });
      outcomes.push({ holdRef: hold.holdRef, outcome });
    } catch (err) {
      outcomes.push({ holdRef: hold.holdRef, outcome: 'error', error: err instanceof Error ? err.message : String(err) });
    }
  }
  return outcomes;
}
```

```ts
// app/api/cron/wallet-holds/route.ts: every 15 minutes: settle holds whose webhook was missed.
import { getStripe, getWalletStore, splitCallbacks } from '@/lib/wallet/host';
import { getWallet, json } from '@/lib/wallet/server';
import { sweepStaleHolds } from '@/lib/wallet/sweeper';

export async function GET(req: Request) {
  const secret = process.env.CRON_SECRET;
  // Fail closed: an unset secret must not leave the route open to everyone.
  if (!secret || req.headers.get('authorization') !== `Bearer ${secret}`) {
    return json({ error: 'unauthorized' }, 401);
  }
  const outcomes = await sweepStaleHolds({
    store: getWalletStore(),
    wallet: getWallet(),
    stripeFor: getStripe,
    ...splitCallbacks,
  });
  const errors = outcomes.filter((o) => o.outcome === 'error');
  if (errors.length > 0) console.error('[wallet sweeper]', errors);
  return json({ swept: outcomes.length, errors: errors.length, outcomes });
}
```

| Setting | Value | Why |
|---|---|---|
| Schedule | every 15 minutes | bounds how long money stays reserved after a missed webhook |
| `olderThanMinutes` | 90 | must exceed the Checkout lifetime (35 minutes by default), or the sweeper expires sessions the customer is still paying |
| Hold with no session | released | it can never be paid |
| Session `open` past the cutoff | expired, then released | closing it first means it cannot be paid after the money is back |
| Session `complete`, unpaid | left alone | an async payment still settling; its own webhook decides |
| `CRON_SECRET` unset | 401 | the route fails closed |

On Vercel, add the cron to `vercel.ts` or `vercel.json` (`{ path: '/api/cron/wallet-holds', schedule:
'*/15 * * * *' }`); Vercel sends `Authorization: Bearer $CRON_SECRET`. Other hosts: any scheduler that can
send that header.
