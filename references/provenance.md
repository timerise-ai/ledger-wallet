# Provenance

The wallet described here was audited against the earlier implementation: a stored-credit balance in a
multi-location booking system, with card top-ups through Stripe Checkout, full and partial payment of orders
from the balance, refunds to the balance, and several deployments sharing one database. The audit read every
path that credited, debited or displayed a balance, and every screen that showed one.

This file is the engineering ledger for anyone editing the skill. It separates three things: what the audit
changed and how the templates verify it, what was kept deliberately with the reason it is safe, and what was
designed in the skill and has never run in production.

## Fixed in the templates

### 1. Cancelling an order refunded its current total, whatever was paid

Cancelling a confirmed order credited the order's stored total to the balance, regardless of how, or whether,
it had been paid: an order still awaiting payment at the counter became free credit, and the card-paid part
of a mixed payment came back as store credit. The stored total could also be raised after payment through a
second edit route whose key check passed when the header was absent.
**Shipped:** `refundOrder` takes the amounts the order recorded when it was paid, per source, and sends each
part back to where it came from ([split-payment.md](split-payment.md), [integration.md](integration.md)).
Tests: the four refund tests in `test/stripe.test.ts`.

### 2. The cancel refund was not atomic and not idempotent

The refund read the balance, added to it outside any transaction, and wrote the balance, the entry and the
order status as three independent writes. A top-up landing in between was overwritten; the entry and the
balance could disagree; two concurrent cancels both refunded.
**Shipped:** every movement goes through `postInTx` inside one transaction, and the refund's ref is
`order:<id>:refund` ([engine.md](engine.md)). Tests: `concurrent duplicates credit once`, the conformance
suite on both real backends.

### 3. The top-up webhook credited on every delivery

The webhook created a new credit for each `checkout.session.completed` it received, with no check for an
earlier delivery and no check of `payment_status`; asynchronous payment success was not handled.
**Shipped:** the credit's ref is the Checkout session id; credit only on `paid` or
`async_payment_succeeded` ([stripe-top-up.md](stripe-top-up.md)). Tests: `credits once however many times
Stripe delivers`, `does not credit a completed session whose async payment has not settled`.

### 4. The balance was debited before the order existed

A full or partial balance payment was debited, and only then was the order created. When creation failed on
a capacity conflict, the request answered 409 and the money was gone with no order to recover it from.
**Shipped:** a wallet-only payment is posted inside the order's own transaction; a mixed payment holds
instead of debiting ([integration.md](integration.md), [split-payment.md](split-payment.md)). Tests: the
order examples in `test/postgres.test.ts` and `test/firestore.test.ts`.

### 5. Paying a pending order from the balance could charge twice

The order's pending status was checked outside the transaction that debited, so two concurrent requests both
debited. The card session of the same order was not reliably expired, so the card and the balance could both
pay it. The failure rollback returned only the latest debit, not the part paid from the balance earlier.
**Shipped:** hold, capture and release keyed by the order, with the Checkout session created first and linked
to the hold; capture and release are mutually exclusive. Tests: the `holds` suite, `a racing capture and
release settle a hold exactly once` on every backend.

### 6. A balance payment fell through to a card payment

Choosing to pay from the balance with too little in it silently created a card Checkout instead, including
when card payments were switched off.
**Shipped:** `planPayment` with `'wallet'` throws `INSUFFICIENT_FUNDS` with both numbers; paying the rest by
card is a separate, explicit choice. Test: `never switches a wallet payment to a card silently`.

### 7. A refund could be recorded without the money

Cancelling a paid registration marked it refunded in one transaction and credited the balance in another; a
failed credit was logged and dropped, and credits of every kind were recorded as top-ups.
**Shipped:** a refund is one idempotent post that is safe to retry until it succeeds, with its own entry kind
([engine.md](engine.md)). The bookable-events adapter keeps that skill's refs
([integration.md](integration.md)).

### 8. One balance was shared by every deployment

Balances lived on the customer record, which all deployments sharing the database used. Money paid into one
deployment's Stripe account was spendable at another, and credit granted to demo sign-ups was spendable
everywhere.
**Shipped:** one wallet per tenant and customer, the tenant always from the session or the webhook endpoint,
and a webhook tenant check ([data-model.md](data-model.md), [stripe-top-up.md](stripe-top-up.md)). Tests:
`refs are scoped per tenant`, `ignores another tenant's session`.

### 9. The ledger could not explain the balance

Profile creation, seeds and demo sign-ups wrote balances with no entry; entries had no actor; the entry type
did not distinguish a refund from a top-up. The sum of entries did not equal the balance, and nothing
checked.
**Shipped:** every change is an entry with a kind, an actor and a reason; seeds and imports post
adjustments; `reconcile` compares, and the staff panel shows the result ([operations.md](operations.md)).
Test: `reconcile is clean after normal use and catches a write that bypassed the ledger`.

### 10. Currency handling

Defaults and staff APIs hardcoded a pair of currencies; the preferred currency defaulted to one of them. A
zero-decimal currency was stored with two extra digits and rounded at the Stripe boundary, so a top-up of a
fractional amount charged and credited a different one. Two formatters produced different output. Top-up
amounts were not checked as integers or bounded. Checkout hardcoded a payment-method list that included a
method limited to one currency.
**Shipped:** any ISO 4217 code in its real minor unit, a currency list resolved from the skill argument, one
formatter, per-currency limits floored at Stripe's minimum, Stripe choosing payment methods
([money.md](money.md)). Tests: `test/money.test.ts`.

### 11. The currency choice did not stick

The server never stored the customer's choice and always answered with the default; the customer screen
applied that answer on every refresh, from a callback with stale dependencies, so the switcher jumped back.
**Shipped:** `PUT /api/wallet/preferred-currency`, and a panel that reads the preference once
([ui.md](ui.md)). Test: `the preferred currency defaults to the policy default and must be allowed`.

### 12. Customer screens hid failures

A failed top-up request showed nothing; a failed balance request showed zero; the return from Stripe was
ignored; history showed one currency at a time without paging; some alerts were in one language only.
**Shipped:** error states with codes and numbers, the return poll, paged history for every currency, keys for
every string ([ui.md](ui.md)).

### 13. Operators could see balances and nothing else

There was no ledger view, no adjustment with a reason, no record of who changed a balance, and customer
lists showed only the default currency, so a customer holding another currency appeared to hold nothing.
Customer detail routes by id had no tenant scope, and the wallet's feature flag was enforced by a banner, not
by the API.
**Shipped:** the staff routes and panel, `customerExists` on every `[customerId]` route, `assertWalletEnabled`
on every customer route ([api-routes.md](api-routes.md), [admin-ui.md](admin-ui.md)). Tests:
`test/routes.test.ts`.

### 14. Scheduled cleanups were open when their secret was unset

The cleanup routes checked the secret only if one was configured.
**Shipped:** the sweeper answers 401 while `CRON_SECRET` is unset. Test: `fails closed when CRON_SECRET is
unset`.

### 15. An unused client-side writer was still exported

A second copy of the balance functions for the browser wrote its entry outside its transaction, so a retried
transaction repeated it. Security rules blocked it and nothing called it, but it was exported next to the
server version.
**Shipped:** no client-side writer exists; clients read through the API.

### 16. A split retried after its Checkout expired held nothing

Found by the agent evals of 0.1.2, in the skill's own split payment (see *Added*). The hold ref was one per
order, and a hold settles once, so a second split for the same order replayed the released hold: the new
Checkout ran with nothing reserved, and a balance spent in the meantime ended in `wallet_short` after the card
paid. A double click opened a second Checkout against the same hold.
**Shipped:** `orderRefs(orderRef, attempt)` gives each attempt its own hold ref; `startSplitPayment` refuses a
settled attempt with `HOLD_NOT_OPEN` and `details.nextAttempt`, returns the open Checkout of an open one, and
expires its own session when a concurrent start won ([split-payment.md](split-payment.md)). Tests: `a retry
after an expired Checkout holds the wallet part again under the next attempt`, `a second start of an open
attempt returns its Checkout instead of opening another`.

### 17. Reconciliation reported a concurrent movement as drift

Found by the agent evals of 0.1.2, in the skill's own reconciliation view. `reconcile` read the balance and
the entry sums as two reads, so a movement committing between them showed a mismatch on the staff panel.
**Shipped:** a mismatch is reported only when two reads in a row agree on it, or after a third
([engine.md](engine.md)). Test: `reconcile does not report a movement that commits between its two reads`.

### 18. A forged history cursor was a 500 on Postgres

Found by the agent evals of 0.1.2, in the skill's own Postgres store. A cursor that decoded to a non-date
reached Postgres as an invalid timestamp and failed the query; Firestore and the memory store already read an
unknown cursor as none. **Shipped:** the Postgres store does the same ([postgres.md](postgres.md)), checked
against a real Postgres. Test: `pages history newest first without gaps or repeats`, on every backend.

### 19. A redelivered failure skipped a split attempt

Found by the agent evals of 0.1.3, in the example split callbacks. 0.1.3 told the host to add 1 to
`splitAttempt` in `onSplitFailed`, but a redelivered `expired` event calls it again, and a late redelivery
could move the order past an attempt the customer had already started, opening a second Checkout and hold.
**Shipped:** the settlement carries `attempt` from the session's metadata, and the example sets
`splitAttempt` to `max(splitAttempt, attempt + 1)`, which a redelivery repeats without effect
([split-payment.md](split-payment.md)). Tests: `releases the wallet part when Checkout expires` and `a retry
after an expired Checkout holds the wallet part again under the next attempt` check the attempt carried.

### 20. The top-up form stayed busy after an error

Found by the agent evals of 0.1.3, in the customer panel. When the top-up route answered an error, the form
showed it but left its button disabled until a reload. **Shipped:** the error branch clears the busy state
([ui.md](ui.md)). No test: the suites have no DOM renderer; checked by reading the handler's three exits.

### 21. A redelivered paid event after a shortfall took the wallet part

Found by the agent evals of 0.1.3, in the late-payment path. After `wallet_short`, a redelivery of the paid
event retries the late `SPEND`, and succeeds if the customer topped up in between, for an order the host has
already flagged. **Shipped:** the rule that the first outcome of a session is final: `confirm` ignores a
flagged order, and the refund of a flagged order includes the entry `<holdRef>:late` if one exists
([split-payment.md](split-payment.md)). The engine is unchanged; recording the shortfall itself would need an
entry kind of its own.

## Found while verifying the templates

- **Same-millisecond entries sorted by hash.** Two movements in one millisecond appeared in either order, so
  `availableAfter` could read out of sequence. The engine now stamps each movement of a balance strictly later
  than the last. Test: `orders history exactly even when the clock does not move`.
- **A delisted currency refused a paid top-up.** An early engine checked the allow-list on credit, which would
  have taken a payment and credited nothing if the currency was delisted between Checkout and the webhook. The
  check moved to Checkout creation. Test: `keeps a disabled currency spendable...`.
- **Checkout expiry at exactly 30 minutes.** Stripe measures `expires_at` against its own clock and accepts 30
  minutes to 24 hours; the template uses 35 by default and clamps to 31 to 1439. Test: `sets a Checkout expiry
  inside the window Stripe accepts`.
- **The test config relied on `__dirname`.** Vite's native config loader, planned as its default, does not
  support it, and Vitest 5 warns on every run. The config resolves the alias from `import.meta.dirname`,
  available from Node.js 20.11 (checked against the Node.js ESM documentation), and is named
  `vitest.config.mts`: as `.ts` in a project whose `package.json` has no `"type": "module"`, Vitest 5 still
  warned about ESM syntax in a CommonJS file. Verified by a clean run under Vitest 5 with no warning.

## Kept deliberately

| Choice | Why it is safe |
|---|---|
| A stored balance next to the ledger, not a sum computed on read | the row is what the transaction locks; reads stay O(1); `reconcile` catches drift |
| One balance per currency, no conversion | a customer's EUR is never repriced; conversion is a product decision with its own rates and disclosures |
| Integer minor units | exact arithmetic; the exponent is now the currency's own |
| Store credit as a refund option (`destination: 'wallet'`) | the earlier implementation refunded everything to the balance; some businesses want that, with consent |
| Server-only writes, customer reads through the API | security rules or RLS deny all client access; one code path scopes by session |
| Descriptions as a key plus parameters | translated in the viewer's language at render time, never stored as a sentence |
| One transaction covering the order, the balance and the entry | the earlier implementation's refund of a failed pending order already did this, idempotently; it is now the rule for every movement |

## Added

Designed in the skill; the earlier implementation had no equivalent.

| Addition | Reference |
|---|---|
| Holds (`HOLD`, `CAPTURE`, `RELEASE`) for mixed payments, and the sweeper | [split-payment.md](split-payment.md) |
| Refunds of the card part through Stripe | [split-payment.md](split-payment.md) |
| Per-tenant wallets and the webhook tenant check | [data-model.md](data-model.md) |
| Staff adjustments with a required note, attribution, reconciliation view | [admin-ui.md](admin-ui.md) |
| Top-up status route and the return poll | [stripe-top-up.md](stripe-top-up.md) |
| Any ISO 4217 currency, the currency list as a skill argument | [money.md](money.md) |
| Postgres store; the store conformance suite on both backends | [postgres.md](postgres.md), [testing-stores.md](testing-stores.md) |
| The bookable-events `BalanceLedger` adapter | [integration.md](integration.md) |

## If you are porting the earlier implementation

Most damaging first:

1. Refund what an order recorded as paid, per source, never its current total; close the unauthenticated
   edit route (entry 1).
2. Make the top-up credit idempotent on the Checkout session, and credit only paid sessions (entry 3).
3. Debit inside the order's transaction, or hold, never before the order exists (entry 4).
4. Serialize paying a pending order and expire its card session first (entry 5).
5. Scope balances per deployment; move demo credit out of live deployments (entry 8).
6. Make every refund one idempotent, retryable post (entries 2 and 7).
7. Stop the silent fallback to card payment (entry 6).
8. Write entries for every balance change and start reconciling (entry 9).
9. Currency, preference and screen fixes (entries 10 to 12), then the operator surface (entry 13).
