# Firestore store

`WalletStore` on Firestore through `firebase-admin`. Server only: clients never read or write wallet
collections directly, they go through the API routes.

| Collection | Document id | Contents |
|---|---|---|
| `wallets` | `<tenantId>:<customerId>` | `preferredCurrency`, `balances.<CUR>.{available, held, lastMovementAt}` |
| `walletEntries` | `entryIdFor(tenantId, ref)`, a sha256 prefix | one immutable `WalletEntry` |
| `walletHolds` | `entryIdFor(tenantId, holdRef)` | `WalletHold`, status OPEN, then CAPTURED or RELEASED |

The entry's document id **is** its idempotency key: two writers with the same ref target the same document,
Firestore's optimistic transactions make one of them retry, and the retry sees the entry and replays it.
Hashing makes any ref usable as an id (refs may contain `/`, which ids cannot).

```ts
// lib/wallet/firestore-store.ts: WalletStore on Firestore (firebase-admin, server only).
//
//   wallets/{tenantId}:{customerId}       preferredCurrency, balances.{CUR}.{available,held}
//   walletEntries/{entryIdFor(t, ref)}    one immutable doc per movement; the id IS the idempotency key
//   walletHolds/{entryIdFor(t, holdRef)}  OPEN, then CAPTURED or RELEASED
import {
  AggregateField, Timestamp, type DocumentData, type DocumentSnapshot, type Firestore, type Transaction,
} from 'firebase-admin/firestore';
import type { CurrencyCode } from './money';
import type { EntryQuery, WalletStore, WalletTx } from './ports';
import type { BalanceAmounts, BalanceState, WalletEntry, WalletHold, WalletSnapshot } from './types';
import { entryIdFor } from './wallet';

const WALLETS = 'wallets';
const ENTRIES = 'walletEntries';
const HOLDS = 'walletHolds';

function walletId(tenantId: string, customerId: string): string {
  if (tenantId.includes('/') || customerId.includes('/')) throw new Error('ids must not contain "/"');
  return `${tenantId}:${customerId}`;
}

const toDate = (v: unknown): Date => (v instanceof Timestamp ? v.toDate() : new Date(v as string));

function entryFromDoc(data: DocumentData): WalletEntry {
  return { ...(data as WalletEntry), createdAt: toDate(data.createdAt) };
}

function holdFromDoc(data: DocumentData): WalletHold {
  return {
    ...(data as WalletHold),
    createdAt: toDate(data.createdAt),
    settledAt: data.settledAt ? toDate(data.settledAt) : null,
  };
}

function balancesFrom(data: DocumentData | undefined) {
  const map = (data?.balances ?? {}) as Record<string, BalanceAmounts>;
  return Object.entries(map).map(([currency, b]) => ({ currency, available: b.available ?? 0, held: b.held ?? 0 }));
}

/**
 * Wraps a Firestore transaction the caller opened, so a wallet movement can commit
 * atomically with the caller's own writes (the order that the payment is for).
 * Firestore rule: every read before every write. Do the caller's reads, then
 * `wallet.postInTx(...)`, then the caller's writes.
 */
export function firestoreWalletTx(db: Firestore, t: Transaction, defaultCurrency: CurrencyCode): WalletTx {
  const walletDocs = new Map<string, DocumentSnapshot>();
  return {
    async readBalance(tenantId, customerId, currency) {
      const id = walletId(tenantId, customerId);
      const snap = walletDocs.get(id) ?? (await t.get(db.collection(WALLETS).doc(id)));
      walletDocs.set(id, snap);
      const b = snap.get(`balances.${currency}`) as (BalanceAmounts & { lastMovementAt?: Timestamp }) | undefined;
      const state: BalanceState = {
        available: b?.available ?? 0,
        held: b?.held ?? 0,
        lastMovementAt: b?.lastMovementAt ? b.lastMovementAt.toDate() : null,
      };
      return state;
    },
    async readEntry(tenantId, ref) {
      const snap = await t.get(db.collection(ENTRIES).doc(entryIdFor(tenantId, ref)));
      return snap.exists ? entryFromDoc(snap.data()!) : null;
    },
    async readHold(tenantId, holdRef) {
      const snap = await t.get(db.collection(HOLDS).doc(entryIdFor(tenantId, holdRef)));
      return snap.exists ? holdFromDoc(snap.data()!) : null;
    },
    async writeBalance(tenantId, customerId, currency, next, at) {
      const id = walletId(tenantId, customerId);
      const ref = db.collection(WALLETS).doc(id);
      const now = Timestamp.fromDate(at);
      const value = { available: next.available, held: next.held, lastMovementAt: now };
      if (walletDocs.get(id)?.exists) {
        // Dotted path: touches only this currency. Codes are [A-Z]{3}, so the path is safe.
        t.update(ref, { [`balances.${currency}`]: value, updatedAt: now });
      } else {
        t.set(ref, {
          tenantId, customerId, preferredCurrency: defaultCurrency,
          balances: { [currency]: value }, createdAt: now, updatedAt: now,
        });
      }
    },
    async createEntry(entry) {
      // create(), not set(): fails if the doc exists, so a lost race can never overwrite.
      t.create(db.collection(ENTRIES).doc(entry.id), { ...entry, createdAt: Timestamp.fromDate(entry.createdAt) });
    },
    async writeHold(hold) {
      t.set(db.collection(HOLDS).doc(entryIdFor(hold.tenantId, hold.holdRef)), {
        ...hold,
        createdAt: Timestamp.fromDate(hold.createdAt),
        settledAt: hold.settledAt ? Timestamp.fromDate(hold.settledAt) : null,
      });
    },
  };
}

export function createFirestoreWalletStore(db: Firestore, defaultCurrency: CurrencyCode): WalletStore {
  return {
    runTransaction(fn) {
      // Firestore retries the callback on contention, which is why apply.ts is pure
      // and every write goes through the transaction object.
      return db.runTransaction((t) => fn(firestoreWalletTx(db, t, defaultCurrency)));
    },

    async getWallet(tenantId, customerId): Promise<WalletSnapshot | null> {
      const snap = await db.collection(WALLETS).doc(walletId(tenantId, customerId)).get();
      if (!snap.exists) return null;
      return {
        tenantId,
        customerId,
        preferredCurrency: (snap.get('preferredCurrency') as string | undefined) ?? defaultCurrency,
        balances: balancesFrom(snap.data()),
      };
    },

    async setPreferredCurrency(tenantId, customerId, currency) {
      await db.collection(WALLETS).doc(walletId(tenantId, customerId)).set(
        { tenantId, customerId, preferredCurrency: currency, updatedAt: Timestamp.now() },
        { merge: true },
      );
    },

    async listEntries(q: EntryQuery) {
      let query = db.collection(ENTRIES)
        .where('tenantId', '==', q.tenantId)
        .where('customerId', '==', q.customerId);
      if (q.currency) query = query.where('currency', '==', q.currency);
      query = query.orderBy('createdAt', 'desc').orderBy('id', 'desc');
      if (q.cursor) {
        const cursorSnap = await db.collection(ENTRIES).doc(q.cursor).get();
        // A cursor from another customer is treated as no cursor, not as a leak.
        if (cursorSnap.exists && cursorSnap.get('customerId') === q.customerId) query = query.startAfter(cursorSnap);
      }
      // Fetch one extra to know whether there is a next page without a second query.
      const snap = await query.limit(q.limit + 1).get();
      const docs = snap.docs.slice(0, q.limit);
      return {
        entries: docs.map((d) => entryFromDoc(d.data())),
        nextCursor: snap.docs.length > q.limit ? (docs[docs.length - 1]?.id ?? null) : null,
      };
    },

    async listOpenHolds(olderThan, limit) {
      const snap = await db.collection(HOLDS)
        .where('status', '==', 'OPEN')
        .where('createdAt', '<', Timestamp.fromDate(olderThan))
        .orderBy('createdAt')
        .limit(limit)
        .get();
      return snap.docs.map((d) => holdFromDoc(d.data()));
    },

    async sumEntries(tenantId, customerId) {
      const wallet = await db.collection(WALLETS).doc(walletId(tenantId, customerId)).get();
      const currencies = balancesFrom(wallet.data()).map((b) => b.currency);
      return Promise.all(currencies.map(async (currency) => {
        const agg = await db.collection(ENTRIES)
          .where('tenantId', '==', tenantId)
          .where('customerId', '==', customerId)
          .where('currency', '==', currency)
          .aggregate({ available: AggregateField.sum('availableDelta'), held: AggregateField.sum('heldDelta') })
          .get();
        const data = agg.data();
        return { currency, available: data.available ?? 0, held: data.held ?? 0 };
      }));
    },
  };
}
```

## Sharing the caller's transaction

`firestoreWalletTx(db, t, DEFAULT_CURRENCY)` wraps a transaction the caller opened, so a wallet movement
commits together with the order it pays for. Firestore's rule applies across both: **the caller's reads, then
`wallet.postInTx(...)`, then the caller's writes**. A read after the wallet's writes throws. Worked example:
[integration.md](integration.md).

One wallet movement per Firestore transaction. A second `postInTx` in the same transaction would read after
the first one's writes.

## Indexes

History queries filter on tenant, customer and optionally currency, and sort by `createdAt` then `id`. The
sweeper queries open holds by age.

```json
{
  "indexes": [
    {
      "collectionGroup": "walletEntries",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "tenantId", "order": "ASCENDING" },
        { "fieldPath": "customerId", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" },
        { "fieldPath": "id", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "walletEntries",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "tenantId", "order": "ASCENDING" },
        { "fieldPath": "customerId", "order": "ASCENDING" },
        { "fieldPath": "currency", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" },
        { "fieldPath": "id", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "walletHolds",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "status", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "ASCENDING" }
      ]
    }
  ]
}
```

Merge these into the host's `firestore.indexes.json` and deploy them before the first history request; a
missing index fails the query with a link to create it. The reconciliation sums use equality filters only
and need no composite index.

## Security rules

```
match /wallets/{id}        { allow read, write: if false; }
match /walletEntries/{id}  { allow read, write: if false; }
match /walletHolds/{id}    { allow read, write: if false; }
```

The Admin SDK bypasses rules, so the server is unaffected. Customer reads go through `GET /api/wallet` and
`GET /api/wallet/entries`, which scope by the session. Opening read access to the owner is possible but then
the rule must match the host's auth uid to `customerId` **and** the tenant, and history pagination moves to
the client; the API route is simpler and keeps one code path.

## Notes

- **`AggregateField.sum`** needs a `firebase-admin` with sum aggregations; the templates were checked with
  version 13. Reconciliation runs one aggregation per currency the wallet holds.
- **Timestamps.** Entries store `createdAt` as a Firestore `Timestamp` (millisecond precision in practice);
  `lastMovementAt` on the balance stores the same instant, which is what keeps same-millisecond entries in
  order.
- **Emulator.** `test/store-conformance.test.ts` runs the full contract, concurrency cases included, against
  the Firestore emulator when `FIRESTORE_EMULATOR_HOST` is set ([testing-stores.md](testing-stores.md)).
