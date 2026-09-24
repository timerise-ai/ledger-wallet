# Engine

Three files make the wallet: a **pure function** that does all the arithmetic, a **port** that a backend
implements, and a **service** that runs every command through the same five steps inside one transaction.

```
 post(input)                                  postInTx(tx, input)   <- caller's own transaction
   |                                             |
   v                                             v
 store.runTransaction(tx => postInTx(tx, input))
   1. resolve currency (and the hold, for CAPTURE / RELEASE)      reads
   2. readBalance (locks the row)  then  readEntry(ref)           reads
      ref exists?  same payload: return it, replayed = true
                   different payload: REF_CONFLICT
   3. applyCommand(balance, command)                              pure: guards and arithmetic
   4. writeBalance, createEntry, writeHold                        writes, after every read
   5. commit (or nothing at all)
```

## The pure core

```ts
// lib/wallet/apply.ts: every balance change, as one pure function.
//
// No I/O, no clock, no mutation of its inputs: the engine calls it inside a
// transaction that a database may retry, so it must return the same answer
// for the same input every time.
import { WalletError } from './errors';
import { isMinorUnitAmount } from './money';
import type { BalanceAmounts, WalletCommand } from './types';

/** CAPTURE / RELEASE carry the hold's amount once the engine has looked it up. */
export type ResolvedCommand =
  | Exclude<WalletCommand, { kind: 'CAPTURE' | 'RELEASE' }>
  | { kind: 'CAPTURE' | 'RELEASE'; amount: number };

export interface Movement {
  amount: number;
  availableDelta: number;
  heldDelta: number;
  next: BalanceAmounts;
}

export function applyCommand(balance: BalanceAmounts, command: ResolvedCommand): Movement {
  if (command.kind === 'ADJUSTMENT') {
    const { delta } = command;
    if (!Number.isSafeInteger(delta) || delta === 0) {
      throw new WalletError('INVALID_AMOUNT', 'Adjustment must be a non-zero integer', { delta });
    }
    return move(balance, Math.abs(delta), delta, 0);
  }

  const { amount } = command;
  if (!isMinorUnitAmount(amount)) {
    throw new WalletError('INVALID_AMOUNT', 'Amount must be a positive integer in minor units', { amount });
  }
  switch (command.kind) {
    case 'TOP_UP':
    case 'REFUND':
      return move(balance, amount, amount, 0);
    case 'SPEND':
      return move(balance, amount, -amount, 0);
    case 'HOLD':
      return move(balance, amount, -amount, amount);
    case 'CAPTURE':
      return move(balance, amount, 0, -amount);
    case 'RELEASE':
      return move(balance, amount, amount, -amount);
  }
}

function move(balance: BalanceAmounts, amount: number, availableDelta: number, heldDelta: number): Movement {
  const next = { available: balance.available + availableDelta, held: balance.held + heldDelta };
  if (next.available < 0) {
    throw new WalletError('INSUFFICIENT_FUNDS', 'Insufficient balance', {
      available: balance.available,
      required: -availableDelta,
    });
  }
  if (next.held < 0) {
    // Only reachable if the holds table and the balance disagree, a bug, not a user error.
    throw new Error(`Held balance would go negative (${balance.held} + ${heldDelta})`);
  }
  if (!Number.isSafeInteger(next.available) || !Number.isSafeInteger(next.held)) {
    throw new WalletError('INVALID_AMOUNT', 'Balance would overflow', { amount });
  }
  return { amount, availableDelta, heldDelta, next };
}
```

`applyCommand` never reads the clock, never mutates its input and never touches a database. A backend may
run the transaction callback more than once (Firestore retries on contention), so the same input must give
the same result every time; `test/wallet.test.ts` checks it on a frozen input.

## The port

```ts
// lib/wallet/ports.ts: what a backend must provide. Firestore and Postgres
// implementations ship in the skill; tests use the in-memory one.
import type { CurrencyCode } from './money';
import type {
  BalanceAmounts, BalanceState, CurrencyBalance, EntryPage, WalletEntry, WalletHold, WalletSnapshot,
} from './types';

/**
 * One database transaction. Contract:
 * - Every `read*` happens before any `write*` (Firestore requires it; the engine obeys it).
 * - `readBalance` locks the row where the backend can (Postgres `FOR UPDATE`), so call it
 *   before `readEntry`: a concurrent duplicate then waits and sees the committed entry.
 * - Writes become visible atomically at commit, or not at all.
 */
export interface WalletTx {
  readBalance(tenantId: string, customerId: string, currency: CurrencyCode): Promise<BalanceState>;
  readEntry(tenantId: string, ref: string): Promise<WalletEntry | null>;
  readHold(tenantId: string, holdRef: string): Promise<WalletHold | null>;
  /** `at` becomes the balance's lastMovementAt, the same instant as the entry's createdAt. */
  writeBalance(tenantId: string, customerId: string, currency: CurrencyCode, next: BalanceAmounts, at: Date): Promise<void>;
  /** Must fail if an entry with the same (tenantId, ref) exists, the last line of idempotency. */
  createEntry(entry: WalletEntry): Promise<void>;
  writeHold(hold: WalletHold): Promise<void>;
}

export interface EntryQuery {
  tenantId: string;
  customerId: string;
  currency?: CurrencyCode;
  cursor?: string | null;
  limit: number;
}

export interface WalletStore {
  runTransaction<T>(fn: (tx: WalletTx) => Promise<T>): Promise<T>;
  getWallet(tenantId: string, customerId: string): Promise<WalletSnapshot | null>;
  setPreferredCurrency(tenantId: string, customerId: string, currency: CurrencyCode): Promise<void>;
  /** Newest first. The cursor is opaque to callers. */
  listEntries(query: EntryQuery): Promise<EntryPage>;
  /** OPEN holds created before `olderThan`, across all tenants, for the sweeper. */
  listOpenHolds(olderThan: Date, limit: number): Promise<WalletHold[]>;
  /** Sum of availableDelta and of heldDelta per currency, for reconciliation. */
  sumEntries(tenantId: string, customerId: string): Promise<CurrencyBalance[]>;
}
```

The order of reads in the contract is load-bearing:

- **Balance before entry.** On Postgres, `readBalance` takes `SELECT ... FOR UPDATE`. Two requests with the
  same ref serialize on that lock, and the second then sees the first one's committed entry and replays it.
  Reading the entry first would let both pass the check before either locked.
- **All reads before all writes.** Firestore rejects a read after a write in the same transaction. The engine
  obeys this, and so must a caller that shares its transaction (see [integration.md](integration.md)).
- **`createEntry` must fail on a duplicate.** Firestore `create()`, a Postgres unique constraint. It is the
  last line of idempotency if every check above it were somehow bypassed.

## The service

```ts
// lib/wallet/wallet.ts: the wallet service. Every change goes through `postInTx`:
// read, replay check, pure apply, atomic write. Retrying any call with the same
// `ref` is a no-op that returns the original entry.
import { createHash } from 'node:crypto';
import { applyCommand, type ResolvedCommand } from './apply';
import { WalletError } from './errors';
import { normalizeCurrency, requireAllowedCurrency, type CurrencyCode, type CurrencyPolicy } from './money';
import type { WalletStore, WalletTx } from './ports';
import type {
  Actor, CurrencyBalance, EntryPage, PostInput, PostResult, Reason, WalletEntry, WalletHold, WalletSnapshot,
} from './types';

export interface WalletDeps {
  store: WalletStore;
  policy: CurrencyPolicy;
  now?: () => Date;
}

/** Stable, path-safe entry id: refs may contain '/', which Firestore ids cannot. */
export function entryIdFor(tenantId: string, ref: string): string {
  return createHash('sha256').update(`${tenantId}\u0000${ref}`).digest('hex').slice(0, 40);
}

export const captureRefFor = (holdRef: string) => `${holdRef}:capture`;
export const releaseRefFor = (holdRef: string) => `${holdRef}:release`;

export interface BalanceRow extends CurrencyBalance {
  /** Held in a currency no longer in WALLET_CURRENCIES: still shown and spendable, no top-ups. */
  disabled: boolean;
}

/** Allowed currencies zero-filled, in policy order, then any other currency with money in it. */
export function balanceRows(snapshot: WalletSnapshot | null, policy: CurrencyPolicy): BalanceRow[] {
  const held = new Map((snapshot?.balances ?? []).map((b) => [b.currency, b]));
  const rows: BalanceRow[] = policy.allowed.map((currency) => ({
    currency,
    available: held.get(currency)?.available ?? 0,
    held: held.get(currency)?.held ?? 0,
    disabled: false,
  }));
  for (const b of snapshot?.balances ?? []) {
    if (!policy.allowed.includes(b.currency) && (b.available !== 0 || b.held !== 0)) {
      rows.push({ ...b, disabled: true });
    }
  }
  return rows;
}

interface MoveInput {
  tenantId: string;
  customerId: string;
  currency: CurrencyCode;
  amount: number;
  ref: string;
  reason: Reason;
  actor: Actor;
  orderRef?: string | null;
  externalRef?: string | null;
}

interface SettleInput {
  tenantId: string;
  customerId: string;
  holdRef: string;
  reason: Reason;
  actor: Actor;
  externalRef?: string | null;
}

export type Wallet = ReturnType<typeof createWallet>;

export function createWallet({ store, policy, now = () => new Date() }: WalletDeps) {
  async function postInTx(tx: WalletTx, input: PostInput): Promise<PostResult> {
    const { tenantId, customerId, ref, command } = input;
    if (!ref || ref.length > 200) throw new WalletError('INVALID_AMOUNT', 'ref must be 1 to 200 characters', { ref });

    // 1. Resolve the currency (and the hold, for CAPTURE / RELEASE). Reads only.
    let hold: WalletHold | null = null;
    let currency: CurrencyCode;
    if (command.kind === 'CAPTURE' || command.kind === 'RELEASE') {
      if (!input.holdRef) throw new WalletError('HOLD_NOT_FOUND', 'holdRef is required');
      hold = await tx.readHold(tenantId, input.holdRef);
      // Another customer's hold is reported as missing, not as forbidden.
      if (!hold || hold.customerId !== customerId) {
        throw new WalletError('HOLD_NOT_FOUND', 'Hold not found', { holdRef: input.holdRef });
      }
      currency = hold.currency;
    } else {
      // Any valid code. Whether NEW money may enter a currency is decided when the
      // Checkout session is created (createTopUpCheckout): by the time Stripe confirms
      // payment the money is the customer's, even if the currency was disabled meanwhile.
      // Spending and refunding in a disabled currency must keep working too.
      const code = normalizeCurrency(input.currency);
      if (!code) throw new WalletError('INVALID_CURRENCY', 'Unknown currency', { currency: input.currency });
      currency = code;
    }

    // 2. Lock the balance, THEN look for the ref (see WalletTx contract).
    const balance = await tx.readBalance(tenantId, customerId, currency);
    const existing = await tx.readEntry(tenantId, ref);
    if (existing) {
      assertSamePayload(existing, input, currency);
      return { entry: existing, replayed: true };
    }

    if (hold && hold.status !== 'OPEN') {
      throw new WalletError('HOLD_NOT_OPEN', `Hold is already ${hold.status.toLowerCase()}`, {
        holdRef: hold.holdRef, status: hold.status,
      });
    }

    // 3. Pure arithmetic and guards.
    const resolved: ResolvedCommand =
      'amount' in command || 'delta' in command ? command : { kind: command.kind, amount: hold!.amount };
    const movement = applyCommand(balance, resolved);

    // 4. Writes, after every read. Movements of one balance get strictly increasing
    //    timestamps, so history sorts exactly even when two land in the same millisecond.
    const clock = now();
    const last = balance.lastMovementAt?.getTime() ?? 0;
    const at = clock.getTime() > last ? clock : new Date(last + 1);
    const entry: WalletEntry = {
      id: entryIdFor(tenantId, ref),
      tenantId,
      customerId,
      ref,
      kind: command.kind,
      currency,
      amount: movement.amount,
      availableDelta: movement.availableDelta,
      heldDelta: movement.heldDelta,
      availableAfter: movement.next.available,
      heldAfter: movement.next.held,
      holdRef: command.kind === 'HOLD' ? ref : (hold?.holdRef ?? null),
      orderRef: input.orderRef ?? hold?.orderRef ?? null,
      externalRef: input.externalRef ?? null,
      reason: input.reason,
      actor: input.actor,
      createdAt: at,
    };
    await tx.writeBalance(tenantId, customerId, currency, movement.next, at);
    await tx.createEntry(entry);
    if (command.kind === 'HOLD') {
      await tx.writeHold({
        tenantId, customerId, holdRef: ref, currency, amount: movement.amount, status: 'OPEN',
        orderRef: input.orderRef ?? null, externalRef: input.externalRef ?? null, createdAt: at, settledAt: null,
      });
    } else if (hold) {
      await tx.writeHold({ ...hold, status: command.kind === 'CAPTURE' ? 'CAPTURED' : 'RELEASED', settledAt: at });
    }
    return { entry, replayed: false };
  }

  const post = (input: PostInput) => store.runTransaction((tx) => postInTx(tx, input));

  const moveFn = (kind: 'TOP_UP' | 'SPEND' | 'REFUND' | 'HOLD') => (input: MoveInput) =>
    post({ ...input, command: { kind, amount: input.amount } });

  const settleFn = (kind: 'CAPTURE' | 'RELEASE') => (input: SettleInput) =>
    post({
      ...input,
      ref: kind === 'CAPTURE' ? captureRefFor(input.holdRef) : releaseRefFor(input.holdRef),
      command: { kind },
    });

  return {
    post,
    /** Run inside a transaction the caller already opened (e.g. with its order write). */
    postInTx,
    topUp: moveFn('TOP_UP'),
    spend: moveFn('SPEND'),
    refund: moveFn('REFUND'),
    /** Reserve `amount`; the entry's ref becomes the holdRef. */
    hold: moveFn('HOLD'),
    capture: settleFn('CAPTURE'),
    release: settleFn('RELEASE'),
    adjust: (input: Omit<MoveInput, 'amount'> & { delta: number }) =>
      post({ ...input, command: { kind: 'ADJUSTMENT', delta: input.delta } }),

    getEntry(tenantId: string, ref: string): Promise<WalletEntry | null> {
      return store.runTransaction((tx) => tx.readEntry(tenantId, ref));
    },

    async getWallet(tenantId: string, customerId: string): Promise<WalletSnapshot> {
      const snapshot = await store.getWallet(tenantId, customerId);
      const preferred = snapshot?.preferredCurrency;
      return {
        tenantId,
        customerId,
        // A preference for a currency that has since been disabled falls back to the default.
        preferredCurrency: preferred && policy.allowed.includes(preferred) ? preferred : policy.defaultCurrency,
        balances: snapshot?.balances ?? [],
      };
    },

    async setPreferredCurrency(tenantId: string, customerId: string, input: unknown): Promise<CurrencyCode> {
      const currency = requireAllowedCurrency(policy, input);
      await store.setPreferredCurrency(tenantId, customerId, currency);
      return currency;
    },

    listEntries(query: { tenantId: string; customerId: string; currency?: string; cursor?: string | null; limit?: number }): Promise<EntryPage> {
      const currency = query.currency ? normalizeCurrency(query.currency) : undefined;
      if (currency === null) throw new WalletError('INVALID_CURRENCY', 'Unknown currency', { currency: query.currency });
      const limit = Math.min(100, Math.max(1, Math.trunc(query.limit ?? 25) || 25));
      return store.listEntries({ ...query, currency, limit });
    },

    /** Empty array = the balance equals the sum of its entries in every currency. */
    async reconcile(tenantId: string, customerId: string) {
      const [snapshot, sums] = await Promise.all([
        store.getWallet(tenantId, customerId),
        store.sumEntries(tenantId, customerId),
      ]);
      const stored = new Map((snapshot?.balances ?? []).map((b) => [b.currency, b]));
      const summed = new Map(sums.map((s) => [s.currency, s]));
      const currencies = new Set([...stored.keys(), ...summed.keys()]);
      const mismatches: Array<{ currency: CurrencyCode; stored: CurrencyBalance; fromEntries: CurrencyBalance }> = [];
      for (const currency of currencies) {
        const a = stored.get(currency) ?? { currency, available: 0, held: 0 };
        const b = summed.get(currency) ?? { currency, available: 0, held: 0 };
        if (a.available !== b.available || a.held !== b.held) mismatches.push({ currency, stored: a, fromEntries: b });
      }
      return mismatches;
    },
  };
}

/** A reused ref must describe the same movement; anything else is a caller bug worth a 409. */
function assertSamePayload(existing: WalletEntry, input: PostInput, currency: CurrencyCode): void {
  const { command } = input;
  const sameAmount =
    'delta' in command ? existing.availableDelta === command.delta
    : 'amount' in command ? existing.amount === command.amount
    : existing.holdRef === input.holdRef;
  if (
    existing.customerId !== input.customerId ||
    existing.kind !== command.kind ||
    existing.currency !== currency ||
    !sameAmount
  ) {
    throw new WalletError('REF_CONFLICT', 'ref was already used for a different movement', {
      ref: existing.ref, existingKind: existing.kind, existingAmount: existing.amount,
    });
  }
}
```

### Behavior worth knowing

| Situation | Result |
|---|---|
| Same ref, same payload, any number of times, concurrently | one entry; every caller gets it; `replayed` is true for all but one |
| Same ref, different amount, kind, currency or customer | `REF_CONFLICT` (409) |
| `TOP_UP` in a currency no longer listed | credited: the Checkout was opened while it was listed, so the money is the customer's |
| `SPEND` / `REFUND` / `ADJUSTMENT` in a delisted currency | allowed: removing a currency never strands a balance |
| Capture after release, or release after capture | `HOLD_NOT_OPEN` (409) with the current status |
| Capture of another customer's hold | `HOLD_NOT_FOUND`, the same as a hold that does not exist |
| Two movements of one balance in the same millisecond | the second is stamped 1 ms later, so history sorts exactly |
| `listEntries` limit | clamped to 1 to 100, default 25; the cursor is opaque |

**Where the allow-list is enforced.** New money is refused at Checkout creation (`createTopUpCheckout` in
[stripe-top-up.md](stripe-top-up.md)), not in the engine. By the time Stripe confirms a payment, the money
belongs to the customer, even if the currency was delisted in between; refusing to credit it would take a
payment and give nothing for it.

**Where the tenant comes from.** Every method takes `tenantId` as an argument, and every route passes the
one from the session or the webhook endpoint. The engine cannot tell a forged tenant from a real one; the
routes must never read it from the request body.

## Wiring

One instance per server process, created lazily from the host's store and the policy in `config.ts`:
`getWallet()` in `lib/wallet/server.ts` ([api-routes.md](api-routes.md)). Tests create their own with the
in-memory store ([testing.md](testing.md)).

`getEntry(tenantId, ref)` answers "has this happened yet?" without a write. The top-up status route and the
bookable-events adapter use it.
