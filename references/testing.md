# Testing: setup, money and the engine

The templates ship with Vitest suites that hold every rule the other references state. Copy them into the
host's test folder with the code; they need no database and no network by default.

| Suite | Tests | Holds | Reference |
|---|---|---|---|
| `test/money.test.ts` | 15 | currency list parsing, exponents, Stripe amounts, form parsing, formatting, top-up limits | this file |
| `test/wallet.test.ts` | 20 | purity, idempotency, concurrency on one balance, currencies apart, holds, history order, reconciliation | this file |
| `test/stripe.test.ts` | 24 | top-up, webhook replay, split payment, sweeper, refunds, bookable-events adapter | [testing-payments.md](testing-payments.md) |
| `test/routes.test.ts` | 9 | auth, tenant scope, roles, feature gate, validation, replayed adjustments, fail-closed cron | [testing-payments.md](testing-payments.md) |
| `test/store-conformance.test.ts` | 8 per backend | the store contract, with real concurrency | [testing-stores.md](testing-stores.md) |
| `test/order-example.test.ts` | 1 per real backend | an order and its payment commit or roll back together | [testing-stores.md](testing-stores.md) |

Without a database, 76 tests run (the conformance suite on the in-memory store) and 2 are skipped. With the
Firestore emulator and a Postgres both configured, 94 run.

```bash
npx vitest run                                    # 76 pass, 2 skipped
FIRESTORE_EMULATOR_HOST=127.0.0.1:8080 \
WALLET_PG_URL=postgres://postgres@localhost:5432/wallet_test \
npx vitest run                                    # 94 pass
npx vitest run test/wallet.test.ts                # one suite
npx vitest run -t 'concurrent spends'             # one test
```

## Configuration

```ts
// vitest.config.ts: maps the '@' alias the templates import with.
import path from 'node:path';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  resolve: { alias: { '@': path.resolve(__dirname) } },
  // Real-backend suites run concurrent transactions; the emulator is slower than a database.
  test: { include: ['test/**/*.test.ts'], testTimeout: 20_000 },
});
```

The route tests import Next.js route handlers, so `next` must be installed (it is, in any host). The
Postgres suites import `pg`; a host on another driver swaps the import in the two test files.

## The in-memory store

A `WalletStore` for tests. Transactions are serialized and their writes buffered until the callback returns,
so a throw leaves nothing behind: the same all-or-nothing guarantee the real stores give, which the engine
tests depend on.

```ts
// lib/wallet/memory-store.ts: WalletStore for tests. Transactions are serialized and
// buffered, so a throw inside one leaves nothing behind, the same guarantee the real
// backends give, which is what the tests rely on.
import type { CurrencyCode } from './money';
import type { EntryQuery, WalletStore, WalletTx } from './ports';
import type { BalanceAmounts, BalanceState, CurrencyBalance, WalletEntry, WalletHold, WalletSnapshot } from './types';

interface WalletRow {
  preferredCurrency: CurrencyCode;
  balances: Map<CurrencyCode, BalanceState>;
}

export function createMemoryWalletStore(defaultCurrency: CurrencyCode = 'USD') {
  const wallets = new Map<string, WalletRow>();
  const entries = new Map<string, WalletEntry>(); // key: tenantId + ref
  const holds = new Map<string, WalletHold>();    // key: tenantId + holdRef
  let queue: Promise<unknown> = Promise.resolve();

  const walletKey = (t: string, c: string) => `${t}\u0000${c}`;
  const refKey = (t: string, r: string) => `${t}\u0000${r}`;

  const store: WalletStore = {
    runTransaction<T>(fn: (tx: WalletTx) => Promise<T>): Promise<T> {
      const run = async () => {
        const pending: Array<() => void> = [];
        const tx: WalletTx = {
          async readBalance(t, c, currency) {
            const b = wallets.get(walletKey(t, c))?.balances.get(currency);
            return b ? { ...b } : { available: 0, held: 0, lastMovementAt: null };
          },
          async readEntry(t, ref) {
            return entries.get(refKey(t, ref)) ?? null;
          },
          async readHold(t, holdRef) {
            const h = holds.get(refKey(t, holdRef));
            return h ? { ...h } : null;
          },
          async writeBalance(t, c, currency, next, at) {
            pending.push(() => {
              const key = walletKey(t, c);
              const row = wallets.get(key) ?? { preferredCurrency: defaultCurrency, balances: new Map() };
              row.balances.set(currency, { available: next.available, held: next.held, lastMovementAt: at });
              wallets.set(key, row);
            });
          },
          async createEntry(entry) {
            pending.push(() => {
              const key = refKey(entry.tenantId, entry.ref);
              if (entries.has(key)) throw new Error(`duplicate ref ${entry.ref}`);
              entries.set(key, { ...entry });
            });
          },
          async writeHold(hold) {
            pending.push(() => holds.set(refKey(hold.tenantId, hold.holdRef), { ...hold }));
          },
        };
        const result = await fn(tx);
        for (const apply of pending) apply(); // commit
        return result;
      };
      const next = queue.then(run, run);
      queue = next.catch(() => undefined);
      return next;
    },

    async getWallet(t, c): Promise<WalletSnapshot | null> {
      const row = wallets.get(walletKey(t, c));
      if (!row) return null;
      return {
        tenantId: t,
        customerId: c,
        preferredCurrency: row.preferredCurrency,
        balances: [...row.balances].map(([currency, b]) => ({ currency, available: b.available, held: b.held })),
      };
    },

    async setPreferredCurrency(t, c, currency) {
      const key = walletKey(t, c);
      const row = wallets.get(key) ?? { preferredCurrency: currency, balances: new Map() };
      row.preferredCurrency = currency;
      wallets.set(key, row);
    },

    async listEntries(q: EntryQuery) {
      const all = [...entries.values()]
        .filter((e) => e.tenantId === q.tenantId && e.customerId === q.customerId)
        .filter((e) => !q.currency || e.currency === q.currency)
        .sort((a, b) => b.createdAt.getTime() - a.createdAt.getTime() || b.id.localeCompare(a.id));
      const start = q.cursor ? all.findIndex((e) => e.id === q.cursor) + 1 : 0;
      const page = all.slice(start, start + q.limit);
      const last = page[page.length - 1];
      return { entries: page, nextCursor: start + q.limit < all.length && last ? last.id : null };
    },

    async listOpenHolds(olderThan, limit) {
      return [...holds.values()]
        .filter((h) => h.status === 'OPEN' && h.createdAt < olderThan)
        .slice(0, limit);
    },

    async sumEntries(t, c): Promise<CurrencyBalance[]> {
      const sums = new Map<CurrencyCode, CurrencyBalance>();
      for (const e of entries.values()) {
        if (e.tenantId !== t || e.customerId !== c) continue;
        const s = sums.get(e.currency) ?? { currency: e.currency, available: 0, held: 0 };
        s.available += e.availableDelta;
        s.held += e.heldDelta;
        sums.set(e.currency, s);
      }
      return [...sums.values()];
    },
  };

  /** Test hook: corrupt a balance directly, as a buggy writer outside the wallet would. */
  function forceBalance(t: string, c: string, currency: CurrencyCode, b: BalanceAmounts) {
    const key = walletKey(t, c);
    const row = wallets.get(key) ?? { preferredCurrency: defaultCurrency, balances: new Map() };
    row.balances.set(currency, { ...b, lastMovementAt: row.balances.get(currency)?.lastMovementAt ?? null });
    wallets.set(key, row);
  }

  return { store, forceBalance };
}
```

`forceBalance` exists for one purpose: simulating a writer that bypasses the ledger, so the reconciliation
test can prove the check catches it. Nothing outside tests may call it.

## Money

```ts
// test/money.test.ts
import { describe, expect, it } from 'vitest';
import {
  assertChargeableAmount, assertTopUpAmount, createCurrencyPolicy, formatMoney, fromStripeAmount,
  minorUnitExponent, normalizeCurrency, parseCurrencyList, parseMajorToMinor, toStripeAmount,
} from '@/lib/wallet/money';

describe('currency list (the skill argument)', () => {
  it('uppercases, keeps order, drops duplicates', () => {
    expect(parseCurrencyList('EUR,usd GBP; eur')).toEqual({ currencies: ['EUR', 'USD', 'GBP'], invalid: [] });
  });
  it('names invalid codes instead of silently dropping them', () => {
    expect(parseCurrencyList('USD XYZ 12')).toEqual({ currencies: ['USD'], invalid: ['XYZ', '12'] });
  });
  it('returns an empty list for no argument, the caller must ask', () => {
    expect(parseCurrencyList(undefined).currencies).toEqual([]);
    expect(parseCurrencyList('  ').currencies).toEqual([]);
  });
  it('makes the first code the default, USD when empty', () => {
    expect(createCurrencyPolicy(['EUR', 'USD']).defaultCurrency).toBe('EUR');
    expect(createCurrencyPolicy([]).allowed).toEqual(['USD']);
  });
  it('rejects unknown codes at startup', () => {
    expect(() => createCurrencyPolicy(['USD', 'ABC'])).toThrow(/ABC/);
  });
  it('normalizes a single code', () => {
    expect(normalizeCurrency(' isk ')).toBe('ISK');
    expect(normalizeCurrency('US')).toBeNull();
    expect(normalizeCurrency(42)).toBeNull();
  });
});

describe('minor units and Stripe amounts', () => {
  it('knows exponents', () => {
    expect([minorUnitExponent('USD'), minorUnitExponent('JPY'), minorUnitExponent('ISK'), minorUnitExponent('KWD')])
      .toEqual([2, 0, 0, 3]);
  });
  it('stores ISK in whole kronur and sends Stripe x100', () => {
    expect(toStripeAmount(500, 'ISK')).toBe(50_000);
    expect(fromStripeAmount(50_000, 'ISK')).toBe(500);
  });
  it('passes other currencies through unchanged', () => {
    for (const c of ['USD', 'JPY', 'KWD']) {
      expect(toStripeAmount(1250, c)).toBe(1250);
      expect(fromStripeAmount(1250, c)).toBe(1250);
    }
  });
  it('rejects amounts Stripe would reject', () => {
    expect(() => assertChargeableAmount(49, 'USD')).toThrow(/minimum/);
    expect(() => assertChargeableAmount(1255, 'KWD')).toThrow(/end in 0/);
    expect(() => assertChargeableAmount(1250, 'KWD')).not.toThrow();
  });
  it('parses form input without float error', () => {
    expect(parseMajorToMinor('19.99', 'USD')).toBe(1999);
    expect(parseMajorToMinor('0,1', 'EUR')).toBe(10);
    expect(parseMajorToMinor('500', 'ISK')).toBe(500);
    expect(parseMajorToMinor('5.5', 'ISK')).toBeNull();
    expect(parseMajorToMinor('1.250', 'KWD')).toBe(1250);
    expect(parseMajorToMinor('-3', 'USD')).toBeNull();
    expect(parseMajorToMinor('1e3', 'USD')).toBeNull();
  });
  it('formats with the decimals of the currency', () => {
    const plain = (s: string) => s.replace(/\s/g, ' '); // Intl puts a no-break space after the code
    expect(formatMoney(1999, 'USD')).toBe('$19.99');
    expect(plain(formatMoney(1500, 'ISK', 'en-US'))).toBe('ISK 1,500');
    expect(plain(formatMoney(1250, 'KWD', 'en-US'))).toBe('KWD 1.250');
  });
});

describe('top-up limits', () => {
  const policy = createCurrencyPolicy(['USD', 'HUF'], { HUF: { min: 100, max: 5_000_000 } });
  it('enforces per-currency bounds', () => {
    expect(() => assertTopUpAmount(policy, 'USD', 499)).toThrow(/between/);
    expect(() => assertTopUpAmount(policy, 'USD', 100_001)).toThrow(/between/);
    expect(assertTopUpAmount(policy, 'USD', 2_000)).toBe(2_000);
  });
  it('never lets a configured minimum sit below the Stripe minimum', () => {
    expect(policy.topUpLimits.HUF?.min).toBe(17_500);
  });
  it('rejects non-integers and strings', () => {
    expect(() => assertTopUpAmount(policy, 'USD', 10.5)).toThrow(/integer/);
    expect(() => assertTopUpAmount(policy, 'USD', '1000')).toThrow(/integer/);
  });
});
```

## The engine

```ts
// test/wallet.test.ts
import { beforeEach, describe, expect, it } from 'vitest';
import { applyCommand } from '@/lib/wallet/apply';
import { createMemoryWalletStore } from '@/lib/wallet/memory-store';
import { createCurrencyPolicy } from '@/lib/wallet/money';
import { balanceRows, createWallet, type Wallet } from '@/lib/wallet/wallet';
import type { Actor } from '@/lib/wallet/types';

const T = 'tenant-a';
const C = 'cust-1';
const actor: Actor = { type: 'system', id: 'test' };
const reason = { key: 'wallet.reason.test' };

let wallet: Wallet;
let mem: ReturnType<typeof createMemoryWalletStore>;
beforeEach(() => {
  mem = createMemoryWalletStore('USD');
  wallet = createWallet({ store: mem.store, policy: createCurrencyPolicy(['USD', 'EUR']) });
});

const topUp = (amount: number, ref: string, currency = 'USD') =>
  wallet.topUp({ tenantId: T, customerId: C, currency, amount, ref, reason, actor });
const available = async (currency = 'USD') =>
  (await wallet.getWallet(T, C)).balances.find((b) => b.currency === currency)?.available ?? 0;

describe('applyCommand is pure', () => {
  it('does not mutate its input and returns the same result twice', () => {
    const balance = Object.freeze({ available: 1000, held: 0 });
    const a = applyCommand(balance, { kind: 'HOLD', amount: 300 });
    const b = applyCommand(balance, { kind: 'HOLD', amount: 300 });
    expect(a).toEqual(b);
    expect(a.next).toEqual({ available: 700, held: 300 });
    expect(balance).toEqual({ available: 1000, held: 0 });
  });
  it('rejects zero, negative and fractional amounts', () => {
    for (const amount of [0, -5, 1.5, Number.NaN]) {
      expect(() => applyCommand({ available: 0, held: 0 }, { kind: 'TOP_UP', amount })).toThrow(/integer/);
    }
  });
});

describe('idempotency', () => {
  it('a retried ref credits once', async () => {
    const first = await topUp(1000, 'stripe:cs:1');
    const second = await topUp(1000, 'stripe:cs:1');
    expect(first.replayed).toBe(false);
    expect(second.replayed).toBe(true);
    expect(second.entry.id).toBe(first.entry.id);
    expect(await available()).toBe(1000);
  });
  it('concurrent duplicates credit once', async () => {
    await Promise.all([topUp(1000, 'dup'), topUp(1000, 'dup'), topUp(1000, 'dup')]);
    expect(await available()).toBe(1000);
  });
  it('a ref reused with a different amount is a conflict, not a silent no-op', async () => {
    await topUp(1000, 'r1');
    await expect(topUp(2000, 'r1')).rejects.toMatchObject({ code: 'REF_CONFLICT' });
  });
  it('refs are scoped per tenant', async () => {
    await topUp(1000, 'same-ref');
    await wallet.topUp({ tenantId: 'tenant-b', customerId: C, currency: 'USD', amount: 700, ref: 'same-ref', reason, actor });
    expect(await available()).toBe(1000);
    expect((await wallet.getWallet('tenant-b', C)).balances[0]?.available).toBe(700);
  });
});

describe('balances', () => {
  it('keeps currencies separate and never converts', async () => {
    await topUp(1000, 'a', 'USD');
    await topUp(500, 'b', 'EUR');
    await expect(
      wallet.spend({ tenantId: T, customerId: C, currency: 'EUR', amount: 800, ref: 'c', reason, actor }),
    ).rejects.toMatchObject({ code: 'INSUFFICIENT_FUNDS', details: { available: 500, required: 800 } });
    expect(await available('USD')).toBe(1000);
  });
  it('concurrent spends cannot overdraw', async () => {
    await topUp(1000, 'seed');
    const results = await Promise.allSettled(
      [1, 2, 3].map((i) => wallet.spend({ tenantId: T, customerId: C, currency: 'USD', amount: 400, ref: `s${i}`, reason, actor })),
    );
    expect(results.filter((r) => r.status === 'fulfilled')).toHaveLength(2);
    expect(await available()).toBe(200);
  });
  it('keeps a disabled currency spendable, and still credits a top-up paid before it was disabled', async () => {
    await topUp(900, 'eur-seed', 'EUR');
    const narrowed = createWallet({ store: mem.store, policy: createCurrencyPolicy(['USD']) });
    await narrowed.spend({ tenantId: T, customerId: C, currency: 'EUR', amount: 900, ref: 'y', reason, actor });
    // The webhook for a Checkout opened while EUR was enabled lands after the switch.
    await narrowed.topUp({ tenantId: T, customerId: C, currency: 'EUR', amount: 500, ref: 'late', reason, actor });
    expect(await available('EUR')).toBe(500);
    await expect(narrowed.topUp({ tenantId: T, customerId: C, currency: 'XXQ', amount: 1, ref: 'z', reason, actor }))
      .rejects.toMatchObject({ code: 'INVALID_CURRENCY' });
  });
  it('lists allowed currencies zero-filled and flags disabled ones that hold money', async () => {
    mem.forceBalance(T, C, 'GBP', { available: 300, held: 0 });
    const rows = balanceRows(await wallet.getWallet(T, C), createCurrencyPolicy(['USD', 'EUR']));
    expect(rows.map((r) => [r.currency, r.available, r.disabled])).toEqual([
      ['USD', 0, false], ['EUR', 0, false], ['GBP', 300, true],
    ]);
  });
  it('adjustments are signed and cannot go below zero', async () => {
    await topUp(1000, 'seed');
    const staff: Actor = { type: 'staff', id: 'staff-9' };
    await wallet.adjust({ tenantId: T, customerId: C, currency: 'USD', delta: -300, ref: 'adj:1', reason, actor: staff });
    await expect(
      wallet.adjust({ tenantId: T, customerId: C, currency: 'USD', delta: -800, ref: 'adj:2', reason, actor: staff }),
    ).rejects.toMatchObject({ code: 'INSUFFICIENT_FUNDS' });
    expect(await available()).toBe(700);
  });
});

describe('holds', () => {
  beforeEach(async () => {
    await topUp(1000, 'seed');
    await wallet.hold({ tenantId: T, customerId: C, currency: 'USD', amount: 600, ref: 'order:42:hold', reason, actor, orderRef: '42' });
  });
  it('moves money from available to held', async () => {
    const w = await wallet.getWallet(T, C);
    expect(w.balances[0]).toMatchObject({ available: 400, held: 600 });
  });
  it('capture spends the held part; replay is a no-op; release afterwards is refused', async () => {
    await wallet.capture({ tenantId: T, customerId: C, holdRef: 'order:42:hold', reason, actor });
    const again = await wallet.capture({ tenantId: T, customerId: C, holdRef: 'order:42:hold', reason, actor });
    expect(again.replayed).toBe(true);
    await expect(wallet.release({ tenantId: T, customerId: C, holdRef: 'order:42:hold', reason, actor }))
      .rejects.toMatchObject({ code: 'HOLD_NOT_OPEN' });
    expect((await wallet.getWallet(T, C)).balances[0]).toMatchObject({ available: 400, held: 0 });
  });
  it('release returns the held part; capture afterwards is refused', async () => {
    await wallet.release({ tenantId: T, customerId: C, holdRef: 'order:42:hold', reason, actor });
    await expect(wallet.capture({ tenantId: T, customerId: C, holdRef: 'order:42:hold', reason, actor }))
      .rejects.toMatchObject({ code: 'HOLD_NOT_OPEN' });
    expect((await wallet.getWallet(T, C)).balances[0]).toMatchObject({ available: 1000, held: 0 });
  });
  it("another customer's hold is not found", async () => {
    await expect(wallet.capture({ tenantId: T, customerId: 'someone-else', holdRef: 'order:42:hold', reason, actor }))
      .rejects.toMatchObject({ code: 'HOLD_NOT_FOUND' });
  });
});

describe('history and reconciliation', () => {
  it('pages newest first with a cursor', async () => {
    for (let i = 1; i <= 5; i++) await topUp(100 * i, `p${i}`);
    const page1 = await wallet.listEntries({ tenantId: T, customerId: C, limit: 2 });
    const page2 = await wallet.listEntries({ tenantId: T, customerId: C, limit: 2, cursor: page1.nextCursor });
    const page3 = await wallet.listEntries({ tenantId: T, customerId: C, limit: 2, cursor: page2.nextCursor });
    expect([...page1.entries, ...page2.entries, ...page3.entries]).toHaveLength(5);
    expect(page3.nextCursor).toBeNull();
  });
  it('orders history exactly even when the clock does not move', async () => {
    const frozen = new Date('2026-01-01T00:00:00Z');
    const w = createWallet({ store: mem.store, policy: createCurrencyPolicy(['USD']), now: () => frozen });
    for (const [i, amount] of [100, 200, 300].entries()) {
      await w.topUp({ tenantId: T, customerId: C, currency: 'USD', amount, ref: `f${i}`, reason, actor });
    }
    const { entries } = await w.listEntries({ tenantId: T, customerId: C });
    expect(entries.map((e) => e.availableAfter)).toEqual([600, 300, 100]);
    expect(entries[0]!.createdAt.getTime() - entries[2]!.createdAt.getTime()).toBe(2);
  });
  it('every entry records the balance after it', async () => {
    await topUp(1000, 'a');
    const { entry } = await wallet.spend({ tenantId: T, customerId: C, currency: 'USD', amount: 250, ref: 'b', reason, actor });
    expect(entry).toMatchObject({ kind: 'SPEND', amount: 250, availableDelta: -250, availableAfter: 750 });
  });
  it('reconcile is clean after normal use and catches a write that bypassed the ledger', async () => {
    await topUp(1000, 'a');
    await wallet.hold({ tenantId: T, customerId: C, currency: 'USD', amount: 300, ref: 'h', reason, actor });
    expect(await wallet.reconcile(T, C)).toEqual([]);
    mem.forceBalance(T, C, 'USD', { available: 5000, held: 300 }); // e.g. a seed script
    expect(await wallet.reconcile(T, C)).toHaveLength(1);
  });
  it('the preferred currency defaults to the policy default and must be allowed', async () => {
    expect((await wallet.getWallet(T, C)).preferredCurrency).toBe('USD');
    await wallet.setPreferredCurrency(T, C, 'eur');
    expect((await wallet.getWallet(T, C)).preferredCurrency).toBe('EUR');
    await expect(wallet.setPreferredCurrency(T, C, 'GBP')).rejects.toMatchObject({ code: 'CURRENCY_NOT_ALLOWED' });
  });
});
```

## What these suites guarantee

| Rule | Test |
|---|---|
| A ref applies once, even when five requests race | `concurrent duplicates credit once` |
| Reusing a ref for something else is an error, not a no-op | `a ref reused with a different amount is a conflict` |
| Two tenants never share an entry | `refs are scoped per tenant` |
| No overdraft under concurrent spends | `concurrent spends cannot overdraw` |
| Currencies never mix | `keeps currencies separate and never converts` |
| Delisting a currency strands nothing | `keeps a disabled currency spendable...` |
| A hold settles exactly once | the three `holds` tests |
| History order is exact even on a frozen clock | `orders history exactly even when the clock does not move` |
| A balance written outside the ledger is detected | `reconcile is clean after normal use and catches a write that bypassed the ledger` |
| The preference defaults to the first listed currency and must be listed | `the preferred currency defaults...` |
