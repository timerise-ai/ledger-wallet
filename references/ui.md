# Customer UI

Three pieces: a **wallet panel** (every balance, the currency switcher, the top-up form, the return from
Stripe), a **history** list, and the **data hooks** behind them. The components ship structure and semantics
only. Swap `<button>`, `<input>`, `<select>`, `<table>` for the host's primitives and add its layout; keep the
labels, `aria-*` attributes, disabled states and error slots.

## What the panel must show

| Element | Rule |
|---|---|
| Balances | every listed currency plus any other currency with money in it, not only the selected one |
| Reserved | the `held` part per currency, labelled as reserved for a pending payment |
| Delisted currency | still shown, marked, spendable; no top-up form for it |
| Currency switcher | only when more than one currency is listed; initialized once from `preferredCurrency`; a change is saved with `PUT /api/wallet/preferred-currency` |
| Top-up amount | parsed with `parseMajorToMinor`; the range hint shows the limits formatted in the currency |
| Top-up error | the server's code, translated, with its numbers; never a generic failure |
| Return from Stripe | `?topUp=success&session=...`: "processing", then "added" once the webhook has credited; "cancelled" for `?topUp=cancelled` |
| Load failure | an error with a retry button; never a balance of 0 |

## Client seam

```ts
// lib/wallet/host-client.ts: the client-side seam: translation, locale, authenticated fetch.
'use client';
// Replace each body with the host's own (next-intl, react-i18next, a dictionary context, ...).

export type Translate = (key: string, params?: Record<string, string | number>) => string;

/** Default: echoes the key, so a missing wiring is visible instead of blank. */
export function useWalletT(): Translate {
  return (key, params) => (params ? `${key} ${JSON.stringify(params)}` : key);
}

export function useWalletLocale(): string {
  return typeof navigator === 'undefined' ? 'en-US' : navigator.language;
}

/** Cookie sessions: same-origin fetch. Bearer-token apps add the Authorization header here. */
export function walletFetch(input: string, init: RequestInit = {}): Promise<Response> {
  return fetch(input, { credentials: 'same-origin', ...init });
}
```

## Data hooks

```ts
// hooks/useWallet.ts: wallet data for the customer UI. Every request is abortable and
'use client';
// every response is checked: a failed fetch is an error state, never a silent zero.
import { useCallback, useEffect, useRef, useState } from 'react';
import { walletFetch } from '@/lib/wallet/host-client';
import type { AmountLimits } from '@/lib/wallet/money';
import type { BalanceRow } from '@/lib/wallet/wallet';

export interface WalletView {
  preferredCurrency: string;
  currencies: string[];
  topUpLimits: Record<string, AmountLimits>;
  balances: BalanceRow[];
}

export interface ApiError {
  code: string;
  message: string;
  details?: Record<string, unknown>;
}

export interface EntryView {
  id: string;
  kind: string;
  currency: string;
  amount: number;
  availableDelta: number;
  availableAfter: number;
  heldAfter: number;
  orderRef: string | null;
  reason: { key: string; params?: Record<string, string> };
  createdAt: string;
  actorType?: string;
  actor?: { type: string; id: string };
}

export async function readApiError(res: Response): Promise<ApiError> {
  try {
    const body = (await res.json()) as { error?: ApiError };
    if (body.error?.code) return body.error;
  } catch {
    /* not JSON */
  }
  return { code: res.status === 401 ? 'UNAUTHORIZED' : 'INTERNAL', message: res.statusText };
}

type Load<T> = { status: 'loading' } | { status: 'error'; error: ApiError } | { status: 'ready'; data: T };

export function useWallet() {
  const [state, setState] = useState<Load<WalletView>>({ status: 'loading' });
  const controller = useRef<AbortController | null>(null);

  const reload = useCallback(async () => {
    controller.current?.abort();
    const ac = new AbortController();
    controller.current = ac;
    try {
      const res = await walletFetch('/api/wallet', { signal: ac.signal });
      if (!res.ok) return setState({ status: 'error', error: await readApiError(res) });
      setState({ status: 'ready', data: (await res.json()) as WalletView });
    } catch (err) {
      if (!ac.signal.aborted) setState({ status: 'error', error: { code: 'NETWORK', message: String(err) } });
    }
  }, []);

  useEffect(() => {
    void reload();
    return () => controller.current?.abort();
  }, [reload]);

  return { state, reload };
}

/** History for one currency, with "load more". Switching currency discards in-flight pages. */
export function useWalletEntries(currency: string | null, endpoint = '/api/wallet/entries') {
  const [entries, setEntries] = useState<EntryView[]>([]);
  const [cursor, setCursor] = useState<string | null>(null);
  const [status, setStatus] = useState<'idle' | 'loading' | 'error'>('idle');
  const [error, setError] = useState<ApiError | null>(null);
  const generation = useRef(0);

  const fetchPage = useCallback(async (after: string | null, gen: number) => {
    setStatus('loading');
    const qs = new URLSearchParams({ limit: '25' });
    if (currency) qs.set('currency', currency);
    if (after) qs.set('cursor', after);
    try {
      const res = await walletFetch(`${endpoint}?${qs}`);
      if (gen !== generation.current) return; // a newer currency/reset won
      if (!res.ok) {
        setError(await readApiError(res));
        return setStatus('error');
      }
      const page = (await res.json()) as { entries: EntryView[]; nextCursor: string | null };
      if (gen !== generation.current) return;
      setEntries((prev) => (after ? [...prev, ...page.entries] : page.entries));
      setCursor(page.nextCursor);
      setStatus('idle');
    } catch (err) {
      if (gen !== generation.current) return;
      setError({ code: 'NETWORK', message: String(err) });
      setStatus('error');
    }
  }, [currency, endpoint]);

  useEffect(() => {
    const gen = ++generation.current;
    setEntries([]);
    setCursor(null);
    void fetchPage(null, gen);
  }, [fetchPage]);

  const loadMore = useCallback(() => {
    if (cursor && status !== 'loading') void fetchPage(cursor, generation.current);
  }, [cursor, status, fetchPage]);

  return { entries, hasMore: cursor !== null, status, error, loadMore };
}
```

| Hook behavior | Why |
|---|---|
| Every request is abortable and aborted on unmount or reload | a slow response cannot overwrite a newer one |
| `res.ok` checked before `res.json()` | a 402 or 500 becomes an error state with its code, not a crash or a zero |
| History pages carry a generation counter | switching currency mid-request discards the old currency's page |
| `useWalletEntries` takes an `endpoint` | the admin panel reuses it with the admin route, which adds actors |

## The panel

```tsx
// components/wallet/WalletPanel.tsx: customer wallet: every balance, top-up, return handling.
'use client';
// STRUCTURE ONLY. Swap <button>/<input>/<select>/<table> for the host's primitives and
// add its layout; keep the semantics (labels, aria-live, disabled states, error slots).
import { useEffect, useRef, useState } from 'react';
import { useWallet, readApiError, type ApiError } from '@/hooks/useWallet';
import { useWalletLocale, useWalletT, walletFetch } from '@/lib/wallet/host-client';
import { formatMoney, parseMajorToMinor } from '@/lib/wallet/money';
import { WalletHistory } from './WalletHistory';

export function WalletPanel() {
  const t = useWalletT();
  const locale = useWalletLocale();
  const { state, reload } = useWallet();
  const [currency, setCurrency] = useState<string | null>(null);
  const initialized = useRef(false);

  // Take the server's preferredCurrency ONCE. Later reloads must not undo the customer's pick.
  useEffect(() => {
    if (state.status === 'ready' && !initialized.current) {
      initialized.current = true;
      setCurrency(state.data.preferredCurrency);
    }
  }, [state]);

  const chooseCurrency = (next: string) => {
    setCurrency(next);
    // Persist so other devices and server-rendered pages agree. Failure is harmless.
    void walletFetch('/api/wallet/preferred-currency', {
      method: 'PUT', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ currency: next }),
    }).catch(() => undefined);
  };

  if (state.status === 'loading') return <p aria-busy="true">{t('wallet.loading')}</p>;
  if (state.status === 'error') {
    return (
      <div role="alert">
        <p>{t(`wallet.errors.${state.error.code}`)}</p>
        <button type="button" onClick={() => void reload()}>{t('wallet.retry')}</button>
      </div>
    );
  }
  const { data } = state;
  const active = currency ?? data.preferredCurrency;

  return (
    <section aria-labelledby="wallet-title">
      <h2 id="wallet-title">{t('wallet.title')}</h2>
      <TopUpReturn onCredited={() => void reload()} />

      {/* Every currency with money in it, not just the selected one. */}
      <table>
        <thead>
          <tr>
            <th scope="col">{t('wallet.currency')}</th>
            <th scope="col">{t('wallet.available')}</th>
            <th scope="col">{t('wallet.reserved')}</th>
          </tr>
        </thead>
        <tbody>
          {data.balances.map((b) => (
            <tr key={b.currency} aria-current={b.currency === active ? 'true' : undefined}>
              <th scope="row">
                {b.currency}
                {b.disabled && <small> {t('wallet.currencyRetired')}</small>}
              </th>
              <td>{formatMoney(b.available, b.currency, locale)}</td>
              <td>{b.held > 0 ? formatMoney(b.held, b.currency, locale) : '-'}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {/* The switcher only exists when there is something to switch. */}
      {data.currencies.length > 1 && (
        <label>
          {t('wallet.currency')}
          <select value={active} onChange={(e) => chooseCurrency(e.target.value)}>
            {data.currencies.map((c) => <option key={c} value={c}>{c}</option>)}
          </select>
        </label>
      )}

      <TopUpForm currency={active} limits={data.topUpLimits[active]} />
      <WalletHistory currency={active} />
    </section>
  );
}

function TopUpForm({ currency, limits }: { currency: string; limits?: { min: number; max: number } }) {
  const t = useWalletT();
  const locale = useWalletLocale();
  const [input, setInput] = useState('');
  const [error, setError] = useState<ApiError | null>(null);
  const [busy, setBusy] = useState(false);
  if (!limits) return null; // a retired currency cannot be topped up

  const amount = parseMajorToMinor(input, currency);
  const inRange = amount !== null && amount >= limits.min && amount <= limits.max;

  async function submit(e: React.FormEvent) {
    e.preventDefault();
    if (!inRange || busy) return;
    setBusy(true);
    setError(null);
    try {
      const res = await walletFetch('/api/wallet/top-up', {
        method: 'POST', headers: { 'content-type': 'application/json' }, body: JSON.stringify({ currency, amount }),
      });
      if (!res.ok) {
        setError(await readApiError(res));
        return;
      }
      const { url } = (await res.json()) as { url: string };
      window.location.assign(url); // leave busy=true: the page is navigating away
      return;
    } catch (err) {
      setError({ code: 'NETWORK', message: String(err) });
    }
    setBusy(false);
  }

  return (
    <form onSubmit={submit} noValidate>
      <label>
        {t('wallet.topUp.amount', { currency })}
        <input
          inputMode="decimal"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          aria-invalid={input !== '' && !inRange}
          aria-describedby="top-up-hint"
        />
      </label>
      <p id="top-up-hint">
        {t('wallet.topUp.range', { min: formatMoney(limits.min, currency, locale), max: formatMoney(limits.max, currency, locale) })}
      </p>
      <button type="submit" disabled={!inRange || busy} aria-busy={busy}>{t('wallet.topUp.submit')}</button>
      {/* The server's reason, with its numbers, not a generic failure message. */}
      {error && <p role="alert">{t(`wallet.errors.${error.code}`, flat(error.details))}</p>}
    </form>
  );
}

/** Reads ?topUp= after Stripe redirects back, and waits for the webhook to land. */
function TopUpReturn({ onCredited }: { onCredited: () => void }) {
  const t = useWalletT();
  const [status, setStatus] = useState<'none' | 'pending' | 'credited' | 'slow' | 'cancelled'>('none');

  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const outcome = params.get('topUp');
    const session = params.get('session');
    if (outcome === 'cancelled') return setStatus('cancelled');
    if (outcome !== 'success' || !session) return;
    setStatus('pending');
    let tries = 0;
    let timer: ReturnType<typeof setTimeout>;
    const poll = async () => {
      const res = await walletFetch(`/api/wallet/top-up/status?session=${encodeURIComponent(session)}`).catch(() => null);
      const body = res?.ok ? ((await res.json()) as { status: string }) : null;
      if (body?.status === 'credited') {
        setStatus('credited');
        onCredited();
      } else if (++tries < 10) {
        timer = setTimeout(poll, 2000);
      } else {
        setStatus('slow'); // paid, webhook late, the money will appear; say so
      }
    };
    void poll();
    return () => clearTimeout(timer);
    // Runs once per page load by design; onCredited only triggers a reload.
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  if (status === 'none') return null;
  return <p role="status" aria-live="polite">{t(`wallet.topUp.return.${status}`)}</p>;
}

function flat(details?: Record<string, unknown>): Record<string, string | number> | undefined {
  if (!details) return undefined;
  const out: Record<string, string | number> = {};
  for (const [k, v] of Object.entries(details)) if (typeof v === 'string' || typeof v === 'number') out[k] = v;
  return out;
}
```

**The preferred currency is read once.** `initialized` guards the effect so a reload after a top-up, a
payment or a focus refresh never switches the customer back to the stored preference while they are looking
at another currency. The choice is saved to the server, so the next device and the next session open on it.

**The return poll stops.** Ten tries at two seconds, then a "still processing" message. The webhook may be
late; the money is not lost, and the message says so.

## History

```tsx
// components/wallet/WalletHistory.tsx: history for one currency, newest first, paged.
'use client';
// Also used by the admin panel with the admin endpoint (which adds the actor column).
import { useWalletEntries } from '@/hooks/useWallet';
import { useWalletLocale, useWalletT } from '@/lib/wallet/host-client';
import { formatMoney } from '@/lib/wallet/money';

export function WalletHistory({ currency, endpoint, showActor = false }: {
  currency: string;
  endpoint?: string;
  showActor?: boolean;
}) {
  const t = useWalletT();
  const locale = useWalletLocale();
  const { entries, hasMore, status, error, loadMore } = useWalletEntries(currency, endpoint);

  return (
    <section aria-labelledby="wallet-history-title">
      <h3 id="wallet-history-title">{t('wallet.history.title')}</h3>
      {status === 'error' && error && <p role="alert">{t(`wallet.errors.${error.code}`)}</p>}
      {entries.length === 0 && status === 'idle' && <p>{t('wallet.history.empty')}</p>}
      {entries.length > 0 && (
        <table>
          <thead>
            <tr>
              <th scope="col">{t('wallet.history.date')}</th>
              <th scope="col">{t('wallet.history.description')}</th>
              {showActor && <th scope="col">{t('wallet.history.by')}</th>}
              <th scope="col">{t('wallet.history.amount')}</th>
              <th scope="col">{t('wallet.history.balanceAfter')}</th>
            </tr>
          </thead>
          <tbody>
            {entries.map((e) => (
              <tr key={e.id}>
                <td><time dateTime={e.createdAt}>{new Date(e.createdAt).toLocaleString(locale)}</time></td>
                <td>
                  {/* Kind label + the translated reason; never a stored sentence in one language. */}
                  {t(`wallet.kind.${e.kind}`)}: {t(e.reason.key, e.reason.params)}
                </td>
                {showActor && <td>{e.actor ? `${e.actor.type}:${e.actor.id}` : '-'}</td>}
                <td>
                  {/* HOLD/CAPTURE/RELEASE move reserved money; show the change to "available". */}
                  {e.availableDelta === 0 ? '-' : `${e.availableDelta > 0 ? '+' : '-'}${formatMoney(Math.abs(e.availableDelta), e.currency, locale)}`}
                </td>
                <td>{formatMoney(e.availableAfter, e.currency, locale)}</td>
              </tr>
            ))}
          </tbody>
        </table>
      )}
      {hasMore && (
        <button type="button" onClick={loadMore} disabled={status === 'loading'} aria-busy={status === 'loading'}>
          {t('wallet.history.more')}
        </button>
      )}
    </section>
  );
}
```

History rows show the change to **available**. A hold shows as money leaving available (it is reserved), a
capture shows no change to available (the reserved money is spent), a release shows money returning. The
reserved column in the panel is where the customer sees what is on hold.

## Strings

All copy is keys in the host's i18n system, in every locale it ships. `t(key, params)` interpolates the
parameters named here.

| Key | Params | Meaning |
|---|---|---|
| `wallet.title`, `wallet.loading`, `wallet.retry`, `wallet.cancel`, `wallet.back` | | chrome |
| `wallet.currency`, `wallet.available`, `wallet.reserved`, `wallet.currencyRetired` | | balance table |
| `wallet.topUp.amount` | `currency` | amount label |
| `wallet.topUp.range` | `min`, `max` (formatted) | limits hint |
| `wallet.topUp.submit` | | button |
| `wallet.topUp.productName` | | Stripe line item name (server side, `translate`) |
| `wallet.topUp.return.pending`, `.credited`, `.slow`, `.cancelled` | | return-from-Stripe status |
| `wallet.history.title`, `.empty`, `.date`, `.description`, `.by`, `.amount`, `.balanceAfter`, `.more` | | history table |
| `wallet.kind.TOP_UP`, `.SPEND`, `.REFUND`, `.ADJUSTMENT`, `.HOLD`, `.CAPTURE`, `.RELEASE` | | entry kind label |
| `wallet.reason.topUp` | | reason: card top-up |
| `wallet.reason.orderPaid`, `.orderHold`, `.orderReleased` | `orderRef` | reason: order payment steps |
| `wallet.reason.adjustment` | `note` | reason: staff correction, shows the note |
| `wallet.reason.eventTicket`, `.eventRefund` | | reason: bookable-events adapter |
| `wallet.reason.seed`, `.openingBalance` | | reason: seeded or imported balances ([operations.md](operations.md)) |
| `wallet.errors.<CODE>` | the error's `details` | one per code in the error contract ([api-routes.md](api-routes.md)), plus `NETWORK` and `UNAUTHORIZED` |
| `wallet.admin.*` | see [admin-ui.md](admin-ui.md) | staff panel |

Write `wallet.errors.INSUFFICIENT_FUNDS` to use `available` and `required`, and
`wallet.errors.AMOUNT_OUT_OF_RANGE` to use `min` and `max`. Those numbers arrive in minor units; format them
with `formatMoney` in the host's `t` wrapper, or pre-format in the component before interpolating.

## Where it lives

A `/wallet` page (or a tab on the account page) renders `<WalletPanel />`. The top-up return URLs in
`app/api/wallet/top-up/route.ts` point at `/wallet`; change both together. At checkout, the host's payment
step reads `GET /api/wallet` for the balance in the order's currency and offers the three choices of
[split-payment.md](split-payment.md), showing the split before the customer confirms.
