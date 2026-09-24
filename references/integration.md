# Integrating with the host's orders

The wallet does not know what an order is. The host does, and it owns three integration points: paying an
order from the wallet atomically, confirming or releasing an order when a split payment settles
([split-payment.md](split-payment.md)), and refunding what an order was actually paid with.

## Pay in the same transaction as the order

A wallet-only payment is a `SPEND` posted **inside the transaction that creates the order**, through
`postInTx`. Either both commit or neither does:

| Failure | Outcome |
|---|---|
| Capacity check fails (slot sold out) | no entry, no order, balance unchanged |
| Balance too low | `INSUFFICIENT_FUNDS`, no order |
| Order insert fails after the debit | the transaction rolls back the debit with it |
| The request is retried with the same order id | the spend replays; the order insert is the host's to make idempotent |

The order records **what it was paid with**, per source. Refunds and support read these fields, never the
order total, which staff may edit later.

### Firestore

```ts
// lib/orders/pay-from-wallet.firestore.ts: EXAMPLE host code: create an order and pay it in
// full from the wallet in ONE Firestore transaction. If the capacity check fails, no money
// moves; if the debit fails, no order exists. There is nothing to refund and nothing to leak.
import type { Firestore } from 'firebase-admin/firestore';
import { DEFAULT_CURRENCY } from '@/lib/wallet/config';
import { firestoreWalletTx } from '@/lib/wallet/firestore-store';
import { orderRefs } from '@/lib/wallet/split';
import type { Wallet } from '@/lib/wallet/wallet';

export class SoldOutError extends Error {}

export async function createOrderPaidFromWallet(db: Firestore, wallet: Wallet, input: {
  tenantId: string;
  customerId: string;
  orderId: string;          // generate before the transaction so retries reuse it
  slotId: string;
  currency: string;
  total: number;            // computed on the SERVER from the price list, never read from the client
}) {
  return db.runTransaction(async (t) => {
    // 1. The host's reads first (Firestore: all reads before any write).
    const slotRef = db.collection('slots').doc(input.slotId);
    const slot = await t.get(slotRef);
    const booked = (slot.get('booked') as number | undefined) ?? 0;
    if (booked >= ((slot.get('capacity') as number | undefined) ?? 0)) throw new SoldOutError();

    // 2. The wallet movement: its reads, then its writes.
    const { entry } = await wallet.postInTx(firestoreWalletTx(db, t, DEFAULT_CURRENCY), {
      tenantId: input.tenantId,
      customerId: input.customerId,
      currency: input.currency,
      ref: orderRefs(input.orderId).spend,
      command: { kind: 'SPEND', amount: input.total },
      orderRef: input.orderId,
      reason: { key: 'wallet.reason.orderPaid', params: { orderRef: input.orderId } },
      actor: { type: 'customer', id: input.customerId },
    });

    // 3. The host's writes. Record what was PAID, per source, refunds read these, not the total.
    t.update(slotRef, { booked: booked + 1 });
    t.set(db.collection('orders').doc(input.orderId), {
      tenantId: input.tenantId,
      customerId: input.customerId,
      status: 'CONFIRMED',
      currency: input.currency,
      total: input.total,
      payment: { walletPaid: input.total, cardPaid: 0, walletEntryId: entry.id, paymentIntentId: null },
    });
    return { orderId: input.orderId, walletEntryId: entry.id };
  });
}
```

### Postgres

```ts
// lib/orders/pay-from-wallet.postgres.ts: EXAMPLE host code: the same flow on Postgres.
// Postgres has no read-before-write rule. Call postInTx after any slow work, so the wallet's
// row lock (taken inside it) is held only until the commit.
import { DEFAULT_CURRENCY } from '@/lib/wallet/config';
import { postgresWalletTx, type SqlPool } from '@/lib/wallet/postgres-store';
import { orderRefs } from '@/lib/wallet/split';
import type { Wallet } from '@/lib/wallet/wallet';

export async function createOrderPaidFromWallet(pool: SqlPool, wallet: Wallet, input: {
  tenantId: string;
  customerId: string;
  orderId: string;
  slotId: string;
  currency: string;
  total: number;
}) {
  const client = await pool.connect();
  try {
    await client.query('begin');
    const { rowCount } = await client.query(
      `update slots set booked = booked + 1 where id = $1 and booked < capacity`, [input.slotId],
    );
    if (!rowCount) throw new Error('SOLD_OUT');

    const { entry } = await wallet.postInTx(postgresWalletTx(client, DEFAULT_CURRENCY), {
      tenantId: input.tenantId,
      customerId: input.customerId,
      currency: input.currency,
      ref: orderRefs(input.orderId).spend,
      command: { kind: 'SPEND', amount: input.total },
      orderRef: input.orderId,
      reason: { key: 'wallet.reason.orderPaid', params: { orderRef: input.orderId } },
      actor: { type: 'customer', id: input.customerId },
    });

    await client.query(
      `insert into orders (id, tenant_id, customer_id, status, currency, total, wallet_paid, card_paid, wallet_entry_id)
       values ($1, $2, $3, 'CONFIRMED', $4, $5, $5, 0, $6)`,
      [input.orderId, input.tenantId, input.customerId, input.currency, input.total, entry.id],
    );
    await client.query('commit');
    return { orderId: input.orderId, walletEntryId: entry.id };
  } catch (err) {
    await client.query('rollback');
    throw err;
  } finally {
    client.release();
  }
}
```

Both examples run against the Firestore emulator and a real Postgres in `test/order-example.test.ts`
([testing-stores.md](testing-stores.md)): a sold-out slot moves no money and a short wallet creates no order.

**Price on the server.** `total` is computed from the host's own price list inside the request, never taken
from the client. A client-supplied total makes the customer the one who decides how much is debited.

## Order records

Whatever the host's order looks like, it needs these fields for the wallet to work with it:

| Field | Set when | Read by |
|---|---|---|
| `currency` | the order is priced | every wallet call for the order |
| `payment.walletPaid` | the spend or the capture commits | `refundOrder` |
| `payment.cardPaid` | the card part is paid | `refundOrder` |
| `payment.paymentIntentId` | the card part is paid | `refundOrder` (Stripe refund) |
| `payment.walletEntryId` or `holdRef` | the wallet part is taken | support, reconciliation against orders |

## bookable-events

The `bookable-events` skill takes an optional `BalanceLedger` port so event tickets can be paid from, and
refunded to, a stored-credit wallet. This adapter implements it exactly:

```ts
// lib/wallet/balance-ledger.ts: adapts the wallet to the `BalanceLedger` port of the
// bookable-events skill, so event tickets can be paid from and refunded to the wallet.
import { isWalletError } from './errors';
import { normalizeCurrency } from './money';
import type { Actor } from './types';
import type { Wallet } from './wallet';

/** The port, exactly as bookable-events declares it. Every call is idempotent on `ref`. */
export interface BalanceLedger {
  debit(customerId: string, amount: number, currency: string, ref: string): Promise<'ok' | 'insufficient'>;
  credit(customerId: string, amount: number, currency: string, ref: string): Promise<void>;
  hasEntry(ref: string): Promise<boolean>;
}

export function toBalanceLedger(wallet: Wallet, tenantId: string): BalanceLedger {
  const actor: Actor = { type: 'system', id: 'events' };
  // bookable-events uses lowercase codes ('usd'); the wallet stores uppercase.
  const code = (c: string) => {
    const n = normalizeCurrency(c);
    if (!n) throw new Error(`Unknown currency ${c}`);
    return n;
  };
  return {
    async debit(customerId, amount, currency, ref) {
      try {
        await wallet.spend({
          tenantId, customerId, currency: code(currency), amount, ref,
          reason: { key: 'wallet.reason.eventTicket' }, actor,
        });
        return 'ok';
      } catch (err) {
        if (isWalletError(err) && err.code === 'INSUFFICIENT_FUNDS') return 'insufficient';
        throw err;
      }
    },
    async credit(customerId, amount, currency, ref) {
      await wallet.refund({
        tenantId, customerId, currency: code(currency), amount, ref,
        reason: { key: 'wallet.reason.eventRefund' }, actor,
      });
    },
    async hasEntry(ref) {
      return (await wallet.getEntry(tenantId, ref)) !== null;
    },
  };
}
```

Pass `toBalanceLedger(getWallet(), tenantId)` as the engine's `ledger`. The adapter accepts bookable-events'
lowercase currency codes and stores them uppercase, maps `INSUFFICIENT_FUNDS` to `'insufficient'`, and keeps
bookable-events' refs as the wallet's refs, so both skills agree on what has already happened. Covered by the
adapter test in `test/stripe.test.ts`.

## Other callers

Any other module that moves money in or out of the wallet (a membership renewal, a gift card redemption,
a loyalty conversion) calls `wallet.spend`, `wallet.refund` or `wallet.topUp` with:

- a **ref derived from its own record** (`membership:<id>:2026-10`, `giftcard:<code>`), never a random id,
  so a retry replays;
- a **reason key** its UI can translate, with the parameters the sentence needs;
- an **actor** naming the module (`{ type: 'system', id: 'memberships' }`).

Nothing else writes balances. Seeds and data imports included: see [operations.md](operations.md).
