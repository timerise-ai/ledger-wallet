# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
