# Operations

Running the wallet: configuration, scheduled work, what to watch, and the procedures support needs.

## Configuration

| Setting | Where | Notes |
|---|---|---|
| Currencies and default | `WALLET_CURRENCIES` in `lib/wallet/config.ts` | first code is the default; see [money.md](money.md) |
| Top-up limits | overrides in `lib/wallet/config.ts` | review for every currency worth much less than a dollar |
| Stripe key per tenant | `getStripe(tenantId)` in `host.ts` | the account that receives that tenant's money |
| Webhook secret per tenant | `getWebhookSecret(tenantId)` | one endpoint per Stripe account |
| `CRON_SECRET` | env | the sweeper refuses every request while it is unset |
| Feature gate | `isWalletEnabled(tenantId)` | checked by every customer route, not only by the UI |

## Scheduled work

| Job | Schedule | What it does |
|---|---|---|
| `GET /api/cron/wallet-holds` | every 15 minutes | settles holds older than 90 minutes whose webhook never arrived ([split-payment.md](split-payment.md)) |
| Reconciliation | nightly, optional | calls `wallet.reconcile` for wallets that moved that day and reports mismatches |

A nightly reconciliation is a host job: list the customers with entries since yesterday (a query on
`createdAt`), call `reconcile(tenantId, customerId)` for each, and alert on any non-empty result. The staff
panel runs the same check on every view.

## What to watch

| Signal | Meaning | Action |
|---|---|---|
| Webhook 500s in the Stripe Dashboard | credits or captures failing; Stripe keeps retrying for up to three days | read the `[wallet webhook]` log line with the event id; fix, then let Stripe retry or resend |
| `tenant_mismatch` outcomes | an endpoint receives another tenant's sessions | check endpoint URLs and secrets per Stripe account |
| `wallet_short` outcomes | a card paid after its hold was released and the balance no longer covered the wallet part | the order was flagged by `onSplitPaid`; refund the card part or collect the difference |
| Sweeper `error` outcomes | Stripe unreachable, or a session id from another account | the next run retries; persistent errors mean `stripeFor` returns the wrong account |
| Holds open longer than a day | an async payment still settling, or a sweeper not running | check the cron's last run; an unpaid `complete` session is expected for bank debits |
| Reconciliation mismatch | something wrote a balance without an entry | find the writer; correct with an `ADJUSTMENT`, never by editing the balance |

## Procedures

**Correct a balance.** An admin posts an adjustment from the staff panel with a note. Never edit a balance
row or document by hand: the reconciliation check flags it, and history no longer explains the balance.

**Refund an order.** Call `refundOrder` with the amounts the order recorded when it was paid. For a card
payment already refunded in the Stripe Dashboard, refund only the wallet part here (`cardPaid: 0`), or the
Stripe call answers `charge_already_refunded`, which the template treats as done.

**Delist a currency.** Remove it from `WALLET_CURRENCIES`. Existing balances stay visible and spendable,
marked as retired; new top-ups in it are refused at Checkout; a Checkout already open still credits. To empty
the balances, refund or adjust them individually; do not convert them.

**Add a currency.** Add it to `WALLET_CURRENCIES`, review its top-up limits, enable payment methods for it in
the Stripe Dashboard, and add its formatting check to the host's UI tests.

**Rotate a webhook secret.** When an endpoint secret is rolled, Stripe can keep the old one active for up to
24 hours and signs each event with both. Deploy the new secret within that window.

## Seeds, demos and imports

Every balance a seed or an import creates is an entry, posted through the wallet:

- **Seeds and demo data** call `wallet.adjust` with ref `seed:<customerId>:<currency>` and a reason like
  `wallet.reason.seed`. Re-running the seed replays instead of doubling the balance.
- **Importing balances from another system** posts one `ADJUSTMENT` per customer per currency with ref
  `import:<oldSystemId>:<currency>`, reason `wallet.reason.openingBalance`, actor `{ type: 'system', id:
  'import' }`. The history then starts with an explained opening balance.
- **Demo money stays in demo tenants.** Wallets are per tenant, so credit granted in a demo tenant cannot pay
  for anything in a live one. Keep demo sign-ups on their own tenant id.

## Go-live checklist

- [ ] `WALLET_CURRENCIES` resolved and reviewed; limits checked per currency
- [ ] Store migrated: Firestore indexes deployed, or `db/wallet.sql` applied
- [ ] `host.ts` implemented; every route answers something other than the "implement" error
- [ ] Webhook endpoint per Stripe account with `WALLET_WEBHOOK_EVENTS`; a test top-up credited once after
      a resend
- [ ] Cron scheduled with `CRON_SECRET`; a manual call without the header answers 401
- [ ] Split payment tested: pay (captured), abandon until expiry (released), async method (held until settled)
- [ ] Refund tested both ways: wallet part back in the wallet, card part in the Stripe Dashboard
- [ ] Staff adjustment tested, including a double click on confirm
- [ ] Strings present in every locale the host ships
- [ ] `vitest run` passes, with the store conformance suite pointed at a staging database
