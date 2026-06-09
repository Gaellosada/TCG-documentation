# trajectoirecap.platform.simulator

## Purpose
Local research/backtesting library. Runs simulations over price and options data from the `tcg` Mongo cluster; outputs results to stdout, files, or (in limited classes) local Mongo. **Not deployed to ECS — no Dockerfile, no buildspec, no ECS task definition.**

## Status
**Active research tool — NOT a scheduled ECS job.** Evidence: no Dockerfile, no buildspec, no EventBridge/CloudWatch scheduler JSON in repo. Most recent commit: 2022-09-27. Runs are triggered manually by developers from IDE.

## Stack
| Layer | Technology |
|---|---|
| Language | Java 13 (`maven-compiler-plugin` `<release>13</release>`) |
| Framework | Spring Boot `2.3.0.RELEASE` (parent POM) — minimal usage; no `@SpringBootApplication` class |
| Build | Maven (`spring-boot-maven-plugin` `2.3.1.RELEASE`, `repackage` goal → fat JAR `simulator.jar`) |
| Key libs | `spring-boot-starter-data-mongodb` (no Spring data repos used; raw `MongoClient` only); `jfreechart` 1.0.13; `commons-math3` 3.3; `jblas` 1.2.3; `guava` 21.0; `feign` 9.3.1; `ehcache` 2.10.5; Lombok 1.18.12 |

## Entry Points
No single `@SpringBootApplication`. Each simulation is a standalone `main()`. Key mains:

| Class | Package | Purpose |
|---|---|---|
| `CombinedTF` | `com.simulations.trend` | Trend-following combo (Ehlers/Instant/MA) over crypto futures |
| `CryptoTF` | `com.simulations.trend` | Crypto trend-following; writes results to `FileOutputStream` |
| `BuyOptionFromImpliedVolWeeklyOneDH` | `com.simulations.options.ivBased.buy` | Options buy sim with delta-hedge, reads `btcOptionStruct`/`ethOptionStruct` from local Mongo |
| `SellOptionFromImpliedVolWeeklyOneDH` | `com.simulations.options.ivBased.sell` | Options sell sim with delta-hedge |
| `SystematicOptionalSP500` | `com.simulations.options.ivBased.volArbitrage` | SPX vol-arb sim, reads `spxOptionStructVol` |
| `MongoOptionsVolSurfaceWriteDeribit` | `com.simulator.database.mongodb` | One-off: build `btcOptionStruct`/`ethOptionStruct` vol-surface collections from raw `option` docs |
| `MongoOptionsVolSurfaceWriteSP` | `com.simulator.database.mongodb` | One-off: build `spxOptionStructVol` from raw `option` docs |
| `VolSurfaceCleanerSP` | `com.simulator.database.mongodb` | Clean/complete SPX vol surface; reads `spxOptionStructVol` from local Mongo |
| `VixMeanRevert` | `com.simulations.meanrevert` | VIX mean-revert sim |
| `VixCarry` / `SpxFirstFutures` | `com.simulations.vix` | VIX carry and SPX front-month futures sims |

## Architecture — Core Packages
```
com.simulator.process/     ← simulation engine: Manager, TradingMediator, AllocationProcess variants,
                              TradingProcess types (Symmetric/Asymmetric daily/intraday)
com.simulator.financial/   ← account model: Portfolio, Cash, TradingAccountAsset, DeltaHedge
com.simulator.indicator/   ← indicator library (MA, Bollinger, ATR, EhlersTrend, etc.)
com.simulator.interpreter/ ← signal generation: InterpreterFactory, Condition, LineInterpreterTF,
                              StopLoss, Regime interpreters
com.simulator.util/        ← TimeSeriesDaily, TimeSeriesCode, DayStructure, PriceStructure
com.simulator.statmath/    ← BasicBlackScholes, LetsBeRational (exact IV solver)
com.simulator.database/    ← data loaders: MongoConnect, MongoFutures, MongoOptionsOther,
                              MongoOptionsVolSurfaceRead/Write*, FileFiller, YahooDownloader,
                              QuandlDownloader
com.simulations/           ← runnable experiments: trend/, options/, vix/, meanrevert/, intraday/,
                              monitoring/
```

## Interactions
- **Calls no other TCG service.** Pure batch: reads Mongo → runs simulation → writes to stdout / local file / local Mongo.
- **No REST endpoints exposed.** No `@RestController`, no Spring web context.
- `ClientMessageSender` (HTTP POST/GET utility) present but only used in old `main()` test stubs.
- External data loaders (used historically; **not active scheduled pipelines**):
  - `YahooDownloader` — `ichart.finance.yahoo.com:80`
  - `QuandlDownloader` / `QuandlDownloaderOld` — `quandl.com/api/v1` + `v3`
  - `DownloadVolIndices` — `stoxx.com`, `cboe.com`, `nikkei.co.jp`

## Data

### MongoDB — `tcg` database on `10.0.5.10:27017,10.0.6.10:27017` (replica set `tcg-rs`)

| Collection | Direction | Used by |
|---|---|---|
| `tcg.future` | READ | `MongoFutures.fillFuturesFromMongoDB()` — loads all futures contracts for a given `rootUnderlying` |
| `tcg.forex` | READ | `MongoConnect.fillTimeSeries("forex", ...)` — loads FX spot series (BTC_USD, ETH_USD, EUR_USD, etc.) |
| `tcg.option` | READ | `MongoOptionsVolSurfaceWriteSP/Deribit` — reads raw per-contract option EOD data to build vol surface |
| `tcg.optionExpiry` | READ | `MongoOptionsOther.readOptionExpiryForStructure()` |
| `tcg.<arbitrary>` (passed as `collectionName` param) | READ | `MongoConnect.fillTimeSeries()` — generic time-series loader; callers use e.g. `"quotes"`, `"tcgTimeSeries"` |

### Local Mongo `localhost:27017`

| Collection | Direction | Used by |
|---|---|---|
| `tcg.btcOptionStruct` | WRITE | `MongoOptionsVolSurfaceWriteDeribit.writeBTCOptionVolSurface()` |
| `tcg.ethOptionStruct` | WRITE | `MongoOptionsVolSurfaceWriteDeribit.writeETHOptionVolSurface()` |
| `tcg.spxOptionStructVol` | WRITE | `MongoOptionsVolSurfaceWriteSP.writeSPXOptionVolSurface()` |
| `tcg.btcOptionStruct` / `tcg.ethOptionStruct` / `tcg.spxOptionStructVol` | READ | All options simulation `main()` classes |
| `tcg.<collectionName>` | WRITE | `MongoOptionsVolSurfaceWriteCommon.writeOptionVolSurfaceToMongoDB()` |

> **Note on LOCAL_URI vs TCG_MDB_URI:** Read classes (`MongoOptionsVolSurfaceRead`, `MongoOptionsVolSurfaceWriteCommon`) connect to `localhost:27017`. This is intentional: vol-surface builds run against prod data, persist results locally, then simulations read from local. `MongoFutures`, `MongoConnect`, `MongoOptionsOther`, `MongoOptionsVolSurface*Write*SP/Deribit` connect to prod `10.0.5.10`.

### `dwh` Postgres
None. This repo has no JDBC / Postgres dependency.

## Run Locally

```bash
mvn package -DskipTests
# Then run a specific main class, e.g.:
java -cp target/simulator.jar com.simulations.trend.CombinedTF
# Or run directly from IDE (typical usage)
```

**Required: SSM tunnel to Mongo** (for any class using `TCG_MDB_URI`):
```bash
aws ssm start-session --target i-0132f2ba5f7ed8c81 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["10.0.5.10"],"portNumber":["27017"],"localPortNumber":["27017"]}'
```

### Config Keys / Env Vars
No `application.properties` / `application.yml` in repo. All config is **hardcoded**:

| Item | Location | Value |
|---|---|---|
| Prod Mongo URI | `MongoUtils.java` constant `TCG_MDB_URI` | **Hardcoded** — credentials in `src/main/java/com/simulator/database/mongodb/MongoUtils.java` (SECURITY TODO) |
| Local Mongo URI | `MongoUtils.java` constant `LOCAL_URI` | `mongodb://localhost:27017` |
| Quandl API key | `QuandlDownloader.java` field `QUANDL_AUTH`; also in `QuandlDownloaderOld.java` and inline in `QuandlDownloader.java:212-213`; also in `ClientMessageSender.java:147` | **Hardcoded** (SECURITY TODO) |
| File paths | Various simulation `main()` methods (e.g. `VolSurfaceCleanerSP`, `MongoOptionsVolSurfaceWriteSP`) | Hardcoded developer-machine paths (`/Users/alejandro/...`) — must be changed before running |

## Gotchas

1. **No single entry point.** Each simulation class has its own `main()`. The fat JAR built by `spring-boot-maven-plugin` with `repackage` goal cannot be invoked directly without `-cp` or `--main-class`; typical usage is IDE run.

2. **Hardcoded prod Mongo credentials.** `MongoUtils.TCG_MDB_URI` contains a plaintext admin password directly in source (`src/main/java/com/simulator/database/mongodb/MongoUtils.java`). SECURITY TODO — do not publish this file publicly.

3. **Hardcoded Quandl API key** in `QuandlDownloader.java`, `QuandlDownloaderOld.java`, and `ClientMessageSender.java`. SECURITY TODO.

4. **Hardcoded developer file paths.** Approximately 48 occurrences of `/Users/alejandro/...` or `/Users/asantamaria/...` paths in simulation `main()` methods and utility classes. These must be updated before running file-based loaders (`FileFiller`, `VolSurfaceCleanerSP`, etc.).

5. **LOCAL_URI for vol-surface writes.** `MongoOptionsVolSurfaceWriteCommon` and `MongoOptionsVolSurfaceRead` write/read from `localhost:27017` only. A local Mongo instance must be running when building or reading the pre-computed vol-surface collections.

6. **`simulatorPortfolio` / `simulatorTrades` not written here.** Those `tcg` collections are written by `signals.parent/crypto.signals` (`CryptoSignalApplication`), NOT by this repo.

7. **Spring Boot dependency is vestigial.** `spring-boot-starter-data-mongodb` is declared but no Spring application context is started; there are no `@SpringBootApplication`, `@Service`, `@Component`, or `@Repository` annotations anywhere in the source tree. Spring is effectively unused at runtime.

8. **Git remote is AWS CodeCommit** (`git-codecommit.eu-central-1.amazonaws.com/v1/repos/trajectoirecap.platform.simulator`).
