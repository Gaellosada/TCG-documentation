# victor

## Purpose
Python Shiny web app for internal strategy tracking. Displays live signal/backtest dashboards for VIX-based trading strategies; includes a Bloomberg chat parser tool. Named after developer Victor Moreno.

## Status
**LIVE** — deployed to Posit Connect (`allasso.app`); `prod` branch active; recent commits on `prod` branch (AWS CodeCommit `victor`).

## Stack

| Component | Detail |
|-----------|--------|
| Language | Python (no version pin in requirements.txt; `numba==0.60.0` requires Python 3.8–3.11) |
| Web framework | Shiny for Python (`shiny==1.2.1`) + `shinywidgets` + `shinyswatch` |
| Charts | Plotly (`plotly` via `highcharts-core==1.10.1` also present but unused in app.py) |
| Data processing | `pandas==2.2.3`, `numpy==2.0.2`, `scipy==1.14.1` |
| HTTP client | `requests==2.32.3` |
| Options pricing | `py-vollib-vectorized==0.1.1`, `py-vollib==1.0.1` |
| Performance stats | `QuantStats==0.0.64` |
| Formatting | `black==24.10.0` |
| Deploy tool | `rsconnect-python` (CLI: `rav.yaml` script) |
| Env config | `python-dotenv==1.0.0` |
| Auth | HTTP Basic Auth via `FLASK_SITE_USER` / `FLASK_SITE_PASS` env vars (`src/auth/utils.py`) — Flask remnant; auth wrapper present but app.py uses Shiny, not Flask |

## Deployment

| Item | Detail |
|------|--------|
| Host | Posit Connect at `https://allasso.app` |
| App URL | `https://allasso.app/content/a5d48a2a-80cc-4da3-87e6-0d0d116c416d/` |
| App mode | `python-shiny` |
| Deploy command | `rsconnect deploy shiny -n allasso --entrypoint src:app ./` (defined in `rav.yaml` as `rav deploy`) |
| Git remote | AWS CodeCommit `eu-central-1`: `https://git-codecommit.eu-central-1.amazonaws.com/v1/repos/victor` |
| Branches | `prod` (live), `dev` |
| Trigger | Manual deploy via `rav deploy`; no EventBridge/ECS/Step Functions involvement |
| No Dockerfile | Not containerized; no buildspec; not ECS |

## Interactions

| Direction | Target | Protocol/Detail |
|-----------|--------|-----------------|
| READS | `https://allasso.app/api_int/v1/equity/full-history` | GET, param `underlying_id` (e.g. `IND.VIX.CBOE`, `IND.VVIX.CBOE`), returns Parquet |
| READS | `https://allasso.app/api_int/v1/future/full-history` | GET, param `underlying_id`, returns Parquet |
| POSTS | `https://allasso.app/api-strategy/backtests/backtesting` | POST JSON TOM (Trade Order Model), returns backtest JSON |
| POSTS | `https://allasso.app/api-strategy/backtests/backtesting_raw` | POST JSON TOM (TCG raw format), returns backtest JSON |
| Auth header | `Authorization: Key <ALLASSO_API_KEY>` | API key from env var `ALLASSO_API_KEY` (via `.env`) |

`allasso.app` is both the deployment host **and** the data/backtest API provider.

## Data

**No direct database connections.** No MongoDB, no Postgres (`dwh`), no S3, no boto3, no pymongo. All data flows exclusively through the Allasso API.

Local file cache per strategy (written by `daily_data_run.py`, read by `app.py`):

| File | Location | Content |
|------|----------|---------|
| `raw_signals.csv` | `src/01_portofolio/01_TCG_24_09_Short_Simple_New/data/` | Computed signals DataFrame |
| `backtest_result.txt` | same | JSON backtest result from Allasso API |
| `tom.txt` | same | JSON Trade Order Model submitted to Allasso |

Only strategy `01_TCG_24_09_Short_Simple_New` has a functional data pipeline. Strategies 03–12 have stub or empty `get_the_data.py` files.

## Strategies (12 registered, 1 implemented)

| # | Name | Status |
|---|------|--------|
| 01 | TCG_24_09_Short_Simple_New | Implemented — full signal + backtest pipeline (Short VIX futures + long calls) |
| 02 | Azure_VIX_Signals_1st_Trade_Entry_4th_Exit_Vix20 | Stub only (`__init__.py` empty) |
| 03 | TCG_24_12_OCEAN3 | `get_the_data()` returns `None` |
| 04–12 | Various (NAVY, Cerulean, Lazulis, Market Timer…) | Empty `get_the_data.py` files |

## Tools Panel

| Tool | File | Function |
|------|------|----------|
| Chat Analyser | `src/02_tools/chat_analyzer/analyzer.py` | Parses pasted Bloomberg chat text; filters by speaker, contract name, buy/sell keywords; returns tabular trade list |

## Strategy 01 — Signal Logic (Short Simple VIX)
- **Instruments:** VIX spot (`IND.VIX.CBOE`), VVIX spot (`IND.VVIX.CBOE`), VIX futures (F1/F2/F3)
- **Signal:** dual-regime (Low Vol: VVIX20 ≤ 105; High Vol: VVIX20 > 105)
- **Trade:** Short -5 lots steepest VIX future + Long 50 lots 145%-moneyness calls per $1M; rolled 2 DTE before expiry
- **TOM submitted to Allasso:** `strategy_id = "Custom Structure"`, `initial_equity = 1000000`, `equity_type = "compounding"`

## Run Locally

**Entry point:** `python src/app.py`

**Daily data refresh** (run before app to update signals/backtest):
```
python src/utils/daily_data_run.py
```
(Note: `daily_data_run.py` imports from `tracking.src.*` — requires the repo to be importable as `tracking` package, i.e. run from the parent of the `victor/` directory with repo dir named `tracking`, or adjust `sys.path`.)

**Required env vars** (set in `.env` at repo root):

| Var | Used by | Purpose |
|-----|---------|---------|
| `ALLASSO_API_KEY` | `src/utils/utils.py` | Auth for all Allasso API calls |
| `FLASK_SITE_USER` | `src/auth/utils.py` | HTTP Basic Auth username (Flask auth wrapper — not active in Shiny app) |
| `FLASK_SITE_PASS` | `src/auth/utils.py` | HTTP Basic Auth password |

**Deploy to Posit Connect:**
```
rav deploy
```
(requires `rsconnect-python` installed and `allasso` server configured in `~/.rsconnect-python/`)

## Gotchas

- **Import path quirk:** `daily_data_run.py` and `get_the_data.py` import as `from tracking.src.*` — the repo must be on `sys.path` under the name `tracking` (the directory name is `victor`, not `tracking`). The deployed path on the server is `/home/vmoreno/tracking` (from `rsconnect-python/tracking.json`).
- **`app.py` references `src/short_simple_vix/data/`** (old path) in some code — current actual data path is `src/01_portofolio/01_TCG_24_09_Short_Simple_New/data/`. Verify which path is active before running.
- **11 of 12 strategies are stubs.** The dashboard only fully works for strategy 01.
- **`src/settings.py` is fully commented out** — dead code; env vars loaded directly in each module via `python-dotenv`.
- **`auth/utils.py` is a Flask Basic Auth decorator** — vestigial; `app.py` uses Shiny (not Flask); auth is not active in the running app.
- **No automated scheduling.** `daily_data_run.py` must be run manually to refresh data; no cron/EventBridge trigger.
- **No MongoDB, no `dwh` access.** Entirely self-contained via Allasso API. Not part of the ECS/EventBridge prod pipeline documented in the data audit.
- **`src/utils/drafts.txt`** — scratch notes file committed to repo (non-functional).
- Credentials (`ALLASSO_API_KEY`) loaded from `.env` — verify `.env` is NOT committed (`.gitignore` present but not audited here).
