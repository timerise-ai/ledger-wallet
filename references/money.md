# Money and currencies

Every amount is an integer in the currency's **real minor unit**: USD 1999 is $19.99, JPY 500 is 500 yen,
ISK 500 is 500 kronur, KWD 1250 is 1.250 dinar. Currency codes are **uppercase ISO 4217** (`'USD'`)
everywhere in storage, APIs and UI, and are lowercased only in the calls to Stripe.

## Resolving the currency list

The host's supported currencies are an input to this skill, not a constant inside it. Before generating any
file:

1. **Read the skill arguments.** `/ledger-wallet EUR,USD,GBP` (Claude Code), `$ledger-wallet EUR USD`
   (Codex CLI), or the codes written after the skill name in any other host. Split on commas, spaces or
   semicolons and run the tokens through `parseCurrencyList`: codes are uppercased, order is kept, duplicates
   are dropped.
2. **Reject unknown codes by name.** If `invalid` is not empty, tell the user which tokens are not ISO 4217
   codes and ask again. Never drop them silently.
3. **No codes given: ask.** Ask the user which currencies customers can hold, offering `USD` as the default
   answer. If the host already has a currency setting (an env var, a config file, a settings table), show
   what it contains and offer it as the answer.
4. **The first code is the default.** It becomes `DEFAULT_CURRENCY` and the `preferredCurrency` of every new
   wallet. With no answer at all, the list is `['USD']`.
5. **Write the list into `lib/wallet/config.ts`** by replacing the `WALLET_CURRENCIES` array. Everything else
   reads from there: the allow-list, the top-up limits, the API's zero-filled balance list, the UI switcher
   (shown only when more than one currency is listed), and the seeds.
6. **Review the top-up limits** for each listed currency. The defaults below suit currencies worth roughly a
   dollar or a euro. For a currency worth much less, add an override in `config.ts`.

| Currency class | Examples | Default top-up min | Default top-up max |
|---|---|---|---|
| Two-decimal | USD, EUR, GBP, CHF | 500 (5.00) | 100000 (1,000.00) |
| Zero-decimal | JPY, KRW, ISK | 500 (500 units) | 200000 |
| Three-decimal | KWD, BHD, OMR | 2000 (2.000) | 300000 (300.000) |

`createCurrencyPolicy` raises any minimum that sits below the Stripe minimum charge for that currency, so a
top-up the policy allows is always one Stripe can charge.

```ts
// lib/wallet/config.ts: the host's single source of truth for wallet currencies.
//
// The skill's "resolve the currency list" step REPLACES the array below with the
// codes the user passed as the skill argument (or answered when asked).
// The first code is the default. With no answer: ['USD'].
import { createCurrencyPolicy } from './money';

export const WALLET_CURRENCIES = ['USD'] as const;

export const walletPolicy = createCurrencyPolicy(WALLET_CURRENCIES, {
  // Per-currency top-up bounds in minor units, for currencies the exponent defaults
  // do not suit, e.g. HUF: { min: 20_000, max: 20_000_000 }.
});

export const DEFAULT_CURRENCY = walletPolicy.defaultCurrency;
```

Adding a currency later is a one-line change to that array. **Removing one never hides money**: balances in
a currency no longer listed stay visible (flagged `disabled`), spendable and refundable; only new top-ups in
it are refused. See [operations.md](operations.md).

## The currency module

```ts
// lib/wallet/money.ts: currency codes, minor units, Stripe conversion, limits.
//
// Two different "minor units" exist and must never be mixed:
//   stored amount: the currency's real minor unit (USD 1999 = $19.99, ISK 500 = 500 kr, KWD 1250 = 1.250 KD)
//   Stripe amount: what the Stripe API expects; differs from stored for ISK and UGX only (x100)
import { WalletError } from './errors';

/** Uppercase ISO-4217 code: 'USD', 'EUR', 'ISK'. Lowercased only inside the Stripe calls. */
export type CurrencyCode = string;

// Stored with 0 decimals. Stripe's zero-decimal list plus ISK (zero-decimal in practice).
const ZERO_DECIMAL = new Set([
  'BIF', 'CLP', 'DJF', 'GNF', 'ISK', 'JPY', 'KMF', 'KRW', 'MGA', 'PYG',
  'RWF', 'UGX', 'VND', 'VUV', 'XAF', 'XOF', 'XPF',
]);
// Stored with 3 decimals (ISO 4217). Card amounts are kept ending in 0; see assertChargeableAmount.
const THREE_DECIMAL = new Set(['BHD', 'JOD', 'KWD', 'OMR', 'TND']);
// Zero-decimal currencies Stripe still wants as two-decimal integers (500 ISK is sent as 50000).
const STRIPE_TWO_DECIMAL_LEGACY = new Set(['ISK', 'UGX']);

// Stripe's minimum charge per currency, in stored minor units, as listed on Stripe's
// "Supported currencies" page. A charge converted into a different settlement currency must
// also meet that currency's minimum after conversion. Unlisted: let Stripe decide.
const STRIPE_MINIMUM: Record<string, number> = {
  USD: 50, AED: 200, ARS: 50, AUD: 50, BRL: 50, CAD: 50, CHF: 50, COP: 50, CZK: 1500,
  DKK: 250, EUR: 50, GBP: 30, HKD: 400, HUF: 17500, IDR: 50, ILS: 50, INR: 50, JPY: 50,
  KRW: 50, MXN: 1000, MYR: 200, NOK: 300, NZD: 50, PHP: 50, PLN: 200, RON: 200, RUB: 50,
  SEK: 300, SGD: 50, THB: 1000, ZAR: 50,
};

let knownCodes: Set<string> | null | undefined;
function isKnownCode(code: string): boolean {
  if (knownCodes === undefined) {
    // Intl knows every active ISO-4217 code in Node 18+ and current browsers.
    // Where it is missing, fall back to the shape check alone.
    const intl = Intl as { supportedValuesOf?: (key: 'currency') => string[] };
    knownCodes = intl.supportedValuesOf ? new Set(intl.supportedValuesOf('currency')) : null;
  }
  return knownCodes === null || knownCodes.has(code);
}

/** Uppercase ISO-4217 code, or null. The only way a code enters the wallet. */
export function normalizeCurrency(input: unknown): CurrencyCode | null {
  if (typeof input !== 'string') return null;
  const code = input.trim().toUpperCase();
  return /^[A-Z]{3}$/.test(code) && isKnownCode(code) ? code : null;
}

/**
 * Parse the skill argument / user answer: "EUR, usd GBP" gives ['EUR', 'USD', 'GBP'].
 * Order is kept (the first code becomes the default), duplicates dropped,
 * invalid tokens reported by name. An empty result means "ask the user".
 */
export function parseCurrencyList(input: string | readonly string[] | undefined): {
  currencies: CurrencyCode[];
  invalid: string[];
} {
  const tokens = (typeof input === 'string' ? input.split(/[\s,;]+/) : [...(input ?? [])])
    .map((t) => t.trim())
    .filter(Boolean);
  const currencies: CurrencyCode[] = [];
  const invalid: string[] = [];
  for (const token of tokens) {
    const code = normalizeCurrency(token);
    if (!code) invalid.push(token);
    else if (!currencies.includes(code)) currencies.push(code);
  }
  return { currencies, invalid };
}

/** Decimal places of the stored amount. */
export function minorUnitExponent(currency: CurrencyCode): 0 | 2 | 3 {
  if (ZERO_DECIMAL.has(currency)) return 0;
  if (THREE_DECIMAL.has(currency)) return 3;
  return 2;
}

export function isMinorUnitAmount(amount: unknown): amount is number {
  return typeof amount === 'number' && Number.isSafeInteger(amount) && amount > 0;
}

/** Stored minor units to a Stripe `unit_amount` or refund `amount`. */
export function toStripeAmount(amount: number, currency: CurrencyCode): number {
  return STRIPE_TWO_DECIMAL_LEGACY.has(currency) ? amount * 100 : amount;
}

/** Stripe `amount_total` to stored minor units. */
export function fromStripeAmount(stripeAmount: number, currency: CurrencyCode): number {
  // Stripe only accepts ISK/UGX amounts divisible by 100, so this is exact.
  return STRIPE_TWO_DECIMAL_LEGACY.has(currency) ? Math.round(stripeAmount / 100) : stripeAmount;
}

/** Stripe's currency parameter. */
export function stripeCurrency(currency: CurrencyCode): string {
  return currency.toLowerCase();
}

export function stripeMinimumCharge(currency: CurrencyCode): number | null {
  return STRIPE_MINIMUM[currency] ?? null;
}

/** Throws NOT_CHARGEABLE for an amount Stripe would reject. Call before creating a session. */
export function assertChargeableAmount(amount: number, currency: CurrencyCode): void {
  if (!isMinorUnitAmount(amount)) {
    throw new WalletError('INVALID_AMOUNT', 'Amount must be a positive integer in minor units', { amount });
  }
  const minimum = stripeMinimumCharge(currency);
  if (minimum !== null && amount < minimum) {
    throw new WalletError('NOT_CHARGEABLE', `Below the card minimum for ${currency}`, { amount, minimum, currency });
  }
  if (THREE_DECIMAL.has(currency) && amount % 10 !== 0) {
    throw new WalletError('NOT_CHARGEABLE', `${currency} card amounts must end in 0`, { amount, currency });
  }
}

/** Display only. Never parse the output back. */
export function formatMoney(amount: number, currency: CurrencyCode, locale = 'en-US'): string {
  const digits = minorUnitExponent(currency);
  return new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
    minimumFractionDigits: digits,
    maximumFractionDigits: digits,
  }).format(amount / 10 ** digits);
}

/**
 * Form input to stored minor units, by string arithmetic (never `parseFloat(x) * 100`:
 * 19.99 * 100 is 1998.9999999999998). Accepts "12", "12.5", "12,50". Returns null
 * for anything else, including more decimals than the currency has.
 */
export function parseMajorToMinor(input: string, currency: CurrencyCode): number | null {
  const digits = minorUnitExponent(currency);
  const match = /^(\d{1,12})(?:[.,](\d+))?$/.exec(input.trim());
  if (!match) return null;
  const whole = match[1] ?? '0';
  const fraction = match[2] ?? '';
  if (fraction.length > digits) return null;
  const value = Number(whole) * 10 ** digits + Number(fraction.padEnd(digits, '0') || '0');
  return Number.isSafeInteger(value) ? value : null;
}

// ---------------------------------------------------------------- policy

export interface AmountLimits {
  min: number;
  max: number;
}

export interface CurrencyPolicy {
  /** WALLET_CURRENCIES, in order. */
  allowed: readonly CurrencyCode[];
  /** allowed[0]. New wallets start with this as preferredCurrency. */
  defaultCurrency: CurrencyCode;
  /** Per-currency top-up bounds in stored minor units. */
  topUpLimits: Record<CurrencyCode, AmountLimits>;
}

// Starting points by exponent. A currency worth much less than a dollar (HUF,
// JPY, ISK, KRW) needs an explicit override: 5.00 HUF is below Stripe's minimum.
const DEFAULT_LIMITS_BY_EXPONENT: Record<0 | 2 | 3, AmountLimits> = {
  0: { min: 500, max: 200_000 },
  2: { min: 500, max: 100_000 },
  3: { min: 2_000, max: 300_000 },
};

export function createCurrencyPolicy(
  allowed: readonly string[],
  overrides: Partial<Record<CurrencyCode, AmountLimits>> = {},
): CurrencyPolicy {
  const { currencies, invalid } = parseCurrencyList(allowed);
  if (invalid.length > 0) throw new Error(`Unknown currency codes: ${invalid.join(', ')}`);
  const list = currencies.length > 0 ? currencies : ['USD'];
  const topUpLimits: Record<CurrencyCode, AmountLimits> = {};
  for (const currency of list) {
    const base = overrides[currency] ?? DEFAULT_LIMITS_BY_EXPONENT[minorUnitExponent(currency)];
    // Never let a limit sit below what Stripe can charge.
    const floor = stripeMinimumCharge(currency) ?? 0;
    topUpLimits[currency] = { min: Math.max(base.min, floor), max: base.max };
  }
  return { allowed: list, defaultCurrency: list[0] ?? 'USD', topUpLimits };
}

/** Parses and checks a currency against the policy. Throws INVALID_CURRENCY / CURRENCY_NOT_ALLOWED. */
export function requireAllowedCurrency(policy: CurrencyPolicy, input: unknown): CurrencyCode {
  const code = normalizeCurrency(input);
  if (!code) throw new WalletError('INVALID_CURRENCY', 'Unknown currency', { currency: input });
  if (!policy.allowed.includes(code)) {
    throw new WalletError('CURRENCY_NOT_ALLOWED', `${code} is not enabled`, { currency: code, allowed: policy.allowed });
  }
  return code;
}

export function assertTopUpAmount(policy: CurrencyPolicy, currency: CurrencyCode, amount: unknown): number {
  if (!isMinorUnitAmount(amount)) {
    throw new WalletError('INVALID_AMOUNT', 'Amount must be a positive integer in minor units', { amount });
  }
  const limits = policy.topUpLimits[currency];
  if (!limits) throw new WalletError('CURRENCY_NOT_ALLOWED', `${currency} is not enabled`, { currency });
  if (amount < limits.min || amount > limits.max) {
    throw new WalletError('AMOUNT_OUT_OF_RANGE', `Top-up must be between ${limits.min} and ${limits.max}`, {
      amount, currency, ...limits,
    });
  }
  assertChargeableAmount(amount, currency);
  return amount;
}
```

## Stripe amounts

| Currency class | Examples | Stored | Sent to Stripe |
|---|---|---|---|
| Two-decimal (default) | USD, EUR, GBP, PLN, HUF, TWD | cents | cents |
| Zero-decimal | JPY, KRW, CLP, VND, XOF | whole units | whole units |
| Zero-decimal, two-decimal on Stripe | **ISK, UGX** | whole units | x100 (500 ISK is sent as `50000`) |
| Three-decimal | BHD, JOD, KWD, OMR, TND | thousandths | thousandths, kept ending in 0 |

The zero-decimal list, the ISK and UGX representation and the minimum charge table were checked against
Stripe's "Supported currencies" page. That page also notes two things the table cannot capture:

- **The minimum depends on the settlement currency.** A charge that Stripe converts into the account's
  default settlement currency must meet that currency's minimum after conversion.
- **HUF and TWD are two-decimal for charges** and zero-decimal only for payouts. The wallet stores and charges
  them as two-decimal; nothing here creates payouts.

Stripe's page does not currently state a rule for three-decimal currencies. The templates keep card amounts
in those currencies ending in 0, which Stripe has required for them in the past; the cost is at most 0.009
of the currency, moved from the card part to the wallet part of a split. Confirm with one test-mode charge
before relying on either behavior.

## Rules the module encodes

| Rule | Why |
|---|---|
| `normalizeCurrency` is the only way a code enters the wallet | one place that uppercases, trims and checks against the ISO list |
| `parseMajorToMinor` parses by string, never `parseFloat(x) * 100` | `19.99 * 100` is `1998.9999999999998` in floating point |
| `formatMoney` uses `Intl.NumberFormat` with the currency's own digits | ISK shows no decimals, KWD shows three; never parse its output back |
| Stripe conversion happens only in the three Stripe files | `top-up.ts`, `split.ts`, `refunds.ts`; stored amounts never carry Stripe's representation |
| `assertTopUpAmount` checks integer, range and chargeability | a float, a string or an amount Stripe rejects fails before Checkout, with the numbers |

Tests: [testing.md](testing.md), `test/money.test.ts`.
