# trajectoirecap.platform.parent

**Purpose** — Core production Java platform for Trajectoire CAP: market-data ingestion (~18 vendor feeds), risk/PnL/greeks/VaR compute, trading-signal generation, fund/admin reconciliation & exports, and the `svc` REST backend powering the internal Angular web app. ~2530 files, multi-module Maven.

**Status** — **LIVE prod.** Evidence: AWS CodeCommit `eu-central-1` origin; live ECS Fargate task-defs + EventBridge schedules under `*/scripts/tcg-scheduler*.json`; EKS `svc-app` Deployment; last commit `b0a350d2` (2026-03-19). Account `945696352514`, region `eu-central-1`.

---

## Stack

| Item | Value |
|---|---|
| Language | Java 17 (`<java.version>17</java.version>`, base image ECR `openjdk:17` / CodeBuild `corretto17`) |
| Framework | Spring Boot **2.6.14** (`spring.boot.version`), Spring Cloud `2021.0.4`, Spring Kafka `2.8.3`, Spring Data Mongo / Spring Data JPA |
| Build | Maven (parent `com.trajectoirecap:trajectoirecap.platform.parent:0.0.1-SNAPSHOT`, packaging `pom`) |
| DB drivers | MongoDB (Spring Data Mongo, multi-template), PostgreSQL JDBC `42.3.3` |
| Key libs | AWS SDK v2 `2.27.14` + spring-cloud-aws `2.3.0`; Apache POI `5.2.3` (xlsx); Quartz `2.3.2`; commons-math3 `3.5`; iText `5.5.13.3` + JFreeChart `1.5.4` (PDF/chart reports); Velocity (mail templates); JSch / mwiede-jsch (SFTP); Jsoup `1.15.3` (scrape); Microsoft Graph `6.14.0` + Azure Identity (SharePoint); Bloomberg `blpapi 3.24.4-1`; opencsv, zip4j, guava `33.1.0` |

---

## Modules

Parent aggregates 9 module groups (`<modules>` in root `pom.xml`; **`bb.svc` is commented out** — built separately, not in the reactor). Each leaf is either a **library** (shared code) or a **deployable** (Spring Boot app → Docker image). 18 root `buildspec-*.yml` files define the deployables; the rest build via `buildspec-all.yml`.

### core.parent — shared libraries (no standalone deploy)
| Module | Role |
|---|---|
| `commons` | Base utils, shared types |
| `data` | **Central data layer** — all Mongo `@Document` entities + repositories; wires 5 Mongo DBs (see Data). Depended on by nearly every other module |
| `copilot.data` | Reads `dwh.data_perf.<strategy>` for the cockpit/UI (read-only dwh) |
| `logic` | Domain/business logic |
| `financial.lib` | Financial maths (pricing, greeks, P&L primitives) |
| `kafka` | Spring Kafka producer/consumer wrappers |
| `mail` | Velocity-templated email sender |
| `aws.parent` | AWS clients: `aws.sqs`, `aws.s3`, + sns/ecs/sfn/cloudwatch/codebuild/core (artifacts in dependencyMgmt) — incl. `SchedulerService` (v2 `SchedulerClient`) that the app uses to read/edit EventBridge schedules |
| `office.parent` | `office.sharepoint` — Microsoft Graph SharePoint access (vendor file drops) |

### integration.parent — external vendor feeds (20 submodules; mostly deployable Fargate jobs)
`migration`, `binance.parent`, `deribit.parent`, `cboe`, `ivolatility`, `yahoo`, `coingecko`, `imagine.parent`, `sudrania`, `bitstamp`, `edfman` (+ `edfman.core`), `timeanddate`, `haruko`, `iqfeed`, `ib`, `sapphire.parent`, `dbselect` (+ `dbselect.core`), `kepler`.
- **`migration`** = manual backfill/migration tool and the **sole Java writer to Postgres `dwh`** (see Data). Not scheduled.
- Per-feed mechanism/destination/schedule: see audit `w2-ingestion.md` §2.1. Summary: Yahoo/CBOE/iVolatility/Deribit/CoinGecko/Bitstamp → Mongo instrument collections; Sudrania/EDF Man/dbSelect/Kepler/Sapphire/Imagine/IB → fund/admin collections; `haruko`,`iqfeed`,`binance` are dev/uninstrumented (saves commented out / sink unconfirmed).

### compute.parent — risk & PnL jobs (7, deployable)
| Module | Image | Computes → dest |
|---|---|---|
| `pnl.parent` (`pnl.compute`, `pnl.london.compute`) | `tcg-pnl-compute` | account/strategy/investor PnL → `tcg` PnL collections (London = stale dup) |
| `var` | `tcg-var` | historical VaR → `tcg.historicalVar` |
| `stress.test` | `tcg-stress-test` | scenario stress → `tcg.historicalStressTest` |
| `greeks.core` | `tcg-greeks` | aggregated delta → `tcg.historicalDelta` |
| `risk.report.parent` (`risk.report.mail`) | `tcg-risk-report` | daily risk report + email |
| `trade.position.check` | `tcg-trade-position-check` | reconciliation → email + `tcg.batchMonitor` |
| `drawdown.parent` (`drawdown.compute`) | — | drawdown → `tcg.drawDownHistory` |

### signals.parent — signal generation (6, deployable)
`black.tail.signals.parent` (`tcg-black-tail-signals-batch` → `tcg-signals` 8+ collections), `signals.trading` (reads **dwh** `data_live.signals_live` → `tcg` positions/orders), `crypto.signals`, `crypto.report`, `black.pearl.signals` (VIX-fut + SPX), `certif.report`.

### export.parent — reconciliation / reporting / exports (11, deployable)
`performance.attribution`, `expiration.notification`, `position.export`, `sudrania.file.check`, `sudrania.check`, `signals.report.export` (reads **dwh**), `signals.vix.flows`, `margin`, `pnl.check.export`, `external.risk.report.export`, `black.tail.signals.export`.

### svc — REST backend (deployable; **the web-app server**)
Spring Boot app `SvcApplication`, **port 8080**, profile `prod`. 37 `@RestController`s. Powers the internal Angular UI (browse instruments/futures/options, risk/greeks/portfolio/simulator, signal viewers, scheduler & deploy control, auth). See Interactions.

### Others
| Module | Role |
|---|---|
| `simulator` | Portfolio simulation engine/app (`simulatorPortfolio`/`simulationPerformance` in `tcg`) |
| `tools` | `Run*Task` wrappers calling ECS `runTask` with date overrides — the on-demand/back-fill launcher for the batch images |
| `sample.parent` | `sample.kafka` / `sample.rest` / `sample.aws.sqs` / `sample.mail` — example/reference apps, not prod |
| `bb.svc` | Bloomberg (`blpapi`) data service, SQS-driven (`SqsBbRequestListener`). **Excluded from parent build** (commented in root pom); built via its own `buildspec` / deployed on a Bloomberg-licensed host |

---

## Deployment

Two distinct runtimes:

**1. `svc` web backend → EKS (Kubernetes).** `k8s/svc-app.yaml`: Deployment `svc-app` (1 replica), image `…/tcg-svc`, container port 8080, Service NodePort 30100. `k8s/ingress.yaml`: ALB ingress (`internet-facing`), external-dns hostname **`www.aisy.fr`** → service:80 → 8080. Mongo creds injected as k8s `secretKeyRef` from secret `mongo-secret` (env `USER_NAME`/`USER_PASSWORD`). Build: `buildspec-svc.yml` → `mvn install` → `docker build -f svc/Dockerfile` → push ECR `tcg-svc:latest`.

**2. All batch jobs → ECS Fargate, triggered by EventBridge Scheduler.** Authoritative path:
```
git push → CodeBuild (buildspec-<module>.yml | buildspec-all.yml): mvn install → docker build → ECR push
        → EventBridge Scheduler cron (tz Europe/Paris, role ecsSchedulerRole)
        → ECS Fargate RunTask, cluster tcg-svc-cluster (params via container env overrides)
        → java -jar <module>.jar → Mongo / dwh / email / SQS
```
- Schedules: per-module `*/scripts/tcg-scheduler*.json` (`aws scheduler create-schedule`), applied by `create/update-scheduler.sh`. Example `yahoo`: `cron(0 8 * * ? *)` → task-def `tcg-yahoo` on cluster `tcg-svc-cluster`, subnets `subnet-0ba231b4cb45624a7`/`…fa19`, SG `sg-040738dfe7d0084d9`, public IP DISABLED, `MaximumRetryAttempts: 0`.
- **Step Functions** — `tcg-risk-report-step-function` chains the morning risk pipeline (VaR → stress → load position → risk report); ASL not in repo.
- **Legacy CloudWatch Events** (`*/aws/4-put-rule.json` + `3-register-task-definition.json`, role `ecsEventsRole`) still committed for `crypto.signals`/`crypto.report`/`certif.report`/`pnl.check.export`/`dbselect`/`migration` — being superseded by Scheduler; treat `aws/*-put-rule.json` as stale/templated.
- **`tools/Run*Task`** = on-demand launcher (ECS `runTask`) for modules with no committed schedule.
- 18 deployable buildspecs at repo root (one per image): `svc, cboe, yahoo, sudrania, kepler, imagine, imagine-override-px, greeks, var, stress-test, pnl-compute, position-export, risk-report, trade-position-check, performance-attribution, expiration-notification, black-tail-signals-batch`, + `buildspec-all.yml`.
- **No EKS CronJobs** — EKS hosts only the `svc` web app.

---

## Interactions

**Exposes:** `svc` HTTP API on **:8080**, externally at **`https://www.aisy.fr`** (ALB). JWT-style auth: `POST /token`, `POST /refresh-token` (`AuthController`; `SecurityConfig`; `svc.token-validity-minutes=2`, refresh 48h). Sample endpoints (37 controllers): `/index/list`,`/etf/{id}`,`/forex/list`,`/fund/{id}`,`/future/{collection}/{expi1}/{expi2}`,`/option/{collection}/…`,`/option/volatility-surface/{code}/{date}`,`/intraday/*`,`/instrument-mapping/*`,`/config/*`,`/rt/*` (real-time configs),`/scheduler/update`,`/deploy/list`,`/deploy/start/{id}`,`/event-register`,`/bloomberg/submit`, plus signal controllers (Curve/Navy/Midnight/MarketTimer/PutArbitrage/ShortSimple/Ocean/Cerulean/…).

**Consumed by:**
- **`trajectoirecap.platform.front`** (Angular web app) — the front-end repo that calls `svc` (ref found in `…/component/back-end-log/back-end-log.component.ts`). This is the primary client.
- **TCG-software** (separate Python/FastAPI project) reads the **same Mongo `tcg-instrument`** DB read-only via SSM tunnel (does NOT call `svc`).

**Internal:** modules share `core.parent` libs; jobs talk to each other only via shared Mongo/dwh state + SQS (e.g. `edfman` → S3 + SQS event; `bb.svc` consumes SQS). `svc` itself runs in-process Quartz crons (`svc.sapphire.*.cron`, `svc.dbselect.cron`, `svc.bloomberg.load.cron`, `svc.cockpit.strategy.file.cron`).

---

## Data

MongoDB replica set `tcg-rs` @ **`10.0.5.10:27017`** (prod, private VPC). The `data` module wires **5 separate MongoTemplates**, one per DB, each from its own URI key in `core.parent/data/src/main/resources/mongodb-config.properties` (config classes `MongodbConfig`, `MongodbInstrumentConfig`, `MongodbReportConfig`, `MongodbSignalsConfig`, + raw). Only **7** instrument entities carry an explicit uppercase `@Document(collection=…)` (`INDEX`,`ETF`,`FUND`,`FOREX`,`INTEREST_RATE`,`INDEX_EOD_DATA`,`INTRADAY_INSTRUMENT`); ~60 others derive collection name from the class (Spring decapitalised default).

| Mongo DB | URI key (`mongodb-config.properties`) | Direction | Contents |
|---|---|---|---|
| `tcg` | `mongodb.uri` | R/W | positions, PnL, trades, margins, risk history (delta/VaR/drawdown/stress), simulator & Imagine portfolios, users/auth, configs, vol surfaces, job bookkeeping |
| `tcg-instrument` | `mongodb.instrument.uri` | R/W | EOD OHLCV per instrument (`INDEX`/`ETF`/`FUND`/`FOREX`/`FUT_<root>`/`OPT_<root>`), rates, intraday |
| `tcg-raw` | `mongodb.raw.uri` | R/W | raw vendor feeds pre-normalization (iVolatility, Sudrania, prelude/galaxy PnL) |
| `tcg-report` | `mongodb.report.uri` | R/W | reconciliation reports (`reportEdfVsImaginePnL`) |
| `tcg-signals` | `mongodb.signals.uri` | R/W | computed signals (MA/SMA-cross/Ichimoku/VIX-curve/flows/breakout/Navy/oscillator) + `strategy` regime state |

(Exact collection inventory + document shapes: audit `w1-mongodb.md`.) Note: `tcg-app-data` is **not** owned here — it belongs to TCG-software (Python).

**Postgres `dwh`** on RDS `tcgdwh.ckzfuf3f1u8v.eu-central-1.rds.amazonaws.com/dwh`. 4 modules connect via `spring.datasource.url=jdbc:postgresql://…` in their `application.properties` / `copilot-data-config.properties`:

| Module | dwh access | Tables |
|---|---|---|
| **`migration`** | **WRITE** (sole Java writer; JPA `@Entity`/`@Table` + raw JDBC) — manual backfill, **not scheduled** | `data_ohlc.bitstamp_btcusd`/`_ethusd`(+`_22h`), `data_option.deribit_btc_usd`, `data_raw.raw_deribit_options_eod`, `data_raw.raw_ib_nav`, `archive.raw_coinapi_options_eod`, `accounts.galaxy_2x_pnl`, `accounts.ms_pnl_gross`, `data_perf.<strategy>` (via `RunExportDataToVadimDbFromCsv`) |
| `signals.trading` | READ | `data_live.signals_live` |
| `signals.report.export` | READ | `data_live.signals_live` |
| `copilot.data` | READ | `data_perf.<strategy>` |

`data_live.signals_live` is **written by an external system not in this repo** (every reference here is a `SELECT`). dwh schema detail: audit `w3-compute-sched-sql.md` §2.4.

---

## Run locally

- **Build all:** `mvn install` from repo root (parent reactor; `bb.svc` excluded — build it standalone if needed).
- **`svc` web app:** `mvn install` then `docker build -f ./svc/Dockerfile -t tcg-svc ./svc` && `docker run -p 8080:8080 tcg-svc` (per `svc/readme.txt`). Local run: `java -jar svc.jar` (default profile uses `application.properties`; prod profile is `prod`). Needs `svc/src/main/resources/Logo.png` (copied to `/app/Logo.png` in image).
- **Reach prod Mongo from a dev box:** SSM port-forward through bastion `i-0132f2ba5f7ed8c81` to `10.0.5.10:27017` (the pattern TCG-software's `tcg/core/tunnel.py` uses; env `MONGO_URI`, `LOCAL_PORT=27017`).
- **Run a batch job locally:** run its `<Module>Application` main, or use a `tools/Run*Task` wrapper (these fire ECS RunTask, not a local run). Job params come from ECS container env overrides in prod.
- **Key config keys (names only):** Mongo — `mongodb.uri`, `mongodb.instrument.uri`, `mongodb.raw.uri`, `mongodb.report.uri`, `mongodb.signals.uri`. dwh — `spring.datasource.url` / `.username` / `.password`. `svc` — `server.port`, `svc.working-path`, `svc.refresh-token-secret`, `rsa.private-key`, `rsa.public-key`, `svc.*.cron`. K8s env — `USER_NAME`, `USER_PASSWORD` (from secret `mongo-secret`).

---

## Gotchas

- **SECRETS COMMITTED IN PLAINTEXT (rotate + move to Secrets Manager — Kepler already uses AWS Secrets Manager, the right pattern):**
  - Mongo connection URIs with admin creds in `core.parent/data/src/main/resources/mongodb-config.properties` (all 5 DBs).
  - `dwh` Postgres password in `signals.parent/signals.trading/.../application.properties`, `integration.parent/migration/.../application.properties`, `export.parent/signals.report.export/.../application.properties`, `core.parent/copilot.data/.../copilot-data-config.properties`.
  - `svc` `application.properties` + `application-prod.properties`: `svc.refresh-token-secret`, hardcoded `svc.refresh-token-iv` (16-byte literal, value not reproduced), RSA `rsa.private-key` / `rsa.public-key`.
  - Vendor SFTP/API creds hardcoded in `cboe`/`sudrania`/`ivolatility` `*Config` classes.
- **5 Mongo connections, not 1** — each DB has its own URI/template; a job that touches `tcg` and `tcg-instrument` opens two.
- **Spring naming defaults the collection name** for ~60 unannotated entities (decapitalised class name, e.g. `HistoricalVar`→`historicalVar`); the explicit uppercase names are the exception, not the rule.
- **`bb.svc` is out of the reactor** (commented in root pom) — `mvn install` at root won't build it. Bloomberg needs a licensed host.
- **EKS = `svc` only.** All scheduled work is ECS Fargate; do not look for k8s CronJobs.
- **Schedule source of truth = `*/scripts/tcg-scheduler*.json`** (EventBridge Scheduler). `*/aws/*-put-rule.json` are stale/templated CloudWatch-Events leftovers (uniform `cron(0 7 * * ? *)`); for `dbselect`/`migration` only the stale rule exists — verify live cron in the EventBridge console.
- **Known anomalies (audit-flagged, verify in console):** `signals.trading` scheduler points at the `tcg-var` task definition; `pnl.london.compute`'s scheduler is a verbatim copy of `pnl.compute` (London not independently scheduled).
- **Dev-only / not instrumented in prod:** `binance.trade` (Mongo save commented out), `haruko` (`*Test` entrypoints), `iqfeed` (real-time sink unconfirmed), `sample.parent/*`.
- **Hardcoded local path** in default `svc/application.properties`: `svc.working-path=/Users/thaiooo/Documents/Working` — override for any non-author machine (prod profile uses `./working`).
- **Region/account fixed:** `eu-central-1`, `945696352514`; ECR base `…/openjdk:17`. Container TZ `Europe/Paris` (all cron expressions are Paris time).
