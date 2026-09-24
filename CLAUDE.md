# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json`, build or dev
server here, and nothing in this repository executes. It teaches an agent to build a customer wallet with a
balance per currency, Stripe top-ups, split payments and refunds in *someone else's* **Next.js App Router**
app, on Firestore or Postgres. The version of record is the git tag.

The commands and code in `references/` describe the app the agent will generate, not this repository. The one
thing checked here is that the templates compile and their tests pass, in a scratch project, against the
in-memory store, the Firestore emulator and a real Postgres; the recipe is below.

The skill was written by the engineer who owns the module, and audited against the earlier implementation.
`references/provenance.md` is the rationale layer and the ledger of that audit: what the audit changed and how
the templates verify it, what was kept deliberately with the reason it is safe, and what was designed in the
skill. Read it before "simplifying" anything.

Sibling directories under `../` (`bookable-events`, `stripe-connect-subscriptions`, `booking-kiosk`, and so
on) are other skills, not dependencies. They are referenced by name from `SKILL.md` and `README.md`;
`bookable-events` declares the `BalanceLedger` port that `references/integration.md` implements, so a change
to that port on either side is a change to both. The standard every skill follows is `../skills/STANDARD.md`.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface. The frontmatter has a third
  field beyond the standard's two, `argument-hint`, because the currency list is the skill's argument; hosts
  that do not know the field ignore it. The body carries the architecture and, under it, the **adaptation
  contract** table: this skill keeps its seam there instead of in a `references/adaptation.md`.
- `README.md`: the human-facing front door, in the section order every Timerise skill shares.
- `CHANGELOG.md`: Keep a Changelog, newest release first. The version lives here, in the README's
  current-release line, and in the git tag, and the three agree.
- `LICENSE`: MIT, identical to the siblings'.
- `references/*.md`: fifteen topic files plus `provenance.md`. `data-model.md` holds the rename table;
  `money.md` the currency list procedure and conversions; `engine.md` the pure core, port and service;
  `firestore.md` and `postgres.md` the stores; `stripe-top-up.md`, `split-payment.md` and `integration.md` the
  money flows; `api-routes.md`, `ui.md` and `admin-ui.md` the surface; `operations.md` running it;
  `testing.md`, `testing-payments.md` and `testing-stores.md` the suites; `provenance.md` the ledger.

## Editing conventions

- **Code blocks name their destination on the first line**: `// lib/wallet/wallet.ts: description`, or
  `-- db/wallet.sql: description` for SQL. The path is the first word after the comment marker, without the
  trailing colon. In client components the path comment comes before `'use client'`, which Next.js allows.
- **The code blocks are compiled and run.** Write each TypeScript and SQL block to its path in a scratch
  project (use the scratchpad, never this repo), give it a `node_modules` that has `next`, `react`, `zod`,
  `stripe`, `firebase-admin`, `pg`, `@types/pg` and `vitest`, and run:

  ```bash
  tsc --noEmit -p . && tsc --noEmit -p . --noUncheckedIndexedAccess
  vitest run                                   # 76 pass, 2 skipped: no backends
  firebase emulators:start --only firestore --project demo-wallet &
  createdb wallet_test && psql -d wallet_test -v ON_ERROR_STOP=1 -f db/wallet.sql
  FIRESTORE_EMULATOR_HOST=127.0.0.1:8080 WALLET_PG_URL=postgres://localhost/wallet_test vitest run   # 94 pass
  ```

  No `tsconfig.json` ships with the templates. The scratch one should be `strict`, include `DOM`, use
  `"jsx": "react-jsx"`, and map the `@/` alias (`"paths": { "@/*": ["./*"] }`); `vitest.config.ts` (in
  `testing.md`) maps the same alias for the tests. Both type-checks must be clean and all tests must pass.
  Do not add a `package.json` to this repository.
- **Test counts are claims.** `testing.md` states 15 + 20 + 24 + 9 unit tests, 8 contract tests per backend
  and 2 order tests: 76 without backends, 94 with both. `README.md`, `SKILL.md` and `CHANGELOG.md` repeat the
  totals. Change a suite, change the numbers.
- **Test names are cited.** `testing.md`, `testing-payments.md`, `testing-stores.md` and `provenance.md`
  quote test names; renaming a test means updating the citation.
- **Keep the three indexes in sync** with `references/`: the quick start and the reference directory in
  `SKILL.md`, and the file table in `README.md`. Cross-links between references are relative
  (`[engine.md](engine.md)`); links from `SKILL.md` are `references/`-prefixed.
- **Identifiers are shared across files.** `applyCommand`, `createWallet`, `postInTx`, `WalletStore`,
  `WalletTx`, `WalletEntry`, `WalletHold`, `EntryKind`, `WalletError`, `balanceRows`, `planPayment`,
  `startSplitPayment`, `refundOrder`, `handleWalletEvent`, `sweepStaleHolds`, `toBalanceLedger`,
  `WALLET_CURRENCIES`, `orderRefs`, the ref formats of `data-model.md`, and the metadata types
  `wallet_top_up` and `wallet_split`. Rename in all files or none.
- **Two kinds of name.** The domain vocabulary the host renames is the rename table in `data-model.md`:
  wallet, tenant, customer, entry, top-up, spend, refund, adjustment, hold, order. The identifiers above are
  the authoring contract of this repository, which the host may rename in its own app but which must stay
  consistent here.
- **One writer.** Nothing writes a balance, an entry or a hold outside `postInTx`. The in-memory store's
  `forceBalance` exists only so a test can simulate a writer that breaks this rule.
- **The non-negotiables are never presented as optional.** The five hard rules in `SKILL.md` and the five
  non-negotiables in `README.md` are one list, in one order. Changing one is a MAJOR release and says what
  broke.
- **Measured and vendor numbers are load-bearing.** Stripe minimum charges, the ISK and UGX x100 rule, the
  Checkout expiry window (30 minutes to 24 hours; the template uses 31 to 1439, default 35), the webhook retry
  window (up to three days), the 90-minute sweeper cutoff, the 15-minute schedule, the 10-try return poll, the
  1 to 100 page size. Each appears in code and prose; change one, change all, and cite the Stripe page it was
  checked against.
- **Currency**: uppercase ISO 4217 codes, `USD` default, stored in the currency's own minor unit, lowercased
  only in the three Stripe files. The currency list comes from the skill argument; never ship a fixed list.
- **Env var names** are canonical: `CRON_SECRET`, and in the tests `FIRESTORE_EMULATOR_HOST` and
  `WALLET_PG_URL`. Stripe keys and webhook secrets are per tenant, behind `host.ts`.
- **A template that needs something new from the host adds a row** to the adaptation contract in `SKILL.md`
  and a function to `host.ts`.
- **Do not remove the odd-looking parts.** The allow-list checked at Checkout rather than in the engine, the
  1 ms bump for same-millisecond entries, the Checkout created before the hold, the late `SPEND` after a
  released hold, `charge_already_refunded` treated as done, `FOR UPDATE` on holds. Each is a ledger entry.
  Check `provenance.md` before touching one.
- **`provenance.md` stays truthful.** It never names the earlier implementation's paths or domain terms. A new
  defect fixed is a numbered entry under *Fixed in the templates*; a questionable choice kept goes under
  *Kept deliberately* with the reason; a new capability is a row under *Added*.
- **Origin wording.** Call the codebase the audit ran against "the earlier implementation", and follow
  section 2 of the standard: none of its banned words, no denials, no corpus figures, and no defect counts or
  failure stories on the front door (README intro, `SKILL.md`, the description, this file).
- **Plain punctuation.** No em-dashes, en-dashes, arrows, middle dots, ellipsis characters or smart quotes
  anywhere in this repository's markdown, code blocks included; diagrams are plain ASCII.
  `LC_ALL=C grep -rn '[^ -~	]' --include='*.md' .` must print nothing. Prose wraps at 110 columns; table
  rows and commands stay on one line.
- **Claims are verifiable.** A changed factual claim says how it was verified: against Stripe's documentation
  or API reference, Stripe test mode, the Firestore emulator, a Postgres reproduction, or ISO 4217. Never from
  memory.
- **Commits follow Conventional Commits**, and no file or commit message names a tool or a model as author.
