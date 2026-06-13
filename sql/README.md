# SQL — the `dwh` Postgres warehouse (`tcg_instruments` schema)

AWS RDS **PostgreSQL** (`dwh` database, currently engine 12.22). The analytics warehouse. The `tcg_instruments` schema is the **system of record for EOD market time-series** — OHLCV bars and option greeks for every tradable instrument, backfilled from MongoDB `tcg-instrument`. It replaces Mongo's candle storage for analytical reads.

> **No live credentials in this doc.** Env-var names / placeholders only. Pull the actual host/user/password from the team secrets store and keep them in a local gitignored `.env`. A read-only consumer role **`tcg_read`** exists (CONNECT on `dwh`, USAGE+SELECT on `tcg_instruments` only) — ask for those credentials rather than using an admin login.

**Source of truth for the schema:** `ddl_tcg_instruments.sql` (the as-built DDL — applies front-to-back on a fresh PostgreSQL, matches production). The design rationale lives in `ddl_simple.sql`. Both are in the `mongo-dwh-backfill` / `dwh-timeseries-data-model` task outputs, not this repo.

---

## The schema at a glance — FOUR logical relations

| Relation | Kind | What it holds |
|---|---|---|
| `tcg_instruments.dim_instrument` | table | One row per tradable instrument, **every asset class** (index, etf, fund, forex, future, option) in one table. Class-specific columns are nullable; shape CHECKs keep each row well-formed. |
| `tcg_instruments.fact_price_eod` | partitioned table | One OHLCV bar per `(instrument_id, trade_date)`, any asset class. |
| `tcg_instruments.fact_option_greeks` | partitioned table | One greeks row per `(option instrument_id, trade_date)`. |
| `tcg_instruments.v_option_chain` | view | The options-chain query, pre-joined (greeks + the option's own quotes, per contract per date). The only object that is a view. |

There is no `fact_vol_surface` (Amendment 3 — see below). The whole thing runs on bare PostgreSQL: no extensions, no jobs.

### Partition rule — ALWAYS query the parent tables
`fact_price_eod` and `fact_option_greeks` are **range-partitioned yearly** (`*_1980` … `*_2050`). **Never query a `*_YYYY` partition directly** — query the parent (`fact_price_eod` / `fact_option_greeks`) and filter on `trade_date`; PostgreSQL prunes to the right year automatically. A `trade_date` outside 1980–2050 fails the INSERT by design (no silent catch-all).

---

## Key conventions a consumer needs

### 1. The canonical query: join on `instrument_id`, filter on `symbol`
`symbol` is the durable Mongo `_id` (e.g. `IND_SP_500`, `ETF_SPY`, `FUT_SP_500_EMINI_20241220`). It is **UNIQUE for non-options** (partial unique index). Options also need `expiration_cycle` to be unique — query options through `v_option_chain`, not by symbol alone.

```sql
-- a non-option's history (prunes to the years in range):
SELECT f.trade_date, f.close
FROM tcg_instruments.fact_price_eod f
JOIN tcg_instruments.dim_instrument i USING (instrument_id)
WHERE i.symbol = 'IND_SP_500'
  AND f.trade_date BETWEEN '2015-01-01' AND '2024-12-31'
ORDER BY f.trade_date;
```

### 2. `root_symbol` = the roll/family key; NULL = standalone
`root_symbol` is the roll key for instruments that belong to a contract family, and **NULL by design for standalone instruments** (indexes, ETFs, FX spot, funds) — NULL means "this thing does not roll", which is itself information.

- **Futures:** `root_symbol` is the shared root (e.g. `IND_SP_500` for the E-mini ES family). Fetch a family in roll order: `WHERE root_symbol = 'IND_SP_500' AND asset_class = 'future' ORDER BY expiration, trade_date`.
- **Options (Amendment 2):** `root_symbol` holds the source `rootUnderlying` verbatim (`GOLD`, `BTC`, `ETH`, `IND_NASDAQ_100`, …).

**Canonical family selector** — to browse families uniformly across asset classes, group on `COALESCE(root_symbol, symbol)` together with `asset_class`:
```sql
SELECT COALESCE(root_symbol, symbol) AS family, asset_class, count(*)
FROM tcg_instruments.dim_instrument
GROUP BY 1, 2 ORDER BY 1;
```
A family *including* its spot needs the spot symbol explicitly (e.g. `root_symbol = 'BTC' OR symbol = 'BTC_USD'`) — spot naming (`BTC_USD`) differs from the contract root (`BTC`) in the source.

### 3. Option underlyings: `underlying_id` is NULL for 8 of 10 roots → slice by `root_symbol`
564,293 options carry a **NULL `underlying_id`**: their source `rootUnderlying` is a roll-key root (`GOLD`, `BTC`, `ETH`, `EURUSD`, `JPYUSD`, `IND_NASDAQ_100`, `T_NOTE_10_Y`, `T_BOND`) that does not resolve to an instrument row. Only the options with `root_symbol` **`IND_SP_500`** (S&P 500 options, 422,447) and **`IND_VIX`** (VIX options, 59,700) resolve to real INDEX rows via `underlying_id`. So in `v_option_chain`, `underlying_symbol` is NULL for the eight non-resolving roots — **slice their chains by `dim_instrument.root_symbol` (joined on `option_instrument_id`), not by `underlying_symbol`**:
```sql
-- ETH option chain on a date (underlying_id is NULL → filter root_symbol):
SELECT v.*
FROM tcg_instruments.v_option_chain v
JOIN tcg_instruments.dim_instrument o ON o.instrument_id = v.option_instrument_id
WHERE o.root_symbol = 'ETH' AND v.trade_date = '2024-03-15';
```
Note: index options (SPX/NDX) are listed on the **E-mini future**, so their symbols look like `OPT_FUT_SP_500_EMINI_*` and the greeks feed often leaves `underlying_price` NULL — join the underlying instrument's own `fact_price_eod` bar if you need a spot.

### 4. `days_to_expiry` is NULL for the IVOLATILITY feed → derive it
`fact_option_greeks.days_to_expiry` preserves the feed's own DTE, which means it is **NULL for feeds that omit it — including IVOLATILITY, the source for most index/future/rate options**. `WHERE days_to_expiry BETWEEN ...` silently returns nothing for those. When NULL, derive: `dim_instrument.expiration - trade_date`.

### 5. `adj_close` is sparse → fall back to `close`
`adj_close` (split/dividend-adjusted close, for long-horizon total-return math) is populated **only where the source feed supplies it — in practice the YAHOO feed**: ~38% of YAHOO index bars, ~68% of YAHOO ETF bars, 100% of YAHOO FX bars. It is **always NULL** for the COINGECKO (crypto FX), BLOOMBERG (the fund), and IVOLATILITY (futures/options) feeds, and sparse even within YAHOO. It is a **provider artifact, not a corporate-actions flag** — do not assume an instrument "has" or "lacks" adjustments from it. Rule: use `adj_close` **when present, else fall back to `close`**.

### 6. Deribit crypto premiums are in coin, not USD
DERIBIT crypto-option premiums (OPT_BTC / OPT_ETH) in `fact_price_eod.close` / `v_option_chain.option_close` are quoted in the **underlying coin**, not USD. Multiply by the coin/USD spot (available as the forex pair, e.g. `ETH_USD`) to dollarize before any P&L or moneyness math.

### 7. Expect NULLs in `v_option_chain`
The view is a UNION ALL of greeks-bearing and quotes-only days. A row can have greeks with NULL quotes (no bar that day) or quotes with NULL greeks (no greeks that day — e.g. Deribit ETH options, which are quotes-only). Handle NULLs before arithmetic. NaN never appears — every numeric column is NaN-guarded by a CHECK.

### Amendment 3 — no vol-surface table (yet)
Vol surfaces were deferred out of the backfill (source is `tcg.volatilitySurface`, outside `tcg-instrument`) and the empty `fact_vol_surface` table was dropped. There is **no surface table today**; it returns with its own table when the surfaces phase lands. Its definition is preserved in `ddl_simple.sql`.

---

## Connecting

The RDS instance is reachable the same way as Mongo — **port-forward through the EC2 bastion via AWS SSM Session Manager**, then point `psql`/`psycopg` at `localhost`. (Same pattern as `mongodb/connecting.md`: `aws ssm start-session --document-name AWS-StartPortForwardingSessionToRemoteHost`, with the RDS host/port in `--parameters`.)

Once the tunnel is up, load connection settings from a local `.env` into standard `PG*` environment variables (`PGHOST=localhost`, `PGPORT`, `PGDATABASE=dwh`, `PGUSER`, `PGPASSWORD`) so neither `psql` nor `psycopg` ever sees a credential on the command line:

```bash
# .env (gitignored) — values from the team secrets store, never committed:
PGHOST=localhost
PGPORT=...            # local forwarded port
PGDATABASE=dwh
PGUSER=tcg_read       # the read-only consumer role
PGPASSWORD=...        # fill locally
```
```bash
set -a; . ./.env; set +a       # export PG* into the environment
psql -c "SELECT count(*) FROM tcg_instruments.dim_instrument;"
```

Use the read-only **`tcg_read`** role for consumer queries — it has only USAGE+SELECT on `tcg_instruments`.

> ⚠ Plaintext production credentials are committed in some source repos (security TODO — rotation still open). Do NOT copy them here or into code; pull live creds from the secrets store into your local `.env`.

---

## Where the truth lives
- **`ddl_tcg_instruments.sql`** — the as-built schema (matches prod). Read its table/column COMMENTs: they document every join pattern, NULL semantic, and gotcha summarized above, plus nine worked usage examples (canonical bar query, cross-section, continuous-futures roll with overlap margin, option-chain slices, option-leg selection by delta/strike/DTE, derived trading calendar).
- **`ddl_simple.sql`** — the design DDL + rationale (including the deferred `fact_vol_surface`).
