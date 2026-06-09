# MongoDB — Structure (databases, collections, schemas)

Cluster: replica set `tcg-rs` @ `10.0.5.10:27017` (private subnet, reach via SSM tunnel — see `connecting.md`).
One physical cluster, **6 logical databases**. 5 are owned by the legacy Java platform (`trajectoirecap.platform.parent`); 1 (`tcg-app-data`) is the TCG-software Python write store.

Java DB→config keys: `core.parent/data/src/main/resources/mongodb-config.properties`
(`mongodb.uri/database`, `mongodb.instrument.*`, `mongodb.raw.*`, `mongodb.report.*`, `mongodb.signals.*`).
⚠ **5 plaintext admin connection URIs (with embedded creds) are committed in that file — SECURITY TODO. Values not reproduced here.**

## Databases

| DB | Purpose | Written by | Read by |
|---|---|---|---|
| `tcg-instrument` | Market/instrument data: EOD OHLCV series, futures contracts, options chains, indices, ETF/FUND/FOREX, rates, intraday | Java `repo.instrument.*` | Java + simulator; **TCG-software Python (read-only)** |
| `tcg` | Operational app DB: positions, PnL, trades, margins, risk history, simulator portfolios, users, configs, vol surfaces, real-time configs | Java `repo.data.*` (default `mongoTemplate`) | Java + simulator |
| `tcg-raw` | Raw vendor feeds pre-normalization (iVolatility, Sudrania, Vadim Prelude/Galaxy) | Java `repo.raw.*` | Java |
| `tcg-report` | Reconciliation reports (EDF-vs-Imagine PnL) | Java `repo.report.*` | Java |
| `tcg-signals` | Computed signals & monitor series + strategy regime state | Java `repo.signals.*` | Java strategy engine |
| `tcg-app-data` | TCG-software user-authored indicators/signals/portfolios/baskets. Isolated DB; scoped Mongo role | TCG-software Python `WriteRepository` ONLY | TCG-software Python |

**DB→Mongo template bean map (Java):**
`data.repo.data.*`→`tcg` (default) · `data.repo.instrument.*`→`tcg-instrument` (`mongoTemplateInstrument`) · `data.repo.signals.*`→`tcg-signals` (`mongoTemplateSignals`) · `data.repo.raw.*`→`tcg-raw` (`mongoTemplateRaw`) · `data.repo.report.*`→`tcg-report` (`mongoTemplateReport`). Simulator hardcodes `getDatabase("tcg-instrument")`/`getDatabase("tcg")`.

### Collection-naming rule (Java) — important
Only **7 instrument collections** carry an explicit `@Document(collection="…")` (UPPERCASE — verified in code):
`INDEX`, `ETF`, `FUND`, `FOREX`, `INTRADAY_INSTRUMENT`, `INTEREST_RATE`, `INDEX_EOD_DATA`.
All other ~60 Java entities have NO `collection=` attribute → Spring Data defaults the collection to the **decapitalised class simple-name** (e.g. `HistoricalVar`→`historicalVar`, `IbPnL`→`ibPnL`, `TcgPosition`→`tcgPosition`). Names below marked *(derived)* follow this rule (rule confirmed; exact live casing — verify with `db.getCollectionNames()`).
`FUT_*`/`OPT_*` collections are created **dynamically** by `FutureRepositoryImpl`/`OptionRepositoryImpl` (prefix + uppercased root via `MongoTemplate`), not from a fixed list.
Sole explicit constant: `breakoutBuyVix` entity → `COLLECTION_NAME = "vixBreakoutBuyTrade"`.

---

## `tcg-instrument` — market data (read by Python)

Python read scope (`tcg/data/_mongo/registry.py`): prefix classification — `INDEX`, `ETF`/`FUND`/`FOREX`, `FUT_*` = `all_active`; `OPT_*` = `all_options` (discovered but deferred). Everything else (`INTRADAY_INSTRUMENT`, `INTEREST_RATE`, system collections) is ignored.

| Collection / prefix | Asset class | Writer (Java) | Notes |
|---|---|---|---|
| `INDEX` | INDEX | `IndexRepository` | single coll; `_id` = `internalSymbol` (String, e.g. `"SPX"`) |
| `ETF` | EQUITY | `EtfRepository` | single coll |
| `FUND` | EQUITY | `FundRepository` | single coll |
| `FOREX` | FX | `ForexRepositoryImpl` | single coll |
| `FUT_<root>` | FUTURE | `FutureRepositoryImpl` (prefix `FUT_`) | **one coll per underlying**; ≥37 in code: `FUT_VIX, FUT_SP_500, FUT_SP_500_EMINI, FUT_NASDAQ_100, FUT_IND_VIX, FUT_GOLD, FUT_CRUDE, FUT_EURUSD, FUT_BTC, FUT_ETH, FUT_T_NOTE_10_Y, …` |
| `OPT_<root>` / `OPT_<contract>` | OPTION | `OptionRepositoryImpl` (prefix `OPT_`) | one per underlying + some per-contract docs (e.g. `OPT_FUT_SP_500_EMINI_20250620_4450_P`); ≥20: `OPT_VIX, OPT_SP_500, OPT_NASDAQ_100, OPT_GOLD, OPT_CRUDE, OPT_BTC, OPT_ETH, …` |
| `INTRADAY_INSTRUMENT` | intraday bars | `IntradayInstrumentRepository` | Python excludes `intradayDatas` |
| `INTEREST_RATE` | rates | `InterestRateRepository` | risk-free curve inputs |
| `INDEX_EOD_DATA` | index EOD extras | `IndexEodDataRepository` | `_id`=composite `InstrumentDataId`; field `putImpVol` |
| `interestRatePrice` *(derived)* | rate prices | `InterestRatePriceRepositoryImpl` | per-date rate values |

### Document shapes — `tcg-instrument`

**Provider-keyed `eodDatas` OHLCV map** (all instrument collections):
```json
{ "_id": "SPX",
  "eodDatas": {
     "YAHOO":       [{"date":20240115,"open":4780.0,"high":4802.0,"low":4756.0,"close":4783.0,"volume":3.2e9}, …],
     "IVOLATILITY": [ … ] } }
```
- Top-level keys = provider names (`YAHOO`, `IVOLATILITY`, `CBOE`, `DERIBIT`, `COINGECKO`, …). No provider in query → first available used.
- `date` = **YYYYMMDD integer**; bars sorted ascending.
- Java bar (`model/data/Data.java`) fields: `date, time, open, high, low, close, adjustClose, volume, openInterest, bid, ask, bidSize, askSize`. **Python read model consumes `date/open/high/low/close/volume` only.**

**`_id` polymorphism (legacy):** per-collection `_id` is ObjectId, String (e.g. `"SPX"`), **or composite dict** (e.g. `{symbol, expiry}`). Python normalizes via `serialize_doc_id()`→string (composite → `key=val|key=val` sorted); `deserialize_doc_id()` tries dict→ObjectId→string. Java uses typed `@Id`: `String internalSymbol`, `OptionId`, `InstrumentDataId`, `TimeSeriesBucketId`, `TcgPositionId`.

**Futures (`Future extends Instrument`):** adds `expiration` (LocalDate), `expirationCycle` (String, e.g. `"HMUZ"` quarterly), `contractSize`. (Python stores `expiration` as datetime/ISO/YYYYMMDD-int — legacy inconsistency, parsed by `_parse_expiration`.)
**Options (`Option extends Instrument`):** adds `id` (`OptionId`), `expiration`, `strike`, `type` (C/P), `underlyingSymbol`, `contractSize`, `interpolated`, and **`eodGreeks: Map<Provider, List<Greek>>`**. `eodGreeks`/`intradayDatas` are NOT consumed by the Python read app (excluded from projections).

---

## `tcg` — operational (~40 collections, all *(derived)* unless noted)

- **Positions/trades:** `tcgPosition`, `sapphirePosition`/`sapphireComputePosition`/`sapphireOrder`/`sapphireTrade`, `cryptoAggregatePosition`, `account` (Binance trades — explicit in template ops), `imagineTrade`, `imagineSecurity`
- **PnL:** `ibPnL`, `fundPnL`, `imaginePnL`
- **Risk history:** `historicalDelta`, `margin`, `drawDownHistory`, `historicalVar`, `historicalStressTest`, `performanceAttribution`
- **Fund/investor:** `subscriptionRedemption`, `fundInfo`, `investor`
- **Simulator:** `simulatorPortfolio`, `simulationPortfolio`, `simulationPerformance`, `simulatorTrades`
- **Imagine:** `imaginePortfolio` (explicit; `_id`={date,name}, `positions[]`)
- **Real-time:** `realTimeData`, `singleRealTimeConfig`, `futureRealTimeConfig`
- **Auth/app:** `user`, `refreshToken`, `appConfig`, `app`
- **Mapping/config:** `dataMapping`, `instrumentCodeSettings`
- **Job bookkeeping:** `eventRegister`, `groupTaskParameters`, `batchMonitor`
- **Series/vol/calendar:** `timeSeriesBucket` (`_id`=`TimeSeriesBucketId`, holds `Data` bars), `volatilitySurface`, `workdayException`

**Key shapes:**
- PnL (`ibPnL`/`fundPnL`/`imaginePnL`): `date, dailyGross[Percent], dailyNet[Percent], mtd*, ytd*, assetTradingLevel, asset, assetTradingLevelSplit{}, assetSplit{}, riskWeighSplit{}, source`
- Position (`tcgPosition`): `id, expiration, strike, type, quantity, size, pxD1, px, pnl, imagineHoldingId, details[]`
- Trade (`account`, Binance): `id, price, qty, quoteQty, commission, commissionAsset, time, symbol, buyer, maker, bestMatch, orderId, update`
- VolatilitySurface (`volatilitySurface`): `id, expirations[], ttm[], strikes[], datas[]`

---

## `tcg-signals` — computed signals (~19, all *(derived)*)

| Collection | Entity | Info |
|---|---|---|
| `movingAverageSignal` (×2 repos) | `MovingAverageSignal` | MA signal series |
| `twoLinesCrossSMASignal` | `TwoLinesCrossSMASignal` | SMA-cross signals |
| `sMASignalSpx` / `sMASignalSpxGeneric` | `SMASignalSpx*` | SPX SMA signals |
| `dStatSignalSpx` | `DStatSignalSpx` | SPX D-stat signals |
| `vixCurve` / `curveData` | `VixCurve`/`CurveData` | VIX term-structure / curve |
| `flows` / `flowsDivergence` | `Flows`/`FlowsDivergence` | VIX flows + divergence |
| `vixBreakoutBuyTrade` (explicit const) | `BreakoutBuyVix` | VIX breakout-buy signal/trades |
| `navyLongData` / `navyShortData` / `navyShortV4Data` | `Navy*` | Navy strategy long/short |
| `monitorOscillator` / `monitorLagLines` / `monitorIchimoku` | `Monitor*` | Monitor indicators |
| `midnightData` | `MidnightData` | Midnight signal data |
| `strategy` | `Strategy` | Strategy state: `strategyId, regime, shortFuture, longFuture` |

> The `black.tail.signals.batch` job autowires repos that write `signalsCurve`, `signalsIchimoku`, `signalsMidnight`, `signalsNavyLongData`, `signalsNavyShortData`, `signalsTwoLinesCrossSMA`, `monitorLagLines`, `monitorOscillator` — collection casing differs slightly from the entity-derived names above (TODO: reconcile against live `getCollectionNames()`).

---

## `tcg-raw` — raw vendor feeds (~8, all *(derived)*)

| Collection | Source | Info |
|---|---|---|
| `rawIVolatilityFutFutures` / `rawIVolatilityFutFuturesPrice` | iVolatility | raw futures + price (impl prefixes `rawIVolatilityFutFuturesPrice_<…>`) |
| `rawIVolatilityFutOption` / `rawIVolatilityFutOptionRoot` / `rawIVolatilityFutUnderlying` | iVolatility | raw option chains (impl prefix `rawIVolatilityFutOption_<…>`) |
| `rawIVolatilityInterestRate` | iVolatility | raw rate feed |
| `investorCapitalSummaryMonthly` | Sudrania | monthly investor capital summary (lines) |
| `preludePnL` / `galaxy2xPnL` | Vadim | external PnL feeds |

## `tcg-report` (1, *(derived)*)

| Collection | Entity | Shape |
|---|---|---|
| `reportEdfVsImaginePnL` | `ReportEdfVsImaginePnL` | `_id`=date, `lines{}` map, `messages[]` |

## `tcg-app-data` — `2026-app-data` (TCG-software Python)

**Single collection** holding 4 doc types discriminated by a `type` field (`tcg/types/persistence.py`). DB name env `MONGO_APP_WRITE_DB_NAME` (default `tcg-app-data`); collection env `MONGO_APP_WRITE_COLLECTION` (default `2026-app-data`). Server-side scoped role `appDataWriter` / user `app-writer` grants `find/insert/update/remove/createIndex/listIndexes` on this one namespace only (zero visibility into `tcg-instrument`).

| `type` | Dataclass | Editable payload | Soft-delete |
|---|---|---|---|
| `indicator` | `IndicatorDoc` | `definition` (opaque dict) | `deleted: bool` + `locked` |
| `signal` | `SignalDoc` | `inputs[]`, `rules{}`, `settings{}`, `description` | `category=ARCHIVE` + `locked` |
| `portfolio` | `PortfolioDoc` | `legs[]`, `rebalance` | `category=ARCHIVE` + `locked` |
| `basket` | `BasketDoc` | `legs[]`, `asset_class` | `category=ARCHIVE` |

Common fields: `_id` (string → maps to `id`), `name`, server-stamped `created_at`/`updated_at`. Category enum: RESEARCH/DEV/PROD/ARCHIVE.

---

## Rough scale
- 6 DBs, ~80 collections on one cluster.
- `tcg-instrument`: dynamic — ≥37 `FUT_*` + ≥20 `OPT_*` (from code literals) + 7 single/explicit collections. Live count depends on which underlyings were loaded.
- `tcg` ~40 · `tcg-signals` ~19 · `tcg-raw` ~8 · `tcg-report` 1 · `tcg-app-data` 1.

## TODO / UNKNOWN
- Exact live collection **casing** for the ~60 *(derived)* Java collections — verify with `db.getCollectionNames()` per DB.
- Full live `FUT_*`/`OPT_*` enumeration (created dynamically).
- `tcg-signals` entity-derived names vs the `signals*`-prefixed names the batch job writes — reconcile against live DB.
