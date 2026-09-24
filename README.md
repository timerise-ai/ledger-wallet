# ledger-wallet

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to build a **customer wallet with a balance
per currency** in a **Next.js App Router** app: stored credit topped up through Stripe Checkout, orders paid
from the wallet, by card, or split between the two, refunds that return each part to where it came from,
staff adjustments with a required reason, and a history that explains every balance. Firestore or Postgres;
any ISO currency, with the list passed as the skill argument and USD by default.

Storing a number is the easy half. The hard half is keeping it right while requests retry, webhooks arrive
twice or never, and a card and a balance pay for the same order. The skill keeps **every balance as the sum
of an append-only ledger, with every movement keyed by a ref derived from what caused it**, so any repeat
replays the first result; and it **holds a mixed payment's balance part until the card pays**, so no money
leaves the wallet for an order that never happens.

This skill was written by the engineer who has shipped this module. The earlier implementation it was
audited against was the stored-credit balance of a multi-location venue-booking system. The templates hold
the properties a wallet has to hold: a top-up is credited once, for what Stripe collected, however often the
event arrives; a wallet payment commits with its order or not at all; a balance never goes below zero under
concurrent spends; a hold settles exactly once; a refund returns what was paid, per source; a balance in one
tenant is never spendable in another; the sum of entries equals the balance. The suites, 94 tests on
Firestore and Postgres, state each one, and [`references/provenance.md`](references/provenance.md) has the
record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/ledger-wallet
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/ledger-wallet -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/ledger-wallet.git ~/.claude/skills/ledger-wallet
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/ledger-wallet ~/.agents/skills/ledger-wallet
```

Update the skill with `git pull` in its directory. The current release is **0.1.1**. See
[CHANGELOG.md](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: a wallet or stored credit, top-ups,
paying with a balance, a balance and a card on one order, refunds to a balance, balances in several
currencies, or auditing an existing balance module. Phrases such as "add funds", "pay with wallet", "store
credit", "split payment", "refund to wallet", "multi-currency balance" and "admin balance adjustment" match
it. Invoke it explicitly with `/ledger-wallet` in Claude Code, `$ledger-wallet` in Codex CLI, or from
`/skills` in Gemini CLI.

Pass the currencies customers can hold as the argument, first one the default:
`/ledger-wallet EUR,USD,GBP`. Without an argument the skill asks, offering USD.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: when (not) to use, architecture and the adaptation contract, six critical facts, five hard rules, quick start, reference directory |
| `README.md` | This front door |
| `CHANGELOG.md` | Keep a Changelog, one section per release, newest first |
| `CLAUDE.md` | What this repository is and the conventions for editing the skill itself |
| `LICENSE` | MIT |
| `references/data-model.md` | Rename table, entry kinds and refs, invariants, types, error codes |
| `references/money.md` | Resolving the currency list, minor units for 0, 2 and 3 decimals, Stripe amounts and minimums, limits |
| `references/engine.md` | The pure core, the store port, the wallet service and its behavior table |
| `references/firestore.md` | Firestore store, sharing a transaction, indexes, security rules |
| `references/postgres.md` | Schema, store over a minimal client interface, how each race resolves, driver traps |
| `references/stripe-top-up.md` | Checkout top-up, the webhook handler and route, the return page, setup checklist |
| `references/split-payment.md` | Wallet, card or both; holds; host callbacks; refunds to source; the sweeper |
| `references/integration.md` | Paying inside the order's transaction on both stores, order fields, the bookable-events adapter |
| `references/api-routes.md` | Route surface, the host seam file, shared helpers, schemas, every route handler, error contract |
| `references/ui.md` | Customer wallet panel, history, data hooks, client seam, the strings table |
| `references/admin-ui.md` | Staff panel, adjustment dialog, design decisions, extensions |
| `references/operations.md` | Configuration, scheduled work, what to watch, procedures, seeds and imports, go-live |
| `references/testing.md` | Vitest config, the in-memory store, the money and engine suites |
| `references/testing-payments.md` | The Stripe suite with a fake Stripe, the route suite with the seam mocked |
| `references/testing-stores.md` | The store conformance suite and the order atomicity tests on real backends |
| `references/provenance.md` | The engineering ledger: what the audit changed and how the templates verify it, what was kept on purpose, what is new in the skill, and the order of work on an existing module |

The seam is the adaptation contract table in `SKILL.md`, which this skill keeps there instead of in a
`references/adaptation.md`, with the rename table in `references/data-model.md`. It bounds the store, behind
the `WalletStore` port with Firestore, Postgres and in-memory implementations; identity, roles and tenant
scope, behind `lib/wallet/host.ts`; the host's orders, behind `postInTx`, `SplitCallbacks` and
`refundOrder`; the currency list, in `lib/wallet/config.ts`; and the host's strings, styling, UI primitives
and scheduler.

## The five non-negotiables

1. **Every balance change is an entry, posted through the wallet.** Seeds, imports and corrections included,
   because a balance written anywhere else can no longer be explained or reconciled. The reconciliation test
   holds that a write outside the ledger is detected.
2. **Tenant and customer come from the session or the webhook endpoint, never the request body.** A body
   field lets anyone move money in someone else's wallet or another tenant's. Tests hold that refs are scoped
   per tenant, that another tenant's session is not credited, and that another tenant's customer is a 404.
3. **Money moves once, with what it pays for.** A wallet payment commits in the order's transaction; a mixed
   payment holds and captures only when the card pays, because a debit before the order exists has nothing
   to recover it from. The order tests on Firestore and Postgres hold that a failed order moves no money.
4. **Credit only what Stripe collected.** The route that opens Checkout credits nothing; the webhook credits
   `amount_total` once, on a paid session, because the redirect is not proof of payment. Tests hold that a
   resent event and an unsettled async payment change nothing.
5. **Refund what was paid, to where it came from.** The amounts the order recorded when it was paid, per
   source, never its current total, because a total can change after payment and an unpaid order has
   nothing to refund. The refund tests hold both parts, their idempotency and the unpaid case.

Everything else is the host app's: authentication, tenancy, the order model, strings, locale, styling and the
store behind the seam.

## Requirements

Next.js App Router, the `stripe` package, `zod`, and either `firebase-admin` or a Postgres client with a
`pg`-compatible pool. Vitest on Node.js 20.11 or later for the shipped suites. A scheduler that can call one
route every 15 minutes, such as Vercel Cron. Number formatting uses the platform's `Intl`.

## Verification

Every TypeScript and SQL block in `references/` names its destination on its first line. Before a release the
blocks are written to a scratch project, type-checked under `strict` and under `noUncheckedIndexedAccess`,
the schema is applied to a real Postgres, and the suites run under Vitest against the in-memory store, the
Firestore emulator and Postgres: 68 unit tests, an 8-test store contract on each of the three backends, and 2
order atomicity tests, 94 in all. Stripe facts (zero-decimal currencies, the ISK and UGX representation,
minimum charges, Checkout expiry bounds, webhook retry window) were checked against Stripe's documentation.
`CLAUDE.md` has the recipe.

## Not this

| Not this | Use instead |
|---|---|
| Event registration, tickets, no-show deposits | The sibling [`bookable-events`](https://github.com/timerise-ai/bookable-events) skill, which takes this wallet as its `BalanceLedger` |
| Marketplace payouts, connected accounts, subscriptions | The sibling [`stripe-connect-subscriptions`](https://github.com/timerise-ai/stripe-connect-subscriptions) skill |
| Walk-up touchscreen booking | The sibling [`booking-kiosk`](https://github.com/timerise-ai/booking-kiosk) skill |
| Loyalty points, gift cards, vouchers | A loyalty or voucher system with its own expiry and liability rules |
| Currency conversion, FX wallets | A treasury or FX provider; this wallet never converts |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every block names its destination on the first line, and every TypeScript block is written to compile as one
project under `strict` and `noUncheckedIndexedAccess` and to pass its suites under Vitest, 94 tests with the
Firestore emulator and Postgres. Claims in this skill are meant to be verifiable: if you change a factual
claim, say how you verified it, whether against Stripe's documentation or API reference, Stripe test mode,
the Firestore emulator, a Postgres reproduction, or ISO 4217.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
