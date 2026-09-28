# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.3] - 2026-09-28

Fix release, from scoring the prompt-1 agent eval runs against 0.1.2. Apps built from 0.1.2 should copy in
`lib/wallet/split.ts`, `lib/wallet/wallet.ts` and `lib/wallet/postgres-store.ts`, and keep a `splitAttempt`
on each order.

### Fixed
- A split payment retried after its Checkout expired replayed the released hold, so the new Checkout ran with
  nothing reserved and could end in `wallet_short` after the card paid. `orderRefs` and `startSplitPayment`
  take an `attempt`; a settled attempt is refused with `HOLD_NOT_OPEN` and `details.nextAttempt`, and a second
  start of an open attempt returns its Checkout instead of opening another (`split-payment.md`).
- `reconcile` reported a movement that committed between its two reads as drift; a mismatch is now reported
  only when two reads in a row agree on it (`engine.md`).
- A forged history cursor reached Postgres as an invalid timestamp and answered 500; the Postgres store reads
  it as no cursor, as the other stores do (`postgres.md`).

### Changed
- The store and order tests are one file per backend, `test/postgres.test.ts` and `test/firestore.test.ts`,
  around a shared `test/store-contract.ts`; an app on one store copies its file as written. The Postgres order
  example keeps its tables in a `wallet_example` schema (`testing-stores.md`).
- The test config is `vitest.config.mts`, which Vitest 5 loads without a warning in any project
  (`testing.md`).
- The quick start says to copy templates as written, never to merge the webhook and cron routes, to install
  Vitest and leave the shipped tests unmodified, and what to hand over; `api-routes.md` says what identity is
  in an app with no auth (`SKILL.md`).
- 97 tests: 71 unit tests, the 8-test store contract on three backends, and 2 order tests.

## [0.1.2] - 2026-09-28

Documentation only; the skill content is unchanged from 0.1.1.

### Added
- `evals/prompts.md` with three operator prompts, and `.github/workflows/agent-eval.yml`, the caller of the
  index's eval workflow copied verbatim from section 10 of the standard, so every published release runs
  prompt 1 in Claude Code, Codex CLI and Gemini CLI.

### Changed
- `README.md` lists `evals/` and the eval workflow in the file table; `CLAUDE.md` describes both.
- `provenance.md` and `CLAUDE.md` state that what was designed in the skill has never run in production,
  in the words of the standard.

## [0.1.1] - 2026-09-25

### Fixed
- `vitest.config.ts` in `testing.md` resolves the `@` alias from `import.meta.dirname` instead of
  `__dirname`, which Vite's native config loader does not support and Vitest 5 warns about. The suites need
  Node.js 20.11 or later.

## [0.1.0] - 2026-09-25

Initial release of the `ledger-wallet` skill: a customer wallet with a balance per currency, Stripe
Checkout top-ups, wallet, card and split payments, refunds to source and staff adjustments, for a Next.js App
Router app on Firestore or Postgres.

### Added
- `SKILL.md` entry point: architecture and the adaptation contract, six critical facts, five hard rules, the
  quick-start order and the reference directory. The currency list is the skill argument, USD by default.
- Fifteen references: data model, money, engine, Firestore store, Postgres store, Stripe top-up, split
  payment, integration, API routes, customer UI, staff UI, operations, and three testing references; plus
  `provenance.md`, the audit ledger of the earlier implementation.
- Holds for payments split between the wallet and a card, captured when the card pays and released when it
  does not, with a sweeper for missed webhooks.
- Refunds that return the wallet part to the wallet and the card part to the card, idempotent on both sides.
- Any ISO 4217 currency in its own minor unit, including zero-decimal, three-decimal and the ISK and UGX
  Stripe representation, with per-currency top-up limits floored at Stripe's minimum charge.
- The `BalanceLedger` adapter for the `bookable-events` skill.
- 94 Vitest tests: 68 unit tests, an 8-test store contract run on the in-memory store, the Firestore
  emulator and Postgres, and 2 order atomicity tests.
- `README.md`, MIT `LICENSE`, `CLAUDE.md`.
