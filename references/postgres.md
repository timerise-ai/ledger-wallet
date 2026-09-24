# Postgres store

`WalletStore` on Postgres 13 or later, Supabase included. The store is written against a two-method client
interface that `pg.Pool`, `@neondatabase/serverless`'s `Pool` and similar drivers satisfy, so the host keeps
its driver. An ORM host (Drizzle, Prisma, Kysely) can still use it: pass the underlying pool, or implement
`WalletStore` in the ORM using the same statements.

## Schema

```sql
-- db/wallet.sql: ledger wallet schema (Postgres 13+, Supabase included).
-- Amounts are bigint minor units. Balances are a materialized cache of the entries;
-- the CHECK constraints make an overdraft impossible even for code that bypasses the service.

create table wallets (
  tenant_id          text        not null,
  customer_id        text        not null,
  preferred_currency char(3)     not null check (preferred_currency ~ '^[A-Z]{3}$'),
  created_at         timestamptz not null default now(),
  updated_at         timestamptz not null default now(),
  primary key (tenant_id, customer_id)
);

create table wallet_balances (
  tenant_id   text        not null,
  customer_id text        not null,
  currency    char(3)     not null check (currency ~ '^[A-Z]{3}$'),
  available   bigint      not null default 0 check (available >= 0),
  held        bigint      not null default 0 check (held >= 0),
  -- = created_at of the latest entry; the engine keeps it strictly increasing per balance
  last_movement_at timestamptz,
  primary key (tenant_id, customer_id, currency),
  foreign key (tenant_id, customer_id) references wallets (tenant_id, customer_id)
);

create table wallet_entries (
  id              text        primary key,
  tenant_id       text        not null,
  customer_id     text        not null,
  ref             text        not null,
  kind            text        not null check (kind in ('TOP_UP','SPEND','REFUND','ADJUSTMENT','HOLD','CAPTURE','RELEASE')),
  currency        char(3)     not null,
  amount          bigint      not null check (amount > 0),
  available_delta bigint      not null,
  held_delta      bigint      not null,
  available_after bigint      not null check (available_after >= 0),
  held_after      bigint      not null check (held_after >= 0),
  hold_ref        text,
  order_ref       text,
  external_ref    text,
  reason_key      text        not null,
  reason_params   jsonb       not null default '{}',
  actor_type      text        not null check (actor_type in ('customer','staff','system')),
  actor_id        text        not null,
  created_at      timestamptz not null,
  unique (tenant_id, ref),   -- the idempotency guarantee
  foreign key (tenant_id, customer_id) references wallets (tenant_id, customer_id)
);

create index wallet_entries_history on wallet_entries (tenant_id, customer_id, created_at desc, id desc);
create index wallet_entries_history_by_currency
  on wallet_entries (tenant_id, customer_id, currency, created_at desc, id desc);
create index wallet_entries_order on wallet_entries (tenant_id, order_ref) where order_ref is not null;

create table wallet_holds (
  tenant_id    text        not null,
  hold_ref     text        not null,
  customer_id  text        not null,
  currency     char(3)     not null,
  amount       bigint      not null check (amount > 0),
  status       text        not null check (status in ('OPEN','CAPTURED','RELEASED')),
  order_ref    text,
  external_ref text,
  created_at   timestamptz not null,
  settled_at   timestamptz,
  primary key (tenant_id, hold_ref)
);
create index wallet_holds_open on wallet_holds (created_at) where status = 'OPEN';

-- Entries are history: never edited, never deleted. A correction is a new ADJUSTMENT.
create function wallet_entries_append_only() returns trigger language plpgsql as $$
begin
  raise exception 'wallet_entries is append-only';
end $$;
create trigger wallet_entries_no_update_delete
  before update or delete on wallet_entries
  for each row execute function wallet_entries_append_only();

-- Supabase / any database reachable with client credentials: no client access at all.
-- The server uses a role that bypasses RLS (service role); customers read through the API.
alter table wallets         enable row level security;
alter table wallet_balances enable row level security;
alter table wallet_entries  enable row level security;
alter table wallet_holds    enable row level security;
```

| Constraint | What it holds |
|---|---|
| `available >= 0`, `held >= 0` | no overdraft, even from code that bypasses the service |
| `unique (tenant_id, ref)` | idempotency: a second insert of a ref fails, and the store retries into a replay |
| append-only trigger | history is never edited; a correction is a new `ADJUSTMENT` |
| partial index on open holds | the sweeper's query stays cheap as settled holds accumulate |
| RLS enabled, no policies | nothing is reachable with client credentials; the server uses a role that bypasses RLS |

Run it as a migration in the host's own tool (Drizzle Kit, Prisma migrate, Supabase migrations, plain SQL).

## The store

```ts
// lib/wallet/postgres-store.ts: WalletStore on Postgres, over the smallest client
// interface that node-postgres (`pg.Pool`), Neon's serverless Pool and similar satisfy.
// Schema: db/wallet.sql.
import type { CurrencyCode } from './money';
import type { EntryQuery, WalletStore, WalletTx } from './ports';
import type { Actor, BalanceState, WalletEntry, WalletHold } from './types';

export interface SqlClient {
  query<R = Record<string, unknown>>(text: string, params?: unknown[]): Promise<{ rows: R[]; rowCount?: number | null }>;
}
export interface SqlPool extends SqlClient {
  connect(): Promise<SqlClient & { release(err?: Error | boolean): void }>;
}

// node-postgres returns bigint (int8) and SUM() as strings. Every amount goes through this.
const num = (v: unknown): number => {
  const n = typeof v === 'number' ? v : Number(v ?? 0);
  if (!Number.isSafeInteger(n)) throw new Error(`Amount out of safe range: ${String(v)}`);
  return n;
};
const date = (v: unknown): Date => (v instanceof Date ? v : new Date(String(v)));

type Row = Record<string, unknown>;

function entryFromRow(r: Row): WalletEntry {
  return {
    id: String(r.id), tenantId: String(r.tenant_id), customerId: String(r.customer_id), ref: String(r.ref),
    kind: r.kind as WalletEntry['kind'], currency: String(r.currency), amount: num(r.amount),
    availableDelta: num(r.available_delta), heldDelta: num(r.held_delta),
    availableAfter: num(r.available_after), heldAfter: num(r.held_after),
    holdRef: (r.hold_ref as string | null) ?? null, orderRef: (r.order_ref as string | null) ?? null,
    externalRef: (r.external_ref as string | null) ?? null,
    reason: { key: String(r.reason_key), params: (r.reason_params as Record<string, string>) ?? {} },
    actor: { type: r.actor_type, id: r.actor_id } as Actor,
    createdAt: date(r.created_at),
  };
}

function holdFromRow(r: Row): WalletHold {
  return {
    tenantId: String(r.tenant_id), customerId: String(r.customer_id), holdRef: String(r.hold_ref),
    currency: String(r.currency), amount: num(r.amount), status: r.status as WalletHold['status'],
    orderRef: (r.order_ref as string | null) ?? null, externalRef: (r.external_ref as string | null) ?? null,
    createdAt: date(r.created_at), settledAt: r.settled_at ? date(r.settled_at) : null,
  };
}

/** Wraps a client already inside BEGIN, so a movement commits with the caller's order write. */
export function postgresWalletTx(client: SqlClient, defaultCurrency: CurrencyCode): WalletTx {
  return {
    async readBalance(tenantId, customerId, currency): Promise<BalanceState> {
      await client.query(
        `insert into wallets (tenant_id, customer_id, preferred_currency) values ($1, $2, $3)
         on conflict do nothing`,
        [tenantId, customerId, defaultCurrency],
      );
      await client.query(
        `insert into wallet_balances (tenant_id, customer_id, currency) values ($1, $2, $3)
         on conflict do nothing`,
        [tenantId, customerId, currency],
      );
      // The row lock serializes every movement of this customer's currency.
      const { rows } = await client.query(
        `select available, held, last_movement_at from wallet_balances
         where tenant_id = $1 and customer_id = $2 and currency = $3 for update`,
        [tenantId, customerId, currency],
      );
      const row = rows[0];
      return {
        available: num(row?.available),
        held: num(row?.held),
        lastMovementAt: row?.last_movement_at ? date(row.last_movement_at) : null,
      };
    },
    async readEntry(tenantId, ref) {
      const { rows } = await client.query(`select * from wallet_entries where tenant_id = $1 and ref = $2`, [tenantId, ref]);
      return rows[0] ? entryFromRow(rows[0]) : null;
    },
    async readHold(tenantId, holdRef) {
      // FOR UPDATE: a concurrent capture and release queue here, and the second one
      // re-reads the committed status instead of acting on a stale OPEN.
      const { rows } = await client.query(
        `select * from wallet_holds where tenant_id = $1 and hold_ref = $2 for update`, [tenantId, holdRef],
      );
      return rows[0] ? holdFromRow(rows[0]) : null;
    },
    async writeBalance(tenantId, customerId, currency, next, at) {
      await client.query(
        `update wallet_balances set available = $4, held = $5, last_movement_at = $6
         where tenant_id = $1 and customer_id = $2 and currency = $3`,
        [tenantId, customerId, currency, next.available, next.held, at],
      );
    },
    async createEntry(e) {
      await client.query(
        `insert into wallet_entries (id, tenant_id, customer_id, ref, kind, currency, amount, available_delta,
           held_delta, available_after, held_after, hold_ref, order_ref, external_ref, reason_key, reason_params,
           actor_type, actor_id, created_at)
         values ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10,$11,$12,$13,$14,$15,$16,$17,$18,$19)`,
        [e.id, e.tenantId, e.customerId, e.ref, e.kind, e.currency, e.amount, e.availableDelta, e.heldDelta,
          e.availableAfter, e.heldAfter, e.holdRef, e.orderRef, e.externalRef, e.reason.key,
          JSON.stringify(e.reason.params ?? {}), e.actor.type, e.actor.id, e.createdAt],
      );
    },
    async writeHold(h) {
      await client.query(
        `insert into wallet_holds (tenant_id, hold_ref, customer_id, currency, amount, status, order_ref,
           external_ref, created_at, settled_at)
         values ($1,$2,$3,$4,$5,$6,$7,$8,$9,$10)
         on conflict (tenant_id, hold_ref) do update set status = excluded.status, settled_at = excluded.settled_at`,
        [h.tenantId, h.holdRef, h.customerId, h.currency, h.amount, h.status, h.orderRef, h.externalRef,
          h.createdAt, h.settledAt],
      );
    },
  };
}

// 23505 unique_violation: two first-time requests with one ref raced past readEntry;
// the loser retries and takes the replay path. 40001 / 40P01: serialization / deadlock.
const RETRYABLE = new Set(['23505', '40001', '40P01']);

export function createPostgresWalletStore(pool: SqlPool, defaultCurrency: CurrencyCode): WalletStore {
  return {
    async runTransaction(fn) {
      for (let attempt = 0; ; attempt++) {
        const client = await pool.connect();
        try {
          await client.query('begin');
          const result = await fn(postgresWalletTx(client, defaultCurrency));
          await client.query('commit');
          client.release();
          return result;
        } catch (err) {
          await client.query('rollback').catch(() => undefined);
          client.release();
          const code = (err as { code?: string }).code;
          if (attempt < 3 && code && RETRYABLE.has(code)) continue;
          throw err;
        }
      }
    },

    async getWallet(tenantId, customerId) {
      const { rows: w } = await pool.query(
        `select preferred_currency from wallets where tenant_id = $1 and customer_id = $2`, [tenantId, customerId],
      );
      if (!w[0]) return null;
      const { rows } = await pool.query(
        `select currency, available, held from wallet_balances
         where tenant_id = $1 and customer_id = $2 order by currency`,
        [tenantId, customerId],
      );
      return {
        tenantId,
        customerId,
        preferredCurrency: String(w[0].preferred_currency),
        balances: rows.map((r) => ({ currency: String(r.currency), available: num(r.available), held: num(r.held) })),
      };
    },

    async setPreferredCurrency(tenantId, customerId, currency) {
      await pool.query(
        `insert into wallets (tenant_id, customer_id, preferred_currency) values ($1, $2, $3)
         on conflict (tenant_id, customer_id) do update set preferred_currency = excluded.preferred_currency, updated_at = now()`,
        [tenantId, customerId, currency],
      );
    },

    async listEntries(q: EntryQuery) {
      const params: unknown[] = [q.tenantId, q.customerId];
      let where = `tenant_id = $1 and customer_id = $2`;
      if (q.currency) {
        params.push(q.currency);
        where += ` and currency = $${params.length}`;
      }
      if (q.cursor) {
        // Keyset cursor: "<ISO createdAt>|<id>". Stable under inserts, unlike OFFSET.
        const [at, id] = Buffer.from(q.cursor, 'base64url').toString().split('|');
        if (at && id) {
          params.push(new Date(at), id);
          where += ` and (created_at, id) < ($${params.length - 1}, $${params.length})`;
        }
      }
      params.push(q.limit + 1);
      const { rows } = await pool.query(
        `select * from wallet_entries where ${where} order by created_at desc, id desc limit $${params.length}`, params,
      );
      const entries = rows.slice(0, q.limit).map(entryFromRow);
      const last = entries[entries.length - 1];
      return {
        entries,
        nextCursor: rows.length > q.limit && last
          ? Buffer.from(`${last.createdAt.toISOString()}|${last.id}`).toString('base64url')
          : null,
      };
    },

    async listOpenHolds(olderThan, limit) {
      const { rows } = await pool.query(
        `select * from wallet_holds where status = 'OPEN' and created_at < $1 order by created_at limit $2`,
        [olderThan, limit],
      );
      return rows.map(holdFromRow);
    },

    async sumEntries(tenantId, customerId) {
      const { rows } = await pool.query(
        `select currency, sum(available_delta) as available, sum(held_delta) as held
         from wallet_entries where tenant_id = $1 and customer_id = $2 group by currency`,
        [tenantId, customerId],
      );
      return rows.map((r) => ({ currency: String(r.currency), available: num(r.available), held: num(r.held) }));
    },
  };
}
```

## How concurrency resolves

| Race | What happens |
|---|---|
| Two first-time posts of one ref | both lock the balance row in turn; the second sees the committed entry and replays |
| One ref reused for a different customer (a caller bug) | the second insert raises 23505; the retry reads the entry and answers `REF_CONFLICT` |
| Two spends that together exceed the balance | serialized by `FOR UPDATE`; the second sees the lower balance and gets `INSUFFICIENT_FUNDS` |
| Capture and release of one hold at once | `readHold ... FOR UPDATE` queues the second, which then reads the settled status |
| Deadlock or serialization failure | 40P01 / 40001 are retried up to three times |

The concurrent-duplicate, overdraw and capture-versus-release races run against a real Postgres in
`test/store-conformance.test.ts` when `WALLET_PG_URL` is set ([testing-stores.md](testing-stores.md)).

## Driver traps

- **`bigint` comes back as a string** from node-postgres, and so does `sum(...)`. Every amount goes through
  `num()`, which also refuses values past `Number.MAX_SAFE_INTEGER` instead of silently losing precision.
- **Pooled connections.** A transaction must use one client from `pool.connect()`, never `pool.query()`,
  which may pick a different connection for each statement. The store does this; code sharing the
  transaction must too.
- **Transaction-mode poolers** (Supabase's pooler on port 6543, PgBouncer in transaction mode) are fine: every
  statement of a transaction runs on the connection that began it.
- **Keyset pagination.** The cursor is `createdAt|id`, base64url. Offsets would skip or repeat rows as new
  entries arrive at the top of the list.

## Sharing the caller's transaction

`postgresWalletTx(client, DEFAULT_CURRENCY)` wraps a client already inside `begin`. The wallet movement and
the host's order insert then commit or roll back together. Postgres has no read-before-write rule, but keep
the balance lock short: call `postInTx` late in the transaction, after slow work. Worked example and test:
[integration.md](integration.md).
