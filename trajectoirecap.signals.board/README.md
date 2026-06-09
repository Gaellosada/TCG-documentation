# trajectoirecap.signals.board

## Purpose
Plotly Dash web dashboard (analyst-facing) that displays live signal indicators and backtest results for trading strategies. Consumes data exclusively from `trajectoirecap.signals.rest` via HTTP.

## Status
**Active analyst tooling — NOT a deployed ECS/Fargate service.** Evidence: no Dockerfile, no buildspec, no docker-compose, no EventBridge config in this repo. Latest commit: 2025-02-11. `gunicorn==23.0.0` is in requirements but no `Procfile` or invocation found. Runs locally or on a developer machine.

## Stack
| Component | Detail |
|-----------|--------|
| Language | Python (no pinned version in repo; `signals.rest` Dockerfile uses `python:3.12`) |
| Framework | Dash 2.18.2 + Flask 3.0.3 (Dash's underlying server) |
| Auth | `dash_auth==2.3.0` BasicAuth |
| Charts | Plotly 5.24.1 + Dash AG Grid 31.3.0 |
| HTTP client | `requests==2.32.3` |
| Metrics | `QuantStats==0.0.64`, NumPy 2.2.1, Pandas 2.2.3 |
| Formatting | `black==24.10.0` |
| Env | `python-dotenv==1.0.1` |

## Deployment
No deploy infrastructure found in this repo. The old README instructs:
```
python src/main.py
```
which starts Dash dev server on port `8050`.

For a production-like serve (not verified active): `gunicorn` is installed but no Procfile or invocation script exists.

## Interactions

| Direction | Target | Endpoint | Protocol |
|-----------|--------|----------|----------|
| CALLS | `trajectoirecap.signals.rest` | `GET /all-categories` | HTTP, hardcoded `http://127.0.0.1:80` |
| CALLS | `trajectoirecap.signals.rest` | `GET /strategy/{strategy_name}` | HTTP, hardcoded `http://127.0.0.1` (no port — defaults 80) |
| CALLS | `trajectoirecap.signals.rest` | `GET /signal/{strategy_name}` | HTTP, hardcoded `http://127.0.0.1` (no port) |

`trajectoirecap.signals.rest` must be running locally on port 80 for the board to function. Both repos are co-deployed or run together (signals.rest has its own `docker-compose.yml` with ports `80:80`).

## Data
`trajectoirecap.signals.board` does **not** connect to MongoDB or `dwh` directly. All data arrives through the `signals.rest` API.

- `pymongo==4.10.1` is in `requirements.txt` but **zero** `MongoClient`/`pymongo` usage found in `src/` — unused dependency.
- No `psycopg2`, `sqlalchemy`, or `dwh` SQL found anywhere in `src/`.
- The prior data audit attributed `dwh.data_live.signals_live` writes to the signals Python app; this repo is NOT the writer — `trajectoirecap.signals.rest` is the candidate (has `psycopg2==2.9.10`).

## Entry Point
```
src/main.py          # Dash app factory + navbar + routing
src/routes/strategy.py   # callbacks: equity curve, heatmap, metrics, trades table, signal plot
src/routes/signal.py     # stub callbacks (same pattern, unused in main.py)
src/routes/futures.py    # empty (1 line)
src/routes/options.py    # empty (1 line)
src/layouts/strategy.py  # layout template for /strategy/<name>
src/tools/chat_tool.py   # Bloomberg chat parser tool (layout + callbacks)
src/utils/loading.py     # fetch_from_api(), fetch_strategy_data(), fetch_signal_data()
src/utils/analyzer.py    # Bloomberg chat text parser
src/utils/calculate_metrics.py  # Sharpe, drawdown, vol, annual/monthly returns
src/utils/calculate_monthly_pnl.py  # monthly PnL aggregation
src/utils/plotting.py    # monthly returns heatmap + signal in_pos plot
```

## Run Locally

1. Create and activate venv:
   ```
   python -m venv .venv
   # Linux:  source .venv/bin/activate
   # Win:    .venv\Scripts\activate
   ```

2. Install dependencies:
   ```
   pip install -r requirements.txt
   ```

3. Create `.env` in the repo root (gitignored):
   ```
   APP_USER=<dashboard username>
   APP_PWD=<dashboard password>
   SECRET_KEY=<flask session secret>
   ```

4. Start `trajectoirecap.signals.rest` on port 80 (required — board is blank without it).

5. Launch:
   ```
   cd src
   python main.py
   ```
   Dev server starts on `http://localhost:8050`.

### Env vars
| Key | Used in | Purpose |
|-----|---------|---------|
| `APP_USER` | `src/main.py:32` | BasicAuth username |
| `APP_PWD` | `src/main.py:32` | BasicAuth password |
| `SECRET_KEY` | `src/main.py:45` | Dash BasicAuth session key |

No credentials are hardcoded in `src/`. `.env` is gitignored.

## Gotchas
- **Must run from `src/` directory** — imports are relative (`from layouts.strategy import …`); running `python src/main.py` from repo root will fail with `ModuleNotFoundError`.
- **API URLs are hardcoded** — `http://127.0.0.1:80` and `http://127.0.0.1` (no env var override). If `signals.rest` is on a different host/port, edit `src/utils/loading.py` directly.
- **`src/routes/futures.py` and `src/routes/options.py` are empty** — `/futures/<x>` and `/options/<x>` routes return stub text only.
- **`src/routes/signal.py` defines `signal_callbacks()` but it is never registered in `main.py`** — dead code.
- **Duplicate `dcc.Store(id="strategy-name-store")` in `strategy_template()`** — the layout renders it twice (lines 13 and 46 in `layouts/strategy.py`); Dash suppresses duplicate ID errors via `suppress_callback_exceptions=True`.
- **`pymongo` in requirements but never imported** — likely leftover from an earlier version.
- **`drafts/` folder** — contains `TCG_24_09_Short_Simple_New.py` (a standalone Dash layout prototype calling `http://127.0.0.1:8000/<name>`) and `01_Chat_export.ipynb` (Bloomberg chat export analysis notebook). Neither is imported by the main app.
- The `cache-directory/` folder contains binary cache files (likely `cachelib` from Flask-Caching); do not commit new entries.
