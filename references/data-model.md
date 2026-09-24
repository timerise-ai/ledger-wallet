# Data model

The wallet is three things per tenant: a **balance per customer per currency**, an **append-only ledger of
entries** that explains every change to it, and a **hold table** for money reserved while a card pays the
rest of an order. All amounts are integers in the currency's own minor unit (see [money.md](money.md)).

## Rename table

The canonical vocabulary below is what the templates use. Rename once, as a decision, before generating any
file; then grep for the canonical names to confirm nothing was missed.

| Canonical | Meaning | Typical host names |
|---|---|---|
| wallet | a customer's stored credit, all currencies | account balance, credit, funds, purse |
| tenant | the business that holds the money | organization, workspace, merchant, site, location group |
| customer | whoever owns the wallet | player, member, user, client, guest account |
| entry | one immutable movement in the ledger | transaction, movement, ledger line |
| top-up | money paid in by card | add funds, load, deposit, recharge |
| spend | money paid out of the wallet for something | charge, debit, payment, deduction |
| refund | money returned to the wallet | credit back, return |
| adjustment | a staff correction with a reason | manual credit, write-off, goodwill |
| hold | money reserved while a card pays the rest | reservation, authorization, pending debit |
| order | whatever the wallet pays for | booking, reservation, ticket, invoice, appointment |

Tenant scope is **one field, `tenantId`, always derived on the server** from the session or the webhook
endpoint, never read from a request body. A customer who uses two tenants has two wallets. That is the
point: money paid into one tenant's Stripe account is only spendable at that tenant.

## Entry kinds

| Kind | available | held | Who posts it | ref convention |
|---|---|---|---|---|
| `TOP_UP` | + amount | | Stripe webhook | `stripe:cs:<checkout session id>` |
| `SPEND` | - amount | | order service | `order:<orderId>` |
| `REFUND` | + amount | | refund service | `order:<orderId>:refund` |
| `ADJUSTMENT` | + or - delta | | staff, with a note | `adjust:<requestId uuid>` |
| `HOLD` | - amount | + amount | split payment | `order:<orderId>:hold` (the hold's own id) |
| `CAPTURE` | | - amount | webhook or sweeper | `<holdRef>:capture` |
| `RELEASE` | + amount | - amount | webhook or sweeper | `<holdRef>:release` |

A late capture after a released hold is a plain `SPEND` with ref `<holdRef>:late` (see
[split-payment.md](split-payment.md)). Event tickets through the bookable-events adapter use the refs that
skill generates (see [integration.md](integration.md)).

**The `ref` is the idempotency key.** It is unique per tenant, and every command derives it from something
that already identifies the operation: the Checkout session, the order, the request id generated when a
staff dialog opened. Posting the same ref again returns the original entry with `replayed: true` and changes
nothing. Posting the same ref with a different kind, currency, amount or customer is a `REF_CONFLICT` (409),
because it is always a caller bug.

## Invariants

| Invariant | Enforced by |
|---|---|
| `available >= 0` and `held >= 0` in every currency | `applyCommand`; Postgres CHECK constraints as a second line |
| balance = sum of its entries' deltas, per currency | every write goes through `postInTx`; `reconcile()` verifies |
| an entry is never updated or deleted | Firestore `create()` + no client rules; Postgres trigger |
| a hold settles once, as either CAPTURE or RELEASE | hold status read under the balance lock or row lock |
| currencies are never converted into each other | there is no conversion code; each currency is its own balance |
| entries of one balance have strictly increasing `createdAt` | the engine bumps a same-millisecond write by 1 ms |

## Types

```ts
// lib/wallet/types.ts: the wallet's data model. All amounts are stored minor units.
import type { CurrencyCode } from './money';

export type EntryKind =
  | 'TOP_UP'      // customer paid money in (Stripe)            available +
  | 'SPEND'       // paid for something from the wallet         available -
  | 'REFUND'      // money returned to the wallet               available +
  | 'ADJUSTMENT'  // operator correction, signed, reason required available +/-
  | 'HOLD'        // reserved while a card pays the rest        available -, held +
  | 'CAPTURE'     // the reserved part is now spent             held -
  | 'RELEASE';    // the reservation is returned                held -, available +

export type Actor =
  | { type: 'customer'; id: string }
  | { type: 'staff'; id: string }
  | { type: 'system'; id: string }; // 'stripe-webhook', 'hold-sweeper', 'order-service', ...

/** Translatable description: a key in the host's i18n plus its parameters. Never a literal. */
export interface Reason {
  key: string;
  params?: Record<string, string>;
}

export interface BalanceAmounts {
  available: number;
  held: number;
}

/** What a transaction reads: the amounts plus when this balance last moved. */
export interface BalanceState extends BalanceAmounts {
  lastMovementAt: Date | null;
}

export interface CurrencyBalance extends BalanceAmounts {
  currency: CurrencyCode;
}

export interface WalletSnapshot {
  tenantId: string;
  customerId: string;
  preferredCurrency: CurrencyCode;
  /** Only currencies the customer has ever held. The API zero-fills allowed ones. */
  balances: CurrencyBalance[];
}

export interface WalletEntry {
  id: string;                  // derived from (tenantId, ref), see entryIdFor
  tenantId: string;
  customerId: string;
  ref: string;                 // idempotency key, unique per tenant
  kind: EntryKind;
  currency: CurrencyCode;
  amount: number;              // positive magnitude
  availableDelta: number;      // signed change to available
  heldDelta: number;           // signed change to held
  availableAfter: number;
  heldAfter: number;
  holdRef: string | null;      // HOLD / CAPTURE / RELEASE: the hold this belongs to
  orderRef: string | null;     // the host's order / booking / invoice id
  externalRef: string | null;  // Stripe Checkout session, payment intent, ...
  reason: Reason;
  actor: Actor;
  createdAt: Date;
}

export type HoldStatus = 'OPEN' | 'CAPTURED' | 'RELEASED';

/** A reservation's current state. Entries are the history; this is the row the sweeper scans. */
export interface WalletHold {
  tenantId: string;
  customerId: string;
  holdRef: string;
  currency: CurrencyCode;
  amount: number;
  status: HoldStatus;
  orderRef: string | null;
  externalRef: string | null;  // the Checkout session paying the remainder
  createdAt: Date;
  settledAt: Date | null;
}

/** What the caller asks for. `ref` makes every command safe to retry. */
export type WalletCommand =
  | { kind: 'TOP_UP' | 'SPEND' | 'REFUND' | 'HOLD'; amount: number }
  | { kind: 'ADJUSTMENT'; delta: number }
  | { kind: 'CAPTURE' | 'RELEASE' };

export interface PostInput {
  tenantId: string;
  customerId: string;
  /** Required for every kind except CAPTURE / RELEASE, which take the hold's currency. */
  currency?: CurrencyCode;
  ref: string;
  command: WalletCommand;
  reason: Reason;
  actor: Actor;
  orderRef?: string | null;
  externalRef?: string | null;
  /** CAPTURE / RELEASE: which hold. */
  holdRef?: string;
}

export interface PostResult {
  entry: WalletEntry;
  /** true when `ref` had already been applied, nothing changed this time. */
  replayed: boolean;
}

export interface EntryPage {
  entries: WalletEntry[];
  nextCursor: string | null;
}
```

`availableAfter` and `heldAfter` make every history row self-explaining: support can read a customer's
history top to bottom without recomputing anything. `holdRef` links the three entries of one hold.
`externalRef` is the Stripe object that caused the entry; it is what an operator pastes into the Stripe
Dashboard search.

## Errors

```ts
// lib/wallet/errors.ts: one error type, one code per failure a caller can act on.
export type WalletErrorCode =
  | 'INVALID_AMOUNT'        // not a positive safe integer in minor units
  | 'INVALID_CURRENCY'      // not an ISO-4217 code
  | 'CURRENCY_NOT_ALLOWED'  // valid code, not in WALLET_CURRENCIES
  | 'AMOUNT_OUT_OF_RANGE'   // outside the top-up limits for that currency
  | 'NOT_CHARGEABLE'        // Stripe would reject the amount (minimum, three-decimal rule)
  | 'INSUFFICIENT_FUNDS'    // available < amount
  | 'REF_CONFLICT'          // ref reused with a different payload
  | 'HOLD_NOT_FOUND'
  | 'HOLD_NOT_OPEN'         // capture after release, or release after capture
  | 'WALLET_DISABLED';      // feature gate off for this tenant

export class WalletError extends Error {
  constructor(
    public readonly code: WalletErrorCode,
    message: string,
    public readonly details: Record<string, unknown> = {},
  ) {
    super(message);
    this.name = 'WalletError';
  }
}

export const WALLET_ERROR_STATUS: Record<WalletErrorCode, number> = {
  INVALID_AMOUNT: 400,
  INVALID_CURRENCY: 400,
  CURRENCY_NOT_ALLOWED: 400,
  AMOUNT_OUT_OF_RANGE: 400,
  NOT_CHARGEABLE: 400,
  INSUFFICIENT_FUNDS: 402,
  REF_CONFLICT: 409,
  HOLD_NOT_FOUND: 404,
  HOLD_NOT_OPEN: 409,
  WALLET_DISABLED: 403,
};

export function isWalletError(err: unknown): err is WalletError {
  return err instanceof WalletError;
}
```

`INSUFFICIENT_FUNDS` is a 402 and carries `{ available, required }` in `details`, so the UI can say "you
have $4.00, this costs $10.00" instead of a generic failure. Every route returns the same shape:
`{ error: { code, message, details } }` (see [api-routes.md](api-routes.md)).

## What the wallet does not store

- **No order state.** The host's order records what it was paid with (`walletPaid`, `cardPaid`,
  `paymentIntentId`); the wallet only knows `orderRef`. Refunds read the order's recorded amounts, never its
  current total.
- **No exchange rates.** Multi-currency means several independent balances. A customer holding EUR cannot
  pay a USD order from it; the UI shows each balance separately.
- **No display strings.** `reason` is a key plus parameters, translated at render time in the viewer's
  language.
