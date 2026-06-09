# MongoDB — Ingestion (how each collection gets filled)

Two writer classes fill Mongo: **external feeds** (vendor data → Mongo) and **internal compute/signals jobs** (read Mongo → derive → write Mongo). Both run as Spring-Boot Docker images on **ECS Fargate**, triggered by **AWS EventBridge Scheduler** (authoritative) — a few still on legacy CloudWatch Events Rules.

- Region `eu-central-1`, account `945696352514`. ECS cluster `tcg-svc-cluster`.
- Prod entrypoint = each module's `<Module>Application` `CommandLineRunner`, reading params (`DATE`, `DEPTH`, `SHIFT`, `IS_COMPUTE_559/569/DBSI`, …) from ECS `containerOverrides`. The `run/Run*.java` classes are dev/back-fill utilities (excluded from prod component-scan), NOT the scheduled entrypoints.
- Schedule source-of-truth = each module's `scripts/tcg-scheduler*.json` (`aws scheduler create-schedule`). `aws/*-put-rule.json` files are stale/templated unless the module has NO `scripts/` (then legacy rule is the only signal — cron unconfirmed).
- Schedules are `Europe/Paris`; cron `2-6` = Mon–Fri.

---

## A. External feeds → Mongo

All 18 `integration.parent` submodules. (Detailed per-source mechanism/citations also in `data-stores-and-sources-audit/output/w2-ingestion.md`.)

| Source | Module | Mechanism | Writes (Mongo) | Schedule |
|---|---|---|---|---|
| Yahoo Finance | `yahoo` | REST (Feign) + CSV loaders | `INDEX` (VIX complex, VIX9D/3M/6M, VVIX, SP500), `ETF` (SPY + bond/treasury), `FOREX` (EUR/USD) | `tcg-yahoo-scheduler` `cron(0 8 * * ? *)` daily 08:00 |
| CBOE (DataShop) | `cboe` | SFTP ZIP→CSV + S3 archive | `OPT_*` (Provider.CBOE) — VIX/SPX underlying option quotes | `tcg-cboe-scheduler` `cron(0 6 * * ? *)` daily 06:00 |
| iVolatility | `ivolatility` | FTP ZIP→CSV + S3 | `FUT_*`, `OPT_*`, `tcg-raw.rawIVolatility*`, `dataMapping`; futures/options EOD + instrument defs + rates | `tcg-ivolatility-scheduler` `cron(0 7 * * ? *)` daily 07:00 |
| Deribit | `deribit.data` | REST (Feign) | `OPT_*`/`FUT_*` (Provider.DERIBIT) — BTC/ETH price + chains | `tcg-deribit-scheduler` `cron(0 4 * * ? *)` daily 04:00 |
| CoinGecko | `coingecko` | REST (Feign) | `FOREX` (Provider.COINGECKO) — daily coin prices | `tcg-coingecko-scheduler` `cron(0 9 * * ? *)` daily 09:00 |
| Bitstamp | `bitstamp` | REST (Feign) | Forex repo (BTC_USD/ETH_USD 22h + daily) — **exact repo unconfirmed** | `tcg-bitstamp-scheduler` `cron(0 4 * * ? *)` daily 04:00 |
| Binance | `binance.trade` | REST | `account` (Binance trades) — **save commented out → dev/uninstrumented** | none (manual) |
| Interactive Brokers | `ib` | REST (IB Flex XML) | fund NAV/Flex; persisted to PnL via `migration` (`MigrationIbNavService`) — `RunIbNav` itself only prints | `tcg-ib-scheduler` `cron(30 23 * * ? *)` daily 23:30 |
| Sudrania (fund admin) | `sudrania` | SFTP/FTPS → S3 + SharePoint; `.xlsx` (POI) | `investorCapitalSummaryMonthly` (`tcg-raw`) | `tcg-sudrania-scheduler` `cron(0 13,15,16,17,18 ? * 2-6 *)` Mon–Fri |
| EDF Man / Marex (clearing) | `edfman` | SFTP → S3 + **SQS event** | files→S3 + SQS (no direct Mongo write; downstream consumer persists) | `tcg-edfman-scheduler` `cron(0 9,10,…,17 ? * 2-6 *)` Mon–Fri hourly 09–17 |
| dbSelect (OTC broker) | `dbselect` | SharePoint `.xlsx` (POI) + REST | `fundPnL`, `fundInfo`, `FUT_*`, `OPT_*`, `dataMapping` | only stale `aws/4-put-rule.json` (`cron(0 7 * * ? *)`, templated) → **cron unconfirmed** |
| Kepler Cheuvreux | `kepler` | SFTP (creds in AWS Secrets `prod/kepler/sftp`) → SharePoint | none direct (files→SharePoint; DB-integrated by Sapphire) | no committed scheduler → **unconfirmed** (likely via Sapphire) |
| Sapphire (Kepler integrator) | `sapphire.batch` | reads Kepler files from SharePoint (POI) | `fundPnL`, `fundInfo`, `imaginePnL`, `sapphireOrder`, `sapphirePosition`, `sapphireTrade` | `tcg-sapphire-position-scheduler` `cron(30 8 * * ? *)` daily 08:30 (triggers the compute runner; the Kepler-file `Run*` is manual) |
| Imagine Software (PMS) | `imagine.batch` | REST (token auth) | `imaginePortfolio`, `imagineSecurity` | `tcg-imagine-scheduler` `cron(0 8 ? * 2-6 *)` Mon–Fri 08:00 |
| Imagine — price override | `imagine.override.px` | REST push + email | writes back to Imagine (NOT a Mongo landing) | on-demand/manual |
| IQFeed (DTN real-time) | `iqfeed` | streaming socket (iqfeed4j, `@EnableScheduling`) | real-time cache; **sink unconfirmed**; reads `singleRealTimeConfig`/`futureRealTimeConfig` | long-running service (not an ECS cron) |
| timeanddate.com | `timeanddate` | HTML scrape (Jsoup) | `workdayException` | no committed scheduler → **unconfirmed** (annual/manual) |
| Haruko (crypto OMS) | `haruko` | REST (Feign) | **dest unconfirmed** — entrypoints are `*Test` | none (dev/test) |
| Legacy "Vadim" SQL + backfills (CoinAPI, MS, Galaxy 2x) | `migration` | JPA/JDBC + CSV | `FUT_*`, `OPT_*`, `FOREX`, `ibPnL` — **and the only Java writer to Postgres `dwh`** | only stale `aws/4-put-rule.json` (templated) → **functionally a manual backfill tool** |

**Mechanism notes:**
- Feign REST clients: yahoo, coingecko, bitstamp, deribit, ib, imagine, haruko.
- SFTP (JSch) file-drops: cboe, ivolatility, sudrania, kepler, edfman. S3 archive: cboe, ivolatility, edfman. SharePoint (MS Graph): sudrania, kepler.
- EDF Man is event-driven: after SFTP download posts an SQS message (`-MARGIN`/SOD) triggering a downstream PnL/margin check.
- **iVolatility has NO dedicated buildspec** (built via `buildspec-all.yml`) despite being a major feed.

---

## B. Internal compute/signals jobs → Mongo

| Job (ECS image) | Module | Writes (DB.collection) | Schedule |
|---|---|---|---|
| `tcg-pnl-compute` | `compute/pnl.compute` | `tcg` PnL (`imaginePnL`,`fundPnL`,`ibPnL`, gross) | `cron(0 0 ? * 2-6 *)` 00:00 |
| `tcg-pnl-compute` (London) | `compute/pnl.london.compute` | `tcg` PnL | scheduler JSON is a **verbatim dup of pnl.compute** → London not independently scheduled (flagged bug) |
| `tcg-var` | `compute/var` | `tcg.historicalVar` (559/569/DBSI) | `tcg-var-scheduler` `cron(30 14 ? * 2-6 *)` + histo `cron(0 1 ? * 2-6 *)` |
| `tcg-stress-test` | `compute/stress.test` | `tcg.historicalStressTest` | `cron(30 14 …)` + histo `cron(0 4 …)` |
| `tcg-greeks` | `compute/greeks.core` | `tcg.historicalDelta` | **No scheduler JSON** → likely a step in the risk-report Step Function (unconfirmed) |
| `tcg-risk-report` | `compute/risk.report.mail` | `tcg` (refreshed Imagine positions/portfolio) + email | morning `cron(30 10 ? * 2-6 *)`; indigo `cron(30 11 ? * 2-6 *)` |
| `tcg-trade-position-check` | `compute/trade.position.check` | `tcg.batchMonitor` + email; reloads `tcg` positions | `cron(10,50 9 ? * 2-6 *)` 09:10 & 09:50 |
| (drawdown) | `compute/drawdown.compute` | `tcg.drawDownHistory` | no scheduler → on-demand |
| `tcg-black-tail-signals-batch` | `signals/black.tail.signals.batch` | `tcg-signals` (curve/ichimoku/midnight/navy/two-lines-cross/lag-lines/oscillator) | `cron(15 8 ? * 2-6 *)` 08:15 |
| `tcg-var` (signal trading) | `signals/signals.trading` | `tcg` signal-driven positions/orders + `tcg.batchMonitor` | morning `cron(30 9 …)`; intraday `cron(55 15,16,17,18,19,20 …)`. **TaskDef mis-points at `tcg-var`** (flagged) |
| `tcg-crypto-signals` | `signals/crypto.signals` | `tcg.simulatorPortfolio`/`simulatorTrades` + vol surface + email | `cron(0 7 * * ? *)` 07:00 — **legacy CloudWatch rule** |
| `tcg-crypto-report` | `signals/crypto.report` | email/report | legacy rule |
| (black.pearl) | `signals/black.pearl.signals` | `tcg-signals` (VIX/SPX family) | no scheduler → on-demand |
| (certif.report) | `signals/certif.report` | email/report | legacy rule |
| `tcg-performance-attribution` | `export/performance.attribution` | `tcg.performanceAttribution` + `tcg.batchMonitor` | `cron(0 7 1-3 * ? *)` monthly days 1–3, 07:00 |
| `tcg-expiration-notification` | `export/expiration.notification` | email + refreshed `tcg` positions | `cron(0 9,11 ? * 2-6 *)` 09:00 & 11:00 |
| `tcg-position-export` | `export/position.export` | file/email/SharePoint | no scheduler → on-demand (in buildspec) |
| `tcg-sudrania-file-check` | `export/sudrania.file.check` | email/`tcg.batchMonitor` | `cron(30 18 ? * 2-6 *)` 18:30 |
| (signals.report.export) | `export/signals.report.export` | file/email (reads `dwh`) | no scheduler → on-demand |
| (signals.vix.flows) | `export/signals.vix.flows` | `tcg-signals.signalsVixFlows*` + `tcg.batchMonitor` | no scheduler → on-demand |
| (margin) | `export/margin` | `tcg.margin` + `tcg.batchMonitor` | no scheduler → on-demand |
| (pnl.check.export) | `export/pnl.check.export` | file/email | legacy rule |
| (sudrania.check / external.risk.report.export / pnl.sapphire.mail) | `export/*` | email/file | on-demand |

---

## Scheduling mechanism

```
git push → AWS CodeBuild (buildspec-<module>.yml / buildspec-all.yml): corretto17 → mvn install
         → docker build -f <module>/Dockerfile → docker push 945696352514.dkr.ecr.eu-central-1.amazonaws.com/<image>:latest
         → EventBridge Scheduler (cron, Europe/Paris, RoleArn ecsSchedulerRole)
         → ECS Fargate RunTask on cluster tcg-svc-cluster (containerOverrides env: DATE/DEPTH/SHIFT/…)
         → container CMD: java -jar -Dspring.profiles.active=prod <module>.jar
         → writes Mongo / dwh, sends mail/SQS
```

- **EventBridge Scheduler → ECS Fargate** = authoritative. App reads/edits the same schedules via v2 `SchedulerClient` (`core.parent/aws.parent/aws.ecs/.../SchedulerService.java`).
- **Step Functions** — one state machine `tcg-risk-report-step-function` chains the morning risk pipeline (VaR → stress → load-position → risk-report). ASL definition NOT in repo (unconfirmed). Explains greeks/var/stress having weak standalone schedules.
- **Legacy CloudWatch Events Rules** (`aws/4-put-rule.json` + `5/6`, role `ecsEventsRole`): still live for `crypto.signals`, `crypto.report`, `certif.report`, `pnl.check.export` (+ feeds `dbselect`, `migration` — stale rule files only).
- **EKS hosts the web app only — NO k8s CronJobs** (`grep "kind: CronJob"` = empty). Batch runs on Fargate.

## Security
- ⚠ Mongo admin URIs (with embedded creds) committed in `core.parent/data/.../mongodb-config.properties` (5 URIs) — **SECURITY TODO**, values not reproduced.
- ⚠ Vendor SFTP/API creds committed in `cboe/.../CboeConfig.java`, `sudrania/.../SudraniaConfig.java`, `ivolatility/.../IVolatilityConfig.java` — **SECURITY TODO**. (Kepler correctly uses AWS Secrets Manager.)

## TODO / UNKNOWN
- Live cron for `dbselect`, `migration`, `timeanddate`, `kepler` (no committed `scripts/` scheduler) — verify in EventBridge console (`aws scheduler list-schedules` / `aws events list-rules`).
- `tcg-greeks` trigger; Step Functions ASL graph.
- Sinks: Bitstamp Forex repo, IQFeed real-time, Haruko persistence.
- On-demand modules' real cadence (margin, position.export, signals.vix.flows, signals.report.export, black.pearl, sudrania.check, external.risk.report.export, pnl.sapphire.mail, drawdown).
