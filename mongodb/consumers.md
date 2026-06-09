# MongoDB — Consumers (who READS which collections, and how)

Scope: apps/repos that READ from the `tcg-rs` cluster. Writers are in `ingestion.md`.

## Reader apps at a glance

| App / repo | Driver | Reads DBs | Notes |
|---|---|---|---|
| Java platform (`trajectoirecap.platform.parent`) | Spring Data MongoDB (`MongoTemplate` × 5 beans) | all 6 except `tcg-app-data` | every compute/signals/export/integration module |
| Java simulator (`trajectoirecap.platform.simulator`) | Mongo Java driver, **hardcoded** `getDatabase("tcg-instrument")` / `getDatabase("tcg")` | `tcg-instrument`, `tcg` | prices + operational |
| TCG-software (Python) | **`motor>=3.0`** (`AsyncIOMotorClient`, async) | `tcg-instrument` (read), `tcg-app-data` (read+write) | desktop app; connects via SSM tunnel |
| `trajectoirecap.signals.board` (Python) | **none for Mongo** — `pymongo==4.10.1` declared in `requirements.txt` but never imported in `src/` | — | reads nothing from Mongo; all data via `signals.rest` HTTP API |
| `trajectoirecap.signals.rest` (Python) | **none for Mongo** — `pymongo`/`motor` declared in `requirements.txt` but never imported/called | — | serves precomputed signals from **Redis** (`redis://redis:6379/0`), not Mongo |

---

## TCG-software (Python) — read path detail

- **Driver/client:** `motor.motor_asyncio.AsyncIOMotorClient` (async). Read client in `tcg/data/_mongo/client.py` + `tcg/core/app.py`; write client in `tcg/persistence/_client.py`.
- **Reads `tcg-instrument`** via `tcg/data/_mongo/instruments.py` + `registry.py`. Query pattern:
  - Collection discovery → `db.list_collection_names()` → `CollectionRegistry` classifies by prefix (`FUT_`, `OPT_`, `INDEX`, `ETF`/`FUND`/`FOREX`).
  - Active read set = `INDEX`, `ETF`, `FUND`, `FOREX`, `FUT_*`. `OPT_*` discovered but **deferred** (not surfaced to UI yet).
  - OHLCV pulled from the `eodDatas.<PROVIDER>` array; projections **exclude** `intradayDatas` and `eodGreeks` (`instruments.py`).
  - Futures contracts: query `expiration != null`, filter by `expirationCycle`, sort ascending (`fetch_futures_contracts`).
- **Reads + writes `tcg-app-data.2026-app-data`** via `tcg/persistence/repository.py` (`WriteRepository`). The write client uses a **scoped Mongo user** (env `MONGO_APP_WRITE_URI` or assembled `MONGO_APP_WRITE_USER/PASSWORD`); server-side role restricts it to this single namespace (OperationFailure 13 outside it). `WriteRepository` binds the collection handle once and exposes no collection-name parameter — double isolation.
- Client tuning (write): `serverSelectionTimeoutMS=30000`, `connectTimeoutMS=60000`, `socketTimeoutMS=300000`, `maxPoolSize=3`, `tz_aware=True`.

## Java platform — consumers by collection (selected)

Reads are pervasive (compute reads positions/prices, signals read instrument series, etc.). Notable READ consumers:

| Reader module | Reads (DB.collection) | How |
|---|---|---|
| `compute.parent/var`, `stress.test`, `greeks.core` | `tcg` positions/portfolios; `tcg-instrument` prices | Spring Data repos |
| `compute.parent/risk.report.mail` | `tcg` positions/PnL/VaR/stress; `tcg-instrument` | Spring Data repos; renders+emails |
| `compute.parent/pnl.compute` | `tcg` positions/trades/imagine portfolios; `tcg-instrument` prices | Spring Data repos |
| `signals.parent/black.tail.signals.batch` | `tcg-instrument` (`ETF`,`INDEX`,`INDEX_EOD_DATA`,futures); `tcg` realtime; `tcg-signals.strategy` | autowired `Signals*Repository` set |
| `signals.parent/signals.trading` | `tcg`/`tcg-signals` **+ Postgres `dwh.data_live.signals_live`** | Spring Data (Mongo) + plain JDBC (dwh) |
| `signals.parent/crypto.signals` | `tcg` crypto positions; `tcg-instrument` crypto prices; `tcg.volatilitySurface` | Spring Data repos |
| `signals.parent/crypto.report` | `tcg.simulatorPortfolio`/`simulatorTrades` | Spring Data repos |
| `signals.parent/black.pearl.signals` | `tcg-instrument` (VIX/SPX series) | Spring Data repos |
| `export.parent/performance.attribution` | `tcg` Portfolio/ImaginePortfolio/`fundPnL`/`appConfig` | Spring Data repos |
| `export.parent/expiration.notification` | `tcg` positions/imagine; `tcg-instrument` (expiry); `tcg.instrumentCodeSettings` | Spring Data repos |
| `export.parent/signals.vix.flows` | `tcg-signals.signalsVixFlows` (`findByDateIntGreaterThan`) | Spring Data repos |
| `export.parent/position.export` | `tcg` positions/trades/imagine portfolios | Spring Data repos |
| `iqfeed` integration | `tcg.singleRealTimeConfig`, `tcg.futureRealTimeConfig` (which symbols to subscribe) | Spring Data repos (then streams) |
| Java auth/web | `tcg.user`, `tcg.refreshToken`, `tcg.appConfig`, `tcg.app` | Spring Data repos |
| `DataMappingCache` | `tcg.dataMapping`, `tcg.instrumentCodeSettings` | cached at startup |

> `dwh.data_live.signals_live` is Postgres, not Mongo — listed because `signals.trading`/`signals.report.export` read it alongside Mongo. See the Postgres docs.

## Query-pattern summary

| Stack | Access style |
|---|---|
| Java | Spring Data `MongoRepository` derived queries + `MongoTemplate` (`@Query`, criteria); 5 templates one per DB |
| Java simulator | Native Mongo Java driver, `getDatabase("…")` hardcoded |
| Python (TCG-software) | `motor` async; `find` with projections; collection-name discovery via `list_collection_names()` + prefix registry |
## TODO / UNKNOWN- IQFeed persistence sink (real-time cache vs a Mongo collection) — unconfirmed.
