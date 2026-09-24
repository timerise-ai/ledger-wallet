# Testing: payments and routes

Two suites. `test/stripe.test.ts` drives the Stripe paths against a small fake that records every call, so
the tests are fast and need no network. `test/routes.test.ts` calls the route handlers directly with the host
seam mocked. Setup and the in-memory store are in [testing.md](testing.md).

## Stripe paths

```ts
// test/stripe.test.ts: top-up, split payment, refunds, webhook replay, sweeper.
// Stripe is replaced by a small fake that records calls; no network.
import { beforeEach, describe, expect, it, vi } from 'vitest';
import type Stripe from 'stripe';
import { toBalanceLedger } from '@/lib/wallet/balance-ledger';
import { createMemoryWalletStore } from '@/lib/wallet/memory-store';
import { createCurrencyPolicy } from '@/lib/wallet/money';
import { refundOrder } from '@/lib/wallet/refunds';
import { orderRefs, planPayment, startSplitPayment } from '@/lib/wallet/split';
import { sweepStaleHolds } from '@/lib/wallet/sweeper';
import { createTopUpCheckout } from '@/lib/wallet/top-up';
import { handleWalletEvent } from '@/lib/wallet/webhook';
import { createWallet, type Wallet } from '@/lib/wallet/wallet';
import type { Actor } from '@/lib/wallet/types';

const T = 'tenant-a';
const C = 'cust-1';
const actor: Actor = { type: 'system', id: 'test' };
const reason = { key: 'wallet.reason.test' };
const policy = createCurrencyPolicy(['USD', 'ISK']);

type Session = Partial<Stripe.Checkout.Session> & { id: string };

function fakeStripe() {
  const sessions = new Map<string, Session>();
  const refunds: Array<{ params: Stripe.RefundCreateParams; key?: string }> = [];
  let n = 0;
  const stripe = {
    checkout: {
      sessions: {
        create: vi.fn(async (params: Stripe.Checkout.SessionCreateParams) => {
          const line = params.line_items?.[0]?.price_data;
          const s: Session = {
            id: `cs_${++n}`, url: `https://checkout.test/cs_${n}`, status: 'open', payment_status: 'unpaid',
            currency: line?.currency ?? 'usd', amount_total: line?.unit_amount ?? 0,
            metadata: (params.metadata ?? {}) as Record<string, string>, payment_intent: `pi_${n}`,
          };
          sessions.set(s.id, s);
          return s;
        }),
        retrieve: vi.fn(async (id: string) => sessions.get(id)),
        expire: vi.fn(async (id: string) => {
          const s = sessions.get(id)!;
          if (s.status !== 'open') throw new Error('not open');
          s.status = 'expired';
          return s;
        }),
      },
    },
    refunds: {
      create: vi.fn(async (params: Stripe.RefundCreateParams, opts?: { idempotencyKey?: string }) => {
        if (refunds.some((r) => r.key === opts?.idempotencyKey)) {
          return { id: `re_${refunds.findIndex((r) => r.key === opts?.idempotencyKey) + 1}` };
        }
        refunds.push({ params, key: opts?.idempotencyKey });
        return { id: `re_${refunds.length}` };
      }),
    },
  };
  return { stripe: stripe as unknown as Stripe, raw: stripe, sessions, refunds };
}

const event = (type: Stripe.Event.Type, s: Session) =>
  ({ id: `evt_${Math.random()}`, type, data: { object: s } }) as unknown as Stripe.Event;

let mem: ReturnType<typeof createMemoryWalletStore>;
let wallet: Wallet;
let fake: ReturnType<typeof fakeStripe>;
beforeEach(() => {
  mem = createMemoryWalletStore('USD');
  wallet = createWallet({ store: mem.store, policy });
  fake = fakeStripe();
});
const usd = async () => (await wallet.getWallet(T, C)).balances.find((b) => b.currency === 'USD');
const pay = (s: Session) => Object.assign(s, { status: 'complete', payment_status: 'paid' });

describe('top-up', () => {
  const open = (currency: unknown, amount: unknown) =>
    createTopUpCheckout({
      stripe: fake.stripe, policy, tenantId: T, customerId: C, currency, amount,
      productName: 'Wallet top-up', successUrl: 'https://x/ok?s={CHECKOUT_SESSION_ID}', cancelUrl: 'https://x/no',
    });

  it('refuses a currency that is not enabled and an amount out of range', async () => {
    await expect(open('EUR', 1000)).rejects.toMatchObject({ code: 'CURRENCY_NOT_ALLOWED' });
    await expect(open('USD', 10.5)).rejects.toMatchObject({ code: 'INVALID_AMOUNT' });
    await expect(open('USD', 100)).rejects.toMatchObject({ code: 'AMOUNT_OUT_OF_RANGE' });
  });
  it('lets Stripe choose payment methods and converts ISK for Stripe', async () => {
    await open('isk', 2000);
    const params = fake.raw.checkout.sessions.create.mock.calls[0]![0];
    expect(params).not.toHaveProperty('payment_method_types');
    expect(params.line_items?.[0]?.price_data).toMatchObject({ currency: 'isk', unit_amount: 200_000 });
  });
  it('credits once however many times Stripe delivers', async () => {
    const { sessionId } = await open('USD', 2500);
    const s = pay(fake.sessions.get(sessionId)!);
    const deps = { wallet, tenantId: T };
    expect(await handleWalletEvent(event('checkout.session.completed', s), deps)).toBe('credited');
    expect(await handleWalletEvent(event('checkout.session.completed', s), deps)).toBe('replayed');
    expect(await handleWalletEvent(event('checkout.session.async_payment_succeeded', s), deps)).toBe('replayed');
    expect((await usd())?.available).toBe(2500);
  });
  it('does not credit a completed session whose async payment has not settled', async () => {
    const { sessionId } = await open('USD', 2500);
    const s = Object.assign(fake.sessions.get(sessionId)!, { status: 'complete', payment_status: 'unpaid' });
    expect(await handleWalletEvent(event('checkout.session.completed', s), { wallet, tenantId: T })).toBe('awaiting_payment');
    expect(await usd()).toBeUndefined();
    expect(await handleWalletEvent(event('checkout.session.async_payment_succeeded', s), { wallet, tenantId: T })).toBe('credited');
  });
  it("ignores another tenant's session", async () => {
    const { sessionId } = await open('USD', 2500);
    const s = pay(fake.sessions.get(sessionId)!);
    expect(await handleWalletEvent(event('checkout.session.completed', s), { wallet, tenantId: 'tenant-b' })).toBe('tenant_mismatch');
  });
  it('credits ISK in whole kronur', async () => {
    const { sessionId } = await open('ISK', 2000);
    const s = pay(fake.sessions.get(sessionId)!);
    await handleWalletEvent(event('checkout.session.completed', s), { wallet, tenantId: T });
    expect((await wallet.getWallet(T, C)).balances.find((b) => b.currency === 'ISK')?.available).toBe(2000);
  });
});

describe('planPayment', () => {
  it('never switches a wallet payment to a card silently', () => {
    expect(() => planPayment({ total: 1000, available: 400, currency: 'USD', choice: 'wallet' }))
      .toThrow(/Insufficient/);
  });
  it('splits, raising a tiny card remainder to the Stripe minimum', () => {
    expect(planPayment({ total: 1000, available: 400, currency: 'USD', choice: 'wallet_then_card' }))
      .toEqual({ walletAmount: 400, cardAmount: 600 });
    expect(planPayment({ total: 1000, available: 990, currency: 'USD', choice: 'wallet_then_card' }))
      .toEqual({ walletAmount: 950, cardAmount: 50 });
    expect(planPayment({ total: 1000, available: 5000, currency: 'USD', choice: 'wallet_then_card' }))
      .toEqual({ walletAmount: 1000, cardAmount: 0 });
  });
  it('rounds a three-decimal card part up to end in 0', () => {
    expect(planPayment({ total: 5000, available: 1234, currency: 'KWD', choice: 'wallet_then_card' }))
      .toEqual({ walletAmount: 1230, cardAmount: 3770 });
  });
  it('treats a zero total as nothing to pay', () => {
    expect(planPayment({ total: 0, available: 0, currency: 'USD', choice: 'wallet' })).toEqual({ walletAmount: 0, cardAmount: 0 });
  });
});

describe('split payment', () => {
  const start = async (orderRef = 'o1') => {
    await wallet.topUp({ tenantId: T, customerId: C, currency: 'USD', amount: 400, ref: 'seed', reason, actor });
    return startSplitPayment({
      wallet, stripe: fake.stripe, tenantId: T, customerId: C, orderRef, currency: 'USD',
      plan: { walletAmount: 400, cardAmount: 600 }, productName: 'Order', successUrl: 'https://x/ok', cancelUrl: 'https://x/no', actor,
    });
  };

  it('holds the wallet part, captures it when the card pays, and confirms the order once per delivery', async () => {
    const onSplitPaid = vi.fn(async () => undefined);
    const { sessionId } = await start();
    expect(await usd()).toMatchObject({ available: 0, held: 400 });
    const s = pay(fake.sessions.get(sessionId)!);
    expect(await handleWalletEvent(event('checkout.session.completed', s), { wallet, tenantId: T, onSplitPaid })).toBe('captured');
    await handleWalletEvent(event('checkout.session.completed', s), { wallet, tenantId: T, onSplitPaid });
    expect(await usd()).toMatchObject({ available: 0, held: 0 });
    expect(onSplitPaid).toHaveBeenCalledWith(expect.objectContaining({ orderRef: 'o1', walletSettled: true, cardAmount: 600 }));
  });
  it('sets a Checkout expiry inside the window Stripe accepts', async () => {
    const before = Math.floor(Date.now() / 1000);
    await start();
    const { expires_at } = fake.raw.checkout.sessions.create.mock.calls[0]![0];
    expect(expires_at! - before).toBeGreaterThan(30 * 60);
    expect(expires_at! - before).toBeLessThan(24 * 60 * 60);
  });
  it('releases the wallet part when Checkout expires', async () => {
    const onSplitFailed = vi.fn(async () => undefined);
    const { sessionId } = await start();
    const s = Object.assign(fake.sessions.get(sessionId)!, { status: 'expired' });
    expect(await handleWalletEvent(event('checkout.session.expired', s), { wallet, tenantId: T, onSplitFailed })).toBe('released');
    expect(await usd()).toMatchObject({ available: 400, held: 0 });
    expect(onSplitFailed).toHaveBeenCalledWith(expect.objectContaining({ cause: 'expired' }));
  });
  it('expires the Checkout if the hold cannot be placed', async () => {
    await expect(startSplitPayment({
      wallet, stripe: fake.stripe, tenantId: T, customerId: C, orderRef: 'o2', currency: 'USD',
      plan: { walletAmount: 400, cardAmount: 600 }, productName: 'Order', successUrl: 'https://x/ok', cancelUrl: 'https://x/no', actor,
    })).rejects.toMatchObject({ code: 'INSUFFICIENT_FUNDS' });
    expect(fake.raw.checkout.sessions.expire).toHaveBeenCalled();
  });
  it('takes the wallet part late if the hold was released before the card paid', async () => {
    const { sessionId, holdRef } = await start();
    await wallet.release({ tenantId: T, customerId: C, holdRef, reason, actor });
    const s = pay(fake.sessions.get(sessionId)!);
    expect(await handleWalletEvent(event('checkout.session.completed', s), { wallet, tenantId: T })).toBe('captured_late');
    expect(await usd()).toMatchObject({ available: 0, held: 0 });
  });
  it('reports walletSettled:false when the released money was spent meanwhile', async () => {
    const onSplitPaid = vi.fn(async () => undefined);
    const { sessionId, holdRef } = await start();
    await wallet.release({ tenantId: T, customerId: C, holdRef, reason, actor });
    await wallet.spend({ tenantId: T, customerId: C, currency: 'USD', amount: 400, ref: 'elsewhere', reason, actor });
    const s = pay(fake.sessions.get(sessionId)!);
    expect(await handleWalletEvent(event('checkout.session.completed', s), { wallet, tenantId: T, onSplitPaid })).toBe('wallet_short');
    expect(onSplitPaid).toHaveBeenCalledWith(expect.objectContaining({ walletSettled: false }));
  });
  it('sweeper expires a stale open Checkout and releases its hold', async () => {
    const { holdRef } = await start();
    const later = () => new Date(Date.now() + 3 * 60 * 60_000);
    const out = await sweepStaleHolds({ store: mem.store, wallet, stripeFor: () => fake.stripe, now: later });
    expect(out).toEqual([{ holdRef, outcome: 'released' }]);
    expect(await usd()).toMatchObject({ available: 400, held: 0 });
  });
  it('sweeper captures a hold whose "paid" webhook was missed', async () => {
    const { sessionId, holdRef } = await start();
    pay(fake.sessions.get(sessionId)!);
    const later = () => new Date(Date.now() + 3 * 60 * 60_000);
    const out = await sweepStaleHolds({ store: mem.store, wallet, stripeFor: () => fake.stripe, now: later });
    expect(out).toEqual([{ holdRef, outcome: 'captured' }]);
  });
});

describe('refunds go back to their source', () => {
  const base = () => ({
    wallet, stripe: fake.stripe, tenantId: T, customerId: C, orderRef: 'o9', currency: 'USD',
    walletPaid: 400, cardPaid: 600, paymentIntentId: 'pi_9', reason, actor,
  });
  it('wallet part to the wallet, card part to the card, once, however often it is retried', async () => {
    const first = await refundOrder(base());
    const second = await refundOrder(base());
    expect(first).toMatchObject({ walletRefunded: 400, cardRefunded: 600 });
    expect(second.stripeRefundId).toBe(first.stripeRefundId);
    expect(fake.refunds).toHaveLength(1);
    expect((await usd())?.available).toBe(400);
  });
  it('store-credit mode puts everything in the wallet', async () => {
    await refundOrder({ ...base(), destination: 'wallet' });
    expect((await usd())?.available).toBe(1000);
    expect(fake.refunds).toHaveLength(0);
  });
  it('an unpaid order refunds nothing', async () => {
    const r = await refundOrder({ ...base(), walletPaid: 0, cardPaid: 0, paymentIntentId: null });
    expect(r).toEqual({ walletRefunded: 0, cardRefunded: 0, stripeRefundId: null });
    expect(await usd()).toBeUndefined();
  });
  it('treats "already refunded" from Stripe as done', async () => {
    fake.raw.refunds.create.mockRejectedValueOnce(Object.assign(new Error('x'), { code: 'charge_already_refunded' }));
    await expect(refundOrder(base())).resolves.toMatchObject({ cardRefunded: 600 });
  });
  it('uses the refund ref convention', () => {
    expect(orderRefs('o9').refund).toBe('order:o9:refund');
  });
});

describe('BalanceLedger adapter (bookable-events)', () => {
  it('debits idempotently, reports insufficient, accepts lowercase codes', async () => {
    const ledger = toBalanceLedger(wallet, T);
    await ledger.credit(C, 1000, 'usd', 'evt:1:refund');
    expect(await ledger.debit(C, 700, 'usd', 'evt:2:ticket')).toBe('ok');
    expect(await ledger.debit(C, 700, 'usd', 'evt:2:ticket')).toBe('ok'); // replay
    expect(await ledger.debit(C, 700, 'usd', 'evt:3:ticket')).toBe('insufficient');
    expect(await ledger.hasEntry('evt:2:ticket')).toBe(true);
    expect(await ledger.hasEntry('evt:3:ticket')).toBe(false);
    expect((await usd())?.available).toBe(300);
  });
});
```

| Rule | Test |
|---|---|
| Top-ups outside the policy never reach Stripe | `refuses a currency that is not enabled and an amount out of range` |
| Stripe picks payment methods; ISK is sent x100 | `lets Stripe choose payment methods and converts ISK for Stripe` |
| A top-up credits once across completed, resent and async events | `credits once however many times Stripe delivers` |
| An unsettled async payment is not credit | `does not credit a completed session whose async payment has not settled` |
| No credit across tenants | `ignores another tenant's session` |
| "Wallet" never falls back to a card | `never switches a wallet payment to a card silently` |
| A card remainder is always chargeable | `splits, raising a tiny card remainder to the Stripe minimum`, the three-decimal test |
| The Checkout expiry is inside Stripe's window | `sets a Checkout expiry inside the window Stripe accepts` |
| Hold, then capture on payment, callback idempotent | `holds the wallet part, captures it...` |
| Release on expiry | `releases the wallet part when Checkout expires` |
| No orphan Checkout when the hold fails | `expires the Checkout if the hold cannot be placed` |
| Late payment after release | `takes the wallet part late...`, `reports walletSettled:false...` |
| Missed webhooks are recovered | the two sweeper tests |
| Refunds go to their source, once | the four refund tests |
| The bookable-events port behaves as declared | `debits idempotently, reports insufficient, accepts lowercase codes` |

The fake Stripe covers exactly the calls the templates make: `checkout.sessions.create`, `retrieve`,
`expire`, and `refunds.create` with idempotency keys. Before going live, repeat the flows in Stripe test
mode ([operations.md](operations.md), go-live checklist): the fake proves the wallet's decisions, test mode
proves Stripe agrees with them.

## Routes

```ts
// test/routes.test.ts: the route contract, with the host seam mocked.
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { createMemoryWalletStore } from '@/lib/wallet/memory-store';

const h = vi.hoisted(() => ({
  customer: null as null | { tenantId: string; customerId: string; email: null; locale: string },
  staff: null as null | { tenantId: string; staffId: string; role: 'viewer' | 'operator' | 'admin' },
  enabled: true,
  customers: new Set<string>(),
  store: undefined as unknown,
}));

vi.mock('@/lib/wallet/host', () => ({
  getCustomerSession: async () => h.customer,
  getStaffSession: async () => h.staff,
  customerExists: async (t: string, c: string) => h.customers.has(`${t}/${c}`),
  isWalletEnabled: async () => h.enabled,
  getWalletStore: () => h.store,
  getStripe: () => { throw new Error('no stripe in route tests'); },
  getWebhookSecret: () => null,
  absoluteUrl: (_t: string, _l: string, p: string) => `https://shop.test${p}`,
  translate: (_l: string, k: string) => k,
  splitCallbacks: {},
}));

h.store = createMemoryWalletStore('USD').store;
const { GET: getWalletRoute } = await import('@/app/api/wallet/route');
const { GET: getStatus } = await import('@/app/api/wallet/top-up/status/route');
const { GET: adminGet } = await import('@/app/api/admin/wallets/[customerId]/route');
const { POST: adjust } = await import('@/app/api/admin/wallets/[customerId]/adjust/route');
const { GET: cron } = await import('@/app/api/cron/wallet-holds/route');
const { getWallet } = await import('@/lib/wallet/server');

const params = (customerId: string) => ({ params: Promise.resolve({ customerId }) });
const post = (body: unknown) =>
  new Request('https://shop.test/x', { method: 'POST', body: JSON.stringify(body), headers: { 'content-type': 'application/json' } });
const body = { currency: 'USD', delta: 500, note: 'goodwill credit', requestId: '7f1c1f38-4b8e-4b8a-9d1e-2f3a4b5c6d7e' };

beforeEach(() => {
  h.customer = { tenantId: 'A', customerId: 'c1', email: null, locale: 'en' };
  h.staff = { tenantId: 'A', staffId: 's1', role: 'admin' };
  h.enabled = true;
  h.customers = new Set(['A/c1']);
});

describe('customer routes', () => {
  it('401 without a session, 403 when the wallet is disabled', async () => {
    h.customer = null;
    expect((await getWalletRoute(new Request('https://shop.test/api/wallet'))).status).toBe(401);
    h.customer = { tenantId: 'A', customerId: 'c1', email: null, locale: 'en' };
    h.enabled = false;
    const res = await getWalletRoute(new Request('https://shop.test/api/wallet'));
    expect(res.status).toBe(403);
    expect(await res.json()).toMatchObject({ error: { code: 'WALLET_DISABLED' } });
  });
  it('returns zero-filled balances and no-store', async () => {
    const res = await getWalletRoute(new Request('https://shop.test/api/wallet'));
    expect(res.headers.get('cache-control')).toBe('no-store');
    expect(await res.json()).toMatchObject({ preferredCurrency: 'USD', balances: [{ currency: 'USD', available: 0 }] });
  });
  it("reports another customer's top-up session as pending", async () => {
    await getWallet().topUp({
      tenantId: 'A', customerId: 'c2', currency: 'USD', amount: 900, ref: 'stripe:cs:cs_other',
      reason: { key: 'k' }, actor: { type: 'system', id: 't' },
    });
    const res = await getStatus(new Request('https://shop.test/api/wallet/top-up/status?session=cs_other'));
    expect(await res.json()).toEqual({ status: 'pending' });
  });
});

describe('admin routes', () => {
  it('404s a customer of another tenant', async () => {
    h.staff = { tenantId: 'B', staffId: 's9', role: 'admin' };
    expect((await adminGet(new Request('https://shop.test/x'), params('c1'))).status).toBe(404);
    expect((await adjust(post(body), params('c1'))).status).toBe(404);
  });
  it('only admins adjust', async () => {
    h.staff = { tenantId: 'A', staffId: 's2', role: 'operator' };
    expect((await adjust(post(body), params('c1'))).status).toBe(403);
  });
  it('requires a note and a request id', async () => {
    const res = await adjust(post({ ...body, note: '' }), params('c1'));
    expect(res.status).toBe(400);
    expect((await adjust(post({ ...body, requestId: 'x' }), params('c1'))).status).toBe(400);
    expect((await adjust(new Request('https://shop.test/x', { method: 'POST', body: '{' }), params('c1'))).status).toBe(400);
  });
  it('a double-submitted adjustment applies once and is attributed', async () => {
    const first = await adjust(post(body), params('c1'));
    const second = await adjust(post(body), params('c1'));
    expect([first.status, second.status]).toEqual([201, 200]);
    const view = await (await adminGet(new Request('https://shop.test/x'), params('c1'))).json();
    expect(view.balances[0]).toMatchObject({ currency: 'USD', available: 500 });
    expect(view.entries[0]).toMatchObject({ kind: 'ADJUSTMENT', actor: { type: 'staff', id: 's1' }, reason: { params: { note: 'goodwill credit' } } });
    expect(view.reconciled).toBe(true);
  });
  it('a negative adjustment beyond the balance is a 402 with the numbers', async () => {
    const res = await adjust(post({ ...body, delta: -10_000, requestId: '0b8e0e5a-6c3f-4c55-9a53-6f0d7d9d2b11' }), params('c1'));
    expect(res.status).toBe(402);
    expect(await res.json()).toMatchObject({ error: { code: 'INSUFFICIENT_FUNDS', details: { required: 10_000 } } });
  });
});

describe('cron', () => {
  it('fails closed when CRON_SECRET is unset', async () => {
    delete process.env.CRON_SECRET;
    expect((await cron(new Request('https://shop.test/api/cron/wallet-holds'))).status).toBe(401);
  });
});
```

`vi.mock('@/lib/wallet/host', ...)` replaces the seam, so the test exercises the real route handlers, the
real wallet and the real schemas. The same mock is the quickest way to test a host's own wiring: swap one
function at a time for the real implementation.
