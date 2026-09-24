# API routes

Every wallet route follows one shape: resolve the session through the host seam, check the feature gate,
validate the input with zod, call the wallet, answer with `json()` or `errorResponse()`. The tenant always
comes from the session or the webhook path.

| Route | Who | Does |
|---|---|---|
| `GET /api/wallet` | customer | balances in every listed currency, preferred currency, top-up limits |
| `GET /api/wallet/entries?currency&cursor&limit` | customer | history, newest first; actor type only, never staff ids |
| `PUT /api/wallet/preferred-currency` | customer | stores the customer's currency choice |
| `POST /api/wallet/top-up` | customer | opens Checkout ([stripe-top-up.md](stripe-top-up.md)) |
| `GET /api/wallet/top-up/status?session` | customer | `pending` or `credited`, for the return page |
| `POST /api/wallet/webhook/[tenantId]` | Stripe | credits, captures, releases ([stripe-top-up.md](stripe-top-up.md)) |
| `GET /api/admin/wallets/[customerId]` | staff, any role | balances, attributed history, reconciliation result |
| `POST /api/admin/wallets/[customerId]/adjust` | staff, `admin` | signed correction with a required note |
| `GET /api/cron/wallet-holds` | scheduler | settles holds whose webhook was missed ([split-payment.md](split-payment.md)) |

Paths are a suggestion; rename them to the host's convention (`/api/account/credit`, a `(dashboard)` route
group) and keep the table's contract.

## The host seam

Every host-specific decision is one function in this file. The templates throw a clear error from each until
the host implements it, so a missing wiring fails loudly on the first request instead of behaving as an empty
wallet.

```ts
// lib/wallet/host.ts: THE server-side seam. Every function here is the host's to
// implement with its own auth, tenancy, Stripe setup and database. Nothing else in
// lib/wallet or app/api reaches outside the wallet except through this file.
import type Stripe from 'stripe';
import type { WalletStore } from './ports';
import type { SplitCallbacks } from './webhook';

export interface CustomerSession {
  tenantId: string;              // derived from the session / domain, never from the request body
  customerId: string;
  email: string | null;
  stripeCustomerId?: string | null;
  locale: string;
}

export type StaffRole = 'viewer' | 'operator' | 'admin';

export interface StaffSession {
  tenantId: string;
  staffId: string;
  role: StaffRole;
}

const todo = (name: string): never => {
  throw new Error(`lib/wallet/host.ts: implement ${name}() for this app`);
};

/** The signed-in customer, or null for a 401. */
export async function getCustomerSession(_req: Request): Promise<CustomerSession | null> {
  return todo('getCustomerSession');
}

/** The signed-in staff member, or null for a 401. Role order: viewer < operator < admin. */
export async function getStaffSession(_req: Request): Promise<StaffSession | null> {
  return todo('getStaffSession');
}

/** Does this customer belong to this tenant? Guards every staff route taking a customerId. */
export async function customerExists(_tenantId: string, _customerId: string): Promise<boolean> {
  return todo('customerExists');
}

/** The feature gate. Checked by every wallet route, not only by the UI. */
export async function isWalletEnabled(_tenantId: string): Promise<boolean> {
  return todo('isWalletEnabled');
}

/** The Stripe client for the account that receives this tenant's money. */
export function getStripe(_tenantId: string): Stripe {
  return todo('getStripe');
}

/** The signing secret of this tenant's wallet webhook endpoint. */
export function getWebhookSecret(_tenantId: string): string | null {
  return todo('getWebhookSecret');
}

/** Firestore: createFirestoreWalletStore(db, DEFAULT_CURRENCY). Postgres: createPostgresWalletStore(pool, ...). */
export function getWalletStore(): WalletStore {
  return todo('getWalletStore');
}

/** Absolute URL on the tenant's site, locale-prefixed if the app uses locale routing. */
export function absoluteUrl(_tenantId: string, _locale: string, _path: string): string {
  return todo('absoluteUrl');
}

/** Server-side translation, for the Stripe line-item name. */
export function translate(_locale: string, _key: string, _params?: Record<string, string>): string {
  return todo('translate');
}

/** Order confirmation / release for wallet+card payments. Empty if the app has no split payments. */
export const splitCallbacks: SplitCallbacks = {};
```

| Function | Firebase Auth host | NextAuth / Auth.js host | Supabase host |
|---|---|---|---|
| `getCustomerSession` | verify the ID token or session cookie, look up the customer | `auth()`, map the user | `supabase.auth.getUser()` |
| `getStaffSession` | the host's staff claim or staff table, with role | role on the session | role in `app_metadata` or a staff table |
| `customerExists` | read the customer doc and compare its tenant | query the users table by tenant | query with the service role |
| `getWalletStore` | `createFirestoreWalletStore(getFirestore(), DEFAULT_CURRENCY)` | `createPostgresWalletStore(pool, DEFAULT_CURRENCY)` | `createPostgresWalletStore(pool, DEFAULT_CURRENCY)` with the pooled connection string |

## Shared helpers

```ts
// lib/wallet/server.ts: the one wallet instance, and the HTTP helpers every route shares.
import { NextResponse } from 'next/server';
import type { z } from 'zod';
import { walletPolicy } from './config';
import { WALLET_ERROR_STATUS, WalletError, isWalletError } from './errors';
import { getWalletStore, isWalletEnabled } from './host';
import { createWallet, type Wallet } from './wallet';

let instance: Wallet | undefined;
export function getWallet(): Wallet {
  instance ??= createWallet({ store: getWalletStore(), policy: walletPolicy });
  return instance;
}

export function json(data: unknown, status = 200): NextResponse {
  // Balances are per-customer and change constantly: never cache.
  return NextResponse.json(data, { status, headers: { 'Cache-Control': 'no-store' } });
}

/** One error shape for every wallet route: { error: { code, message, details } }. */
export function errorResponse(err: unknown): NextResponse {
  if (isWalletError(err)) {
    return json({ error: { code: err.code, message: err.message, details: err.details } }, WALLET_ERROR_STATUS[err.code]);
  }
  console.error('[wallet]', err);
  return json({ error: { code: 'INTERNAL', message: 'Something went wrong' } }, 500);
}

export const unauthorized = () => json({ error: { code: 'UNAUTHORIZED', message: 'Sign in required' } }, 401);
export const forbidden = () => json({ error: { code: 'FORBIDDEN', message: 'Not allowed' } }, 403);
export const notFound = () => json({ error: { code: 'NOT_FOUND', message: 'Not found' } }, 404);

export function invalid(error: z.ZodError): NextResponse {
  return json({ error: { code: 'INVALID_REQUEST', message: 'Invalid request', details: { issues: error.issues } } }, 400);
}

export async function readJson(req: Request): Promise<unknown> {
  try {
    return await req.json();
  } catch {
    return null; // fails schema validation: a 400, not a 500
  }
}

export async function assertWalletEnabled(tenantId: string): Promise<void> {
  if (!(await isWalletEnabled(tenantId))) throw new WalletError('WALLET_DISABLED', 'The wallet is not enabled');
}
```

## Validation

```ts
// lib/wallet/schemas.ts: request shapes. Currency *validity* is checked by the wallet
// (it knows the policy); these check structure and bounds.
import { z } from 'zod';

const currency = z.string().trim().min(3).max(3);

export const topUpBody = z.object({
  currency,
  amount: z.number().int().positive(), // minor units, the UI converts with parseMajorToMinor
});

export const preferredCurrencyBody = z.object({ currency });

export const entriesQuery = z.object({
  currency: currency.optional(),
  cursor: z.string().max(500).optional(),
  limit: z.coerce.number().int().min(1).max(100).optional(),
});

export const topUpStatusQuery = z.object({ session: z.string().startsWith('cs_').max(255) });

export const adjustBody = z.object({
  currency,
  delta: z.number().int().refine((d) => d !== 0, 'delta must not be 0'),
  note: z.string().trim().min(3).max(500),  // required: every correction says why
  requestId: z.uuid(),                      // generated when the dialog opens; a double submit is a replay
});
```

The schemas check shape and bounds. Whether a currency is known and enabled is the wallet's call, because
only the policy knows; the wallet answers `INVALID_CURRENCY` or `CURRENCY_NOT_ALLOWED` with the allowed list
in `details`.

## Customer routes

```ts
// app/api/wallet/route.ts: GET: the signed-in customer's balances in every enabled currency.
import { walletPolicy } from '@/lib/wallet/config';
import { getCustomerSession } from '@/lib/wallet/host';
import { assertWalletEnabled, errorResponse, getWallet, json, unauthorized } from '@/lib/wallet/server';
import { balanceRows } from '@/lib/wallet/wallet';

export async function GET(req: Request) {
  try {
    const session = await getCustomerSession(req);
    if (!session) return unauthorized();
    await assertWalletEnabled(session.tenantId);
    const snapshot = await getWallet().getWallet(session.tenantId, session.customerId);
    return json({
      preferredCurrency: snapshot.preferredCurrency,
      currencies: walletPolicy.allowed,
      topUpLimits: walletPolicy.topUpLimits,
      balances: balanceRows(snapshot, walletPolicy),
    });
  } catch (err) {
    return errorResponse(err);
  }
}
```

```ts
// app/api/wallet/entries/route.ts: GET ?currency=&cursor=&limit= : history, newest first.
import { getCustomerSession } from '@/lib/wallet/host';
import { entriesQuery } from '@/lib/wallet/schemas';
import { assertWalletEnabled, errorResponse, getWallet, invalid, json, unauthorized } from '@/lib/wallet/server';

export async function GET(req: Request) {
  try {
    const session = await getCustomerSession(req);
    if (!session) return unauthorized();
    await assertWalletEnabled(session.tenantId);
    const parsed = entriesQuery.safeParse(Object.fromEntries(new URL(req.url).searchParams));
    if (!parsed.success) return invalid(parsed.error);
    const page = await getWallet().listEntries({ tenantId: session.tenantId, customerId: session.customerId, ...parsed.data });
    // Customers see the reason, not which staff member made an adjustment.
    return json({
      nextCursor: page.nextCursor,
      entries: page.entries.map(({ actor, tenantId: _t, customerId: _c, ...e }) => ({ ...e, actorType: actor.type })),
    });
  } catch (err) {
    return errorResponse(err);
  }
}
```

```ts
// app/api/wallet/preferred-currency/route.ts: PUT { currency }: remember the customer's choice.
import { getCustomerSession } from '@/lib/wallet/host';
import { preferredCurrencyBody } from '@/lib/wallet/schemas';
import { assertWalletEnabled, errorResponse, getWallet, invalid, json, readJson, unauthorized } from '@/lib/wallet/server';

export async function PUT(req: Request) {
  try {
    const session = await getCustomerSession(req);
    if (!session) return unauthorized();
    await assertWalletEnabled(session.tenantId);
    const parsed = preferredCurrencyBody.safeParse(await readJson(req));
    if (!parsed.success) return invalid(parsed.error);
    const preferredCurrency = await getWallet().setPreferredCurrency(session.tenantId, session.customerId, parsed.data.currency);
    return json({ preferredCurrency });
  } catch (err) {
    return errorResponse(err);
  }
}
```

The top-up and status routes are in [stripe-top-up.md](stripe-top-up.md).

## Staff routes

```ts
// app/api/admin/wallets/[customerId]/route.ts: GET: balances, history (with actors) and
// a reconciliation check, for staff of the customer's own tenant.
import { walletPolicy } from '@/lib/wallet/config';
import { customerExists, getStaffSession } from '@/lib/wallet/host';
import { entriesQuery } from '@/lib/wallet/schemas';
import { errorResponse, getWallet, invalid, json, notFound, unauthorized } from '@/lib/wallet/server';
import { balanceRows } from '@/lib/wallet/wallet';

export async function GET(req: Request, ctx: { params: Promise<{ customerId: string }> }) {
  try {
    const staff = await getStaffSession(req);
    if (!staff) return unauthorized();
    const { customerId } = await ctx.params;
    // Tenant scope on the [id] route: a customer of another tenant is simply not found.
    if (!(await customerExists(staff.tenantId, customerId))) return notFound();
    const parsed = entriesQuery.safeParse(Object.fromEntries(new URL(req.url).searchParams));
    if (!parsed.success) return invalid(parsed.error);

    const wallet = getWallet();
    const [snapshot, page, mismatches] = await Promise.all([
      wallet.getWallet(staff.tenantId, customerId),
      wallet.listEntries({ tenantId: staff.tenantId, customerId, ...parsed.data }),
      wallet.reconcile(staff.tenantId, customerId),
    ]);
    return json({
      preferredCurrency: snapshot.preferredCurrency,
      currencies: walletPolicy.allowed,
      balances: balanceRows(snapshot, walletPolicy),
      entries: page.entries,
      nextCursor: page.nextCursor,
      reconciled: mismatches.length === 0,
      mismatches,
    });
  } catch (err) {
    return errorResponse(err);
  }
}
```

```ts
// app/api/admin/wallets/[customerId]/adjust/route.ts: POST { currency, delta, note, requestId }:
// a signed correction with a mandatory reason, attributed to the staff member.
import { customerExists, getStaffSession } from '@/lib/wallet/host';
import { adjustBody } from '@/lib/wallet/schemas';
import { errorResponse, forbidden, getWallet, invalid, json, notFound, readJson, unauthorized } from '@/lib/wallet/server';

export async function POST(req: Request, ctx: { params: Promise<{ customerId: string }> }) {
  try {
    const staff = await getStaffSession(req);
    if (!staff) return unauthorized();
    if (staff.role !== 'admin') return forbidden();
    const { customerId } = await ctx.params;
    if (!(await customerExists(staff.tenantId, customerId))) return notFound();
    const parsed = adjustBody.safeParse(await readJson(req));
    if (!parsed.success) return invalid(parsed.error);

    const { entry, replayed } = await getWallet().adjust({
      tenantId: staff.tenantId,
      customerId,
      currency: parsed.data.currency,
      delta: parsed.data.delta,
      ref: `adjust:${parsed.data.requestId}`,
      reason: { key: 'wallet.reason.adjustment', params: { note: parsed.data.note } },
      actor: { type: 'staff', id: staff.staffId },
    });
    return json({ entry, replayed }, replayed ? 200 : 201);
  } catch (err) {
    return errorResponse(err);
  }
}
```

| Guard | Why |
|---|---|
| `customerExists(staff.tenantId, customerId)` on every `[customerId]` route | a customer of another tenant answers 404, the same as one that does not exist |
| `role === 'admin'` for adjustments | reading is broader than writing; map the role names to the host's |
| `requestId` generated when the dialog opens | a double click or a retried request replays instead of crediting twice |
| `note` required, 3 to 500 characters | every correction says why, in the history the customer and support both see |

## Error contract

| Status | `error.code` | When |
|---|---|---|
| 400 | `INVALID_REQUEST` | body or query fails the schema; `details.issues` lists why |
| 400 | `INVALID_AMOUNT`, `INVALID_CURRENCY`, `CURRENCY_NOT_ALLOWED`, `AMOUNT_OUT_OF_RANGE`, `NOT_CHARGEABLE` | the wallet refused the input; `details` carries the numbers |
| 401 | `UNAUTHORIZED` | no session |
| 402 | `INSUFFICIENT_FUNDS` | `details: { available, required }` |
| 403 | `FORBIDDEN`, `WALLET_DISABLED` | role too low; feature off for the tenant |
| 404 | `NOT_FOUND`, `HOLD_NOT_FOUND` | unknown or other-tenant customer; unknown hold |
| 409 | `REF_CONFLICT`, `HOLD_NOT_OPEN` | ref reused for a different movement; hold already settled |
| 500 | `INTERNAL` | anything else, logged with `[wallet]` |

The UI maps `error.code` to a translated message and interpolates `details` ([ui.md](ui.md)). Every wallet
response carries `Cache-Control: no-store`. Route behavior is covered by `test/routes.test.ts`
([testing-payments.md](testing-payments.md)).
