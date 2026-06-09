# trajectoirecap.signals.rest

## Purpose
FastAPI REST server that serves computed trading signals and strategy results to `trajectoirecap.signals.board`. On startup, runs APScheduler jobs that call into `trajectoirecap.signals.lib` strategies to populate Redis; endpoints read back from Redis.

## Status
**LIVE** — active commits (latest: `885fe91`); Dockerfile + docker-compose present; `signals.board` calls this service.

## Stack

| Component | Detail |
|-----------|--------|
| Language | Python (Dockerfile: `python:3.12`) |
| Framework | FastAPI `>=0.68.0` |
| ASGI server | Uvicorn `>=0.15.0` |
| Scheduler | APScheduler `3.11.0` (AsyncIOScheduler) |
| Cache / job store | Redis `5.2.1` (host `redis`, port `6379`, db 0 = data, db 1 = APScheduler job store) |
| Signal lib | `trajectoirecap.signals.lib` — Git submodule at `app/signals` (CodeCommit `eu-central-1`: `trajectoirecap.signals.lib`) |
| Other deps | motor `3.7.0`, pymongo `>=4.10.0`, psycopg2 `2.9.10`, pandas-ta `0.3.14b0`, numpy `1.26.4`, pyarrow `19.0.0`, boto3 `>=1.36.2`, aiohttp `3.11.11` |

**Note:** `pymongo`/`motor`/`psycopg2` are declared dependencies but are **not imported or used** anywhere in this repo's own `app/` code. Usage, if any, lives inside `trajectoirecap.signals.lib` (submodule, not audited here).

## Deployment

- **Containerised:** `docker-compose up --build` (local); production container runtime **UNKNOWN** — no ECS task definition, buildspec, or EventBridge schedule committed to this repo. Audit attributed it to an unconfirmed ECS/container deployment.
- **Port:** `80` (Dockerfile `CMD uvicorn … --port 80`; docker-compose maps `80:80`)
- **Submodule init required before build:**
  ```
  git submodule init
  git submodule update
  ```
- **PYTHONPATH:** must be set to `/trajectoirecap/app/signals/src` (set in docker-compose; required for strategy dynamic imports)

## Interactions

| Direction | Caller / Callee | Protocol | Notes |
|-----------|----------------|----------|-------|
| Called by | `trajectoirecap.signals.board` | HTTP GET on port `80` | Hardcoded to `http://127.0.0.1:80` / `http://127.0.0.1` in `signals.board/src/utils/loading.py` — both services must share network (docker-compose or same host) |
| Calls | `trajectoirecap.signals.lib` | In-process (submodule) | Dynamically imports `Strategy` classes at startup |
| Calls | Redis | TCP `redis:6379` | Read (endpoints) + Write (scheduled jobs) |

## Endpoints

All routes are registered **dynamically** at startup by scanning `app/signals/src/trajectoirecap/services/strategies/` and `contracts_futures/` / `contracts_options/` for `*.py` files. Static routes:

| Method | Path | Tag | Description |
|--------|------|-----|-------------|
| GET | `/hello` | Health check | Returns `{"message": "Hello World!"}` |
| GET | `/health` | Health check | Returns `{"status": "healthy"}` |
| GET | `/category-routes?category=<name>` | Lists | Lists endpoints for one category (`signals`, `strategies`, `futures`, `options`, `health`, `hello`) |
| GET | `/all-categories` | Lists | Lists all registered endpoints grouped by category |

Dynamic routes (populated from submodule, actual paths unknown without submodule checked out):

| Method | Path pattern | Tag | Backend |
|--------|-------------|-----|---------|
| GET | `/signal/{strategy_name}` | Signals | Reads Redis key `{strategy_name}_signals` |
| GET | `/strategy/{strategy_name}` | Strategies | Reads Redis key `{strategy_name}` |
| GET | `/{contract_name}` | Contracts Futures | Reads Redis key `{contract_name}` from `contracts_futures/` subdir |
| GET | `/{contract_name}` | Contracts Options | Reads Redis key `{contract_name}` from `contracts_options/` subdir |

## Scheduled Jobs (APScheduler)

Runs on startup, reads strategies from the same submodule directory:

| Job | Trigger | Action |
|-----|---------|--------|
| `job_signals_{strategy_name}` (one per strategy) | Every 15 minutes | Calls `module.Strategy(redis_client).update_signals()` → writes to Redis |
| `job_strategies_{strategy_name}` (one per strategy) | Cron `hour=8, minute=0` UTC | Calls `module.Strategy(redis_client).update_strategies()` → writes to Redis |

APScheduler job store: Redis db 1 (`redis:6379/1`).

## Data

| Store | DB / Key | Direction | Notes |
|-------|----------|-----------|-------|
| Redis db 0 | `{strategy_name}_signals` | Write (scheduled) / Read (endpoint) | Updated every 15 min by signal jobs |
| Redis db 0 | `{strategy_name}` | Write (scheduled) / Read (endpoint) | Updated daily 08:00 UTC by strategy jobs |
| Redis db 0 | `{contract_name}` | Write (scheduled) / Read (endpoint) | Futures + options contracts |
| Redis db 1 | APScheduler job store | Write + Read | Persists scheduled job state |
| MongoDB `tcg-signals` | (various collections) | UNKNOWN — possibly via `signals.lib` | `pymongo`/`motor` in requirements but not used directly in this repo's `app/`; submodule may connect |
| PostgreSQL `dwh` | UNKNOWN | UNKNOWN | `psycopg2` in requirements but not imported in this repo's `app/`; audit noted writer of `data_live.signals_live` was unconfirmed — possibly via `signals.lib` |

## Run Locally

**Requirements:** Redis running at `redis:6379` (add `127.0.0.1 redis` to `/etc/hosts` or use docker-compose which starts Redis as a sidecar).

```bash
# 1. Init submodule (CodeCommit auth required)
git submodule init
git submodule update

# 2. Install deps
pip install -r requirements.txt

# 3. Set PYTHONPATH
export PYTHONPATH=app/signals/src

# 4. Start (from repo root)
uvicorn app.main:app --host 0.0.0.0 --port 80
```

**Via docker-compose (recommended — starts Redis sidecar automatically):**
```bash
docker-compose up --build
```

### Key env vars / config

| Name | Where set | Value |
|------|-----------|-------|
| `PYTHONPATH` | docker-compose `environment` | `/trajectoirecap/app/signals/src` |
| Redis host | Hardcoded in `app/config/redis_config.py` and `scheduler_config.py` | `redis` (hostname) — must resolve |
| Redis port | Hardcoded | `6379` |

No `.env` file, no `os.environ` reads in this repo's `app/` code. All connection config is hardcoded.

## Gotchas

1. **Submodule mandatory:** `app/signals/` is a Git submodule pointing to CodeCommit (`https://git-codecommit.eu-central-1.amazonaws.com/v1/repos/trajectoirecap.signals.lib`). Startup **raises `FileNotFoundError`** immediately if the submodule is not checked out. All route registration and job scheduling fail without it.

2. **Redis hostname hardcoded as `redis`:** `redis_config.py` and `scheduler_config.py` use `host='redis'` — not configurable via env var. Must resolve via docker-compose network or `/etc/hosts` entry. No fallback.

3. **All routes are dynamic:** actual signal/strategy/contract endpoints are not visible in this repo — they depend entirely on `*.py` files inside the `trajectoirecap.signals.lib` submodule at `trajectoirecap/services/strategies/`, `contracts_futures/`, and `contracts_options/`. Endpoint list changes when the submodule changes.

4. **`pymongo`, `motor`, `psycopg2` are unused in this repo.** They are installed via `requirements.txt` but not imported by any file in `app/`. Actual Mongo/Postgres usage (if any) lives inside the `signals.lib` submodule.

5. **`signals.board` uses hardcoded `127.0.0.1`** for the API URL — both services must run on the same host or network. No service-discovery or configurable URL.

6. **`pyproject.toml` is not the install manifest.** It contains a placeholder project config (`mon_second_package`, `Votre Nom`) with a local file path reference to a developer's machine. `requirements.txt` is the actual dependency list used by the Dockerfile.

7. **`test_hello.py` expects `"Bonjour le monde!"`** but `hello.py` returns `"Hello World!"`. Test is broken as written (will fail against the live code).
