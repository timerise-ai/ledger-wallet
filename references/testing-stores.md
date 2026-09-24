# Testing: stores against real backends

The in-memory store proves the engine's logic. It cannot prove that a Firestore transaction retries, that
`FOR UPDATE` serializes, or that a unique constraint fires under a race. These two suites run the same
contract against every configured backend, concurrency cases included, with real concurrency.

| Backend | Enabled by | Needs |
|---|---|---|
| memory | always | nothing |
| Firestore | `FIRESTORE_EMULATOR_HOST=127.0.0.1:8080` | `firebase emulators:start --only firestore --project demo-wallet` |
| Postgres | `WALLET_PG_URL=postgres://...` | a database with `db/wallet.sql` applied ([postgres.md](postgres.md)) |

Each test uses a fresh random tenant id, so the suites can run repeatedly against a shared staging database
without cleanup.

## The store contract

```ts
// test/store-conformance.test.ts: the same guarantees, against every backend.
//
//   npx vitest run test/store-conformance.test.ts                         # memory only
//   FIRESTORE_EMULATOR_HOST=127.0.0.1:8080 npx vitest run test/store-...  # + Firestore emulator
//   WALLET_PG_URL=postgres://postgres@localhost:5432/wallet_test npx vitest run ...  # + Postgres (db/wallet.sql applied)
//
// Real backends run the concurrency cases with real concurrency, the point of this file.
import { randomUUID } from 'node:crypto';
import { afterAll, beforeEach, describe, expect, it } from 'vitest';
import { createFirestoreWalletStore } from '@/lib/wallet/firestore-store';
import { createMemoryWalletStore } from '@/lib/wallet/memory-store';
import { createCurrencyPolicy } from '@/lib/wallet/money';
import type { WalletStore } from '@/lib/wallet/ports';
import { createPostgresWalletStore, postgresWalletTx, type SqlPool } from '@/lib/wallet/postgres-store';
import type { Actor } from '@/lib/wallet/types';
import { createWallet, type Wallet } from '@/lib/wallet/wallet';

interface Backend {
  store: WalletStore;
  pool?: SqlPool;
  close(): Promise<void>;
}
const backends: Array<[string, () => Promise<Backend>]> = [
  ['memory', async () => ({ store: createMemoryWalletStore('USD').store, close: async () => undefined })],
];
if (process.env.FIRESTORE_EMULATOR_HOST) {
  backends.push(['firestore', async () => {
    const { initializeApp, getApps } = await import('firebase-admin/app');
    const { getFirestore } = await import('firebase-admin/firestore');
    const app = getApps()[0] ?? initializeApp({ projectId: 'demo-wallet' });
    return { store: createFirestoreWalletStore(getFirestore(app), 'USD'), close: async () => undefined };
  }]);
}
if (process.env.WALLET_PG_URL) {
  backends.push(['postgres', async () => {
    const { Pool } = await import('pg');
    const pool = new Pool({ connectionString: process.env.WALLET_PG_URL, max: 10 });
    return { store: createPostgresWalletStore(pool, 'USD'), pool, close: () => pool.end() };
  }]);
}

const actor: Actor = { type: 'system', id: 'conformance' };
const reason = { key: 'wallet.reason.test' };

describe.each(backends)('%s store', (_name, make) => {
  let backend: Backend;
  let wallet: Wallet;
  let T: string; // a fresh tenant per test isolates runs against a shared database
  const C = 'cust-1';

  beforeEach(async () => {
    backend ??= await make();
    wallet = createWallet({ store: backend.store, policy: createCurrencyPolicy(['USD', 'EUR']) });
    T = `t-${randomUUID()}`;
  });
  afterAll(async () => backend?.close());

  const credit = (amount: number, ref: string, currency = 'USD') =>
    wallet.topUp({ tenantId: T, customerId: C, currency, amount, ref, reason, actor });
  const bal = async (currency = 'USD') =>
    (await wallet.getWallet(T, C)).balances.find((b) => b.currency === currency) ?? { available: 0, held: 0 };

  it('concurrent duplicates of one ref apply once', async () => {
    const results = await Promise.all(Array.from({ length: 5 }, () => credit(1000, 'stripe:cs:dup')));
    expect(results.filter((r) => !r.replayed)).toHaveLength(1);
    expect((await bal()).available).toBe(1000);
  });

  it('concurrent spends never overdraw', async () => {
    await credit(1000, 'seed');
    const results = await Promise.allSettled(
      Array.from({ length: 4 }, (_, i) =>
        wallet.spend({ tenantId: T, customerId: C, currency: 'USD', amount: 300, ref: `s${i}`, reason, actor })),
    );
    expect(results.filter((r) => r.status === 'fulfilled')).toHaveLength(3);
    expect((await bal()).available).toBe(100);
  });

  it('a racing capture and release settle a hold exactly once', async () => {
    await credit(1000, 'seed');
    await wallet.hold({ tenantId: T, customerId: C, currency: 'USD', amount: 600, ref: 'h1', reason, actor });
    const settle = { tenantId: T, customerId: C, holdRef: 'h1', reason, actor };
    const results = await Promise.allSettled([wallet.capture(settle), wallet.release(settle)]);
    expect(results.filter((r) => r.status === 'fulfilled')).toHaveLength(1);
    const b = await bal();
    expect(b.held).toBe(0);
    expect([400, 1000]).toContain(b.available);
    expect(await wallet.reconcile(T, C)).toEqual([]);
  });

  it('pages history newest first without gaps or repeats', async () => {
    for (let i = 1; i <= 5; i++) await credit(100 * i, `p${i}`);
    const seen: number[] = [];
    let cursor: string | null = null;
    do {
      const page = await wallet.listEntries({ tenantId: T, customerId: C, limit: 2, cursor });
      seen.push(...page.entries.map((e) => e.amount));
      cursor = page.nextCursor;
    } while (cursor);
    expect(seen).toEqual([500, 400, 300, 200, 100]);
  });

  it('filters history by currency and keeps currencies apart', async () => {
    await credit(1000, 'u', 'USD');
    await credit(700, 'e', 'EUR');
    const eur = await wallet.listEntries({ tenantId: T, customerId: C, currency: 'eur' });
    expect(eur.entries.map((e) => e.currency)).toEqual(['EUR']);
    expect((await bal('EUR')).available).toBe(700);
  });

  it('reconciles and reports open holds', async () => {
    await credit(1000, 'seed');
    await wallet.hold({ tenantId: T, customerId: C, currency: 'USD', amount: 250, ref: 'h2', reason, actor });
    expect(await wallet.reconcile(T, C)).toEqual([]);
    const open = await backend.store.listOpenHolds(new Date(Date.now() + 60_000), 1000);
    expect(open.some((h) => h.tenantId === T && h.holdRef === 'h2' && h.amount === 250)).toBe(true);
  });

  it('stores the preferred currency', async () => {
    await wallet.setPreferredCurrency(T, C, 'EUR');
    expect((await wallet.getWallet(T, C)).preferredCurrency).toBe('EUR');
  });

  it('a movement inside the transaction of the caller rolls back with it', async () => {
    if (!backend.pool) return; // Postgres-specific; Firestore's equivalent is firestoreWalletTx
    await credit(1000, 'seed');
    const client = await backend.pool.connect();
    try {
      await client.query('begin');
      await wallet.postInTx(postgresWalletTx(client, 'USD'), {
        tenantId: T, customerId: C, currency: 'USD', ref: 'order:77', command: { kind: 'SPEND', amount: 400 }, reason, actor,
      });
      await client.query('rollback'); // e.g. the order insert failed on a capacity check
    } finally {
      client.release();
    }
    expect((await bal()).available).toBe(1000);
    expect(await wallet.getEntry(T, 'order:77')).toBeNull();
  });
});
```

| Contract | Why it matters on a real backend |
|---|---|
| Five concurrent posts of one ref apply once | Firestore must retry and replay; Postgres must serialize on the row lock |
| Four concurrent 300 spends from 1000 leave 100 | the overdraft guard holds under contention, not only in sequence |
| Capture and release racing on one hold: exactly one wins | the hold's status is read under a lock or retried on conflict |
| History pages without gaps or repeats | the cursor and the index agree on the sort order |
| A movement inside the caller's rolled-back transaction leaves nothing | `postgresWalletTx` really joins the caller's transaction |

## Orders pay atomically

```ts
// test/order-example.test.ts: the order examples against real backends: a sold-out slot
// moves no money, and a short wallet creates no order. Skipped without a backend.
import { randomUUID } from 'node:crypto';
import { describe, expect, it } from 'vitest';
import { createCurrencyPolicy } from '@/lib/wallet/money';
import { createFirestoreWalletStore } from '@/lib/wallet/firestore-store';
import { createPostgresWalletStore, type SqlPool } from '@/lib/wallet/postgres-store';
import { createWallet } from '@/lib/wallet/wallet';
import { createOrderPaidFromWallet as payFs } from '@/lib/orders/pay-from-wallet.firestore';
import { createOrderPaidFromWallet as payPg } from '@/lib/orders/pay-from-wallet.postgres';

const actor = { type: 'system', id: 't' } as const;
const reason = { key: 'k' };

describe.skipIf(!process.env.WALLET_PG_URL)('postgres order example', () => {
  it('is all-or-nothing', async () => {
    const { Pool } = await import('pg');
    const pool = new Pool({ connectionString: process.env.WALLET_PG_URL }) as unknown as SqlPool & { end(): Promise<void> };
    await pool.query(`create table if not exists slots (id text primary key, booked int not null, capacity int not null)`);
    await pool.query(`create table if not exists orders (id text primary key, tenant_id text, customer_id text, status text,
      currency text, total bigint, wallet_paid bigint, card_paid bigint, wallet_entry_id text)`);
    const wallet = createWallet({ store: createPostgresWalletStore(pool, 'USD'), policy: createCurrencyPolicy(['USD']) });
    const T = `t-${randomUUID()}`;
    const slot = `s-${randomUUID()}`;
    await pool.query(`insert into slots values ($1, 0, 1)`, [slot]);
    await wallet.topUp({ tenantId: T, customerId: 'c', currency: 'USD', amount: 1500, ref: 'seed', reason, actor });
    const order = (id: string, total: number) =>
      payPg(pool, wallet, { tenantId: T, customerId: 'c', orderId: `${T}-${id}`, slotId: slot, currency: 'USD', total });

    await expect(order('o-short', 5000)).rejects.toMatchObject({ code: 'INSUFFICIENT_FUNDS' });
    await order('o1', 1000);
    await expect(order('o2', 100)).rejects.toThrow('SOLD_OUT');
    const bal = (await wallet.getWallet(T, 'c')).balances[0];
    expect(bal?.available).toBe(500);
    const { rows } = await pool.query(`select id from orders where tenant_id = $1 order by id`, [T]);
    expect(rows.map((r) => r.id)).toEqual([`${T}-o1`]);
    await pool.end();
  });
});

describe.skipIf(!process.env.FIRESTORE_EMULATOR_HOST)('firestore order example', () => {
  it('is all-or-nothing', async () => {
    const { initializeApp, getApps } = await import('firebase-admin/app');
    const { getFirestore } = await import('firebase-admin/firestore');
    const db = getFirestore(getApps()[0] ?? initializeApp({ projectId: 'demo-wallet' }));
    const wallet = createWallet({ store: createFirestoreWalletStore(db, 'USD'), policy: createCurrencyPolicy(['USD']) });
    const T = `t-${randomUUID()}`;
    const slot = `s-${randomUUID()}`;
    await db.collection('slots').doc(slot).set({ booked: 0, capacity: 1 });
    await wallet.topUp({ tenantId: T, customerId: 'c', currency: 'USD', amount: 1500, ref: 'seed', reason, actor });
    const order = (id: string, total: number) =>
      payFs(db, wallet, { tenantId: T, customerId: 'c', orderId: `${T}-${id}`, slotId: slot, currency: 'USD', total });

    await expect(order('short', 5000)).rejects.toMatchObject({ code: 'INSUFFICIENT_FUNDS' });
    await order('o1', 1000);
    await expect(order('o2', 100)).rejects.toThrow();
    expect((await wallet.getWallet(T, 'c')).balances[0]?.available).toBe(500);
    expect((await db.collection('orders').doc(`${T}-short`).get()).exists).toBe(false);
    expect((await db.collection('orders').doc(`${T}-o2`).get()).exists).toBe(false);
  });
});
```

These run the two examples of [integration.md](integration.md) as written. The Postgres test creates its own
`slots` and `orders` tables if they are missing; the Firestore test writes to `slots` and `orders` in the
emulator. Point them at the host's real order code once it exists.

## Running the backends locally

```bash
# Firestore emulator (needs Java)
firebase emulators:start --only firestore --project demo-wallet

# Postgres (any local install, or a container)
createdb wallet_test && psql -d wallet_test -v ON_ERROR_STOP=1 -f db/wallet.sql

FIRESTORE_EMULATOR_HOST=127.0.0.1:8080 WALLET_PG_URL=postgres://localhost/wallet_test npx vitest run
```

In CI, run the Postgres suite against a service container and the Firestore suite with the emulator; both
finish in seconds apart from the Firestore contention tests, which wait out the emulator's transaction
retries.
