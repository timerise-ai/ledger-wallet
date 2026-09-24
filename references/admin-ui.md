# Staff UI

The staff view answers the questions support gets: what does this customer have, in which currencies, where
did it come from, who changed it and why, and does it add up. It lives on the host's customer detail page.

| Element | Rule |
|---|---|
| Balances | every currency, available and reserved; delisted currencies marked |
| Reconciliation flag | an alert when the stored balance differs from the sum of entries (see [operations.md](operations.md)) |
| History | the customer's history plus a **by** column: customer, staff id, or system module |
| Adjust | admins only; credit or debit, amount, a required note; a review step that shows the balance before and after; one request id per dialog opening |
| Errors | the server's code with its numbers; an adjustment that would overdraw is refused in the dialog before it is sent, and by the server if it is sent anyway |

## The panel

```tsx
// components/admin/WalletAdminPanel.tsx: staff view of one customer's wallet:
'use client';
// balances in every currency, the reconciliation flag, attributed history, adjustments.
import { useCallback, useEffect, useState } from 'react';
import { readApiError, type ApiError } from '@/hooks/useWallet';
import { useWalletLocale, useWalletT, walletFetch } from '@/lib/wallet/host-client';
import { formatMoney, parseMajorToMinor } from '@/lib/wallet/money';
import type { BalanceRow } from '@/lib/wallet/wallet';
import { WalletHistory } from '../wallet/WalletHistory';

interface AdminView {
  currencies: string[];
  balances: BalanceRow[];
  reconciled: boolean;
}

export function WalletAdminPanel({ customerId, canAdjust }: { customerId: string; canAdjust: boolean }) {
  const t = useWalletT();
  const locale = useWalletLocale();
  const base = `/api/admin/wallets/${encodeURIComponent(customerId)}`;
  const [view, setView] = useState<AdminView | null>(null);
  const [error, setError] = useState<ApiError | null>(null);
  const [currency, setCurrency] = useState<string | null>(null);
  const [adjusting, setAdjusting] = useState(false);
  const [historyKey, setHistoryKey] = useState(0);

  const load = useCallback(async () => {
    const res = await walletFetch(`${base}?limit=1`);
    if (!res.ok) return setError(await readApiError(res));
    const data = (await res.json()) as AdminView;
    setView(data);
    setCurrency((c) => c ?? data.balances[0]?.currency ?? null);
  }, [base]);

  useEffect(() => {
    void load();
  }, [load]);

  if (error) return <p role="alert">{t(`wallet.errors.${error.code}`)}</p>;
  if (!view || !currency) return <p aria-busy="true">{t('wallet.loading')}</p>;
  const selected = view.balances.find((b) => b.currency === currency);

  return (
    <section aria-labelledby="admin-wallet-title">
      <h2 id="admin-wallet-title">{t('wallet.admin.title')}</h2>
      {!view.reconciled && <p role="alert">{t('wallet.admin.reconcileMismatch')}</p>}
      <table>
        <thead>
          <tr>
            <th scope="col">{t('wallet.currency')}</th>
            <th scope="col">{t('wallet.available')}</th>
            <th scope="col">{t('wallet.reserved')}</th>
          </tr>
        </thead>
        <tbody>
          {view.balances.map((b) => (
            <tr key={b.currency} onClick={() => setCurrency(b.currency)} aria-selected={b.currency === currency}>
              <th scope="row">{b.currency}{b.disabled && <small> {t('wallet.currencyRetired')}</small>}</th>
              <td>{formatMoney(b.available, b.currency, locale)}</td>
              <td>{formatMoney(b.held, b.currency, locale)}</td>
            </tr>
          ))}
        </tbody>
      </table>

      {canAdjust && (
        <button type="button" onClick={() => setAdjusting(true)}>{t('wallet.admin.adjust')}</button>
      )}
      {adjusting && selected && (
        <AdjustDialog
          endpoint={`${base}/adjust`}
          balance={selected}
          onClose={(changed) => {
            setAdjusting(false);
            if (changed) {
              void load();
              setHistoryKey((k) => k + 1);
            }
          }}
        />
      )}
      <WalletHistory key={`${currency}:${historyKey}`} currency={currency} endpoint={base} showActor />
    </section>
  );
}

function AdjustDialog({ endpoint, balance, onClose }: {
  endpoint: string;
  balance: BalanceRow;
  onClose: (changed: boolean) => void;
}) {
  const t = useWalletT();
  const locale = useWalletLocale();
  // One id per dialog opening: a double click or a retry after a timeout is a replay, not a second credit.
  const [requestId] = useState(() => crypto.randomUUID());
  const [direction, setDirection] = useState<'credit' | 'debit'>('credit');
  const [input, setInput] = useState('');
  const [note, setNote] = useState('');
  const [confirming, setConfirming] = useState(false);
  const [busy, setBusy] = useState(false);
  const [error, setError] = useState<ApiError | null>(null);

  const magnitude = parseMajorToMinor(input, balance.currency);
  const delta = magnitude ? (direction === 'credit' ? magnitude : -magnitude) : null;
  const after = delta === null ? null : balance.available + delta;
  const valid = delta !== null && after !== null && after >= 0 && note.trim().length >= 3;

  async function apply() {
    setBusy(true);
    setError(null);
    const res = await walletFetch(endpoint, {
      method: 'POST',
      headers: { 'content-type': 'application/json' },
      body: JSON.stringify({ currency: balance.currency, delta, note: note.trim(), requestId }),
    }).catch(() => null);
    setBusy(false);
    if (res?.ok) return onClose(true);
    setError(res ? await readApiError(res) : { code: 'NETWORK', message: '' });
    setConfirming(false);
  }

  return (
    <dialog open aria-labelledby="adjust-title">
      <h3 id="adjust-title">{t('wallet.admin.adjustTitle', { currency: balance.currency })}</h3>
      <fieldset disabled={confirming || busy}>
        <label><input type="radio" checked={direction === 'credit'} onChange={() => setDirection('credit')} /> {t('wallet.admin.credit')}</label>
        <label><input type="radio" checked={direction === 'debit'} onChange={() => setDirection('debit')} /> {t('wallet.admin.debit')}</label>
        <label>{t('wallet.admin.amount')}<input inputMode="decimal" value={input} onChange={(e) => setInput(e.target.value)} /></label>
        <label>{t('wallet.admin.note')}<textarea required minLength={3} maxLength={500} value={note} onChange={(e) => setNote(e.target.value)} /></label>
      </fieldset>
      {after !== null && (
        // Blast radius before commit: the balance this will leave, with real numbers.
        <p aria-live="polite">
          {t('wallet.admin.preview', {
            before: formatMoney(balance.available, balance.currency, locale),
            after: formatMoney(Math.max(after, 0), balance.currency, locale),
          })}
          {after < 0 && <strong role="alert"> {t('wallet.errors.INSUFFICIENT_FUNDS')}</strong>}
        </p>
      )}
      {error && <p role="alert">{t(`wallet.errors.${error.code}`)}</p>}
      {!confirming ? (
        <>
          <button type="button" onClick={() => onClose(false)}>{t('wallet.cancel')}</button>
          <button type="button" disabled={!valid} onClick={() => setConfirming(true)}>{t('wallet.admin.review')}</button>
        </>
      ) : (
        <>
          <button type="button" onClick={() => setConfirming(false)} disabled={busy}>{t('wallet.back')}</button>
          <button type="button" onClick={() => void apply()} disabled={busy} aria-busy={busy}>{t('wallet.admin.confirm')}</button>
        </>
      )}
    </dialog>
  );
}
```

## Design decisions

- **The note is required and visible to the customer.** It becomes the entry's reason parameter, so the
  customer's history shows "Adjustment: goodwill credit for the cancelled class" rather than an unexplained
  number. Staff write it knowing that.
- **Debits are adjustments too.** A correction in either direction uses the same dialog, the same note, the
  same attribution. There is no separate "remove money" path to audit.
- **The review step shows the result**, not just the delta: "Balance goes from $40.00 to $15.00". A misplaced
  decimal is caught by the person who typed it.
- **One request id per opening.** `crypto.randomUUID()` runs once in the dialog's state initializer. Two
  clicks on confirm, or a retry after a network timeout, post the same `adjust:<id>` ref and replay.
- **Reading is broader than writing.** Any staff role can see the wallet; only `admin` can adjust. Map the
  role names to the host's.

## Strings

| Key | Params |
|---|---|
| `wallet.admin.title`, `.adjust`, `.credit`, `.debit`, `.amount`, `.note`, `.review`, `.confirm` | |
| `wallet.admin.adjustTitle` | `currency` |
| `wallet.admin.preview` | `before`, `after` (formatted) |
| `wallet.admin.reconcileMismatch` | |

The shared keys (`wallet.currency`, `wallet.history.*`, `wallet.errors.*`) are in [ui.md](ui.md).

## Extensions

Not in the templates; each is a small addition on top of what is there.

| Extension | Shape |
|---|---|
| Customer list column | the balance in the tenant's default currency, and a count of other currencies held, so a list never shows 0 for a customer who holds only EUR |
| Filter by kind | pass `kind` through `listEntries` and add it to the store query and index |
| Export | stream `listEntries` pages to CSV with `availableAfter` per row |
| Open holds view | `listOpenHolds` scoped to the tenant, with the Checkout session linked to the Stripe Dashboard |
| Refund from the order page | a button that calls `refundOrder` with the order's recorded amounts, showing both parts before confirming |
