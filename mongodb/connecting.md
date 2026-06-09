# MongoDB — Connecting (SSM tunnel + mongosh + Python)

The cluster is on a **private subnet**: `10.0.5.10:27017`, replica set `tcg-rs`, `eu-central-1`. It is NOT publicly reachable. Connect by port-forwarding through an EC2 bastion using **AWS SSM Session Manager**, then point any Mongo client at `localhost`.

> **No live credentials in this doc.** Use env-var names / placeholders only. Get the actual Mongo username/password and AWS keys from the team secrets store — never commit them. The repo's own `.env.example` ships these keys blank; copy it to `.env` and fill locally.

## Prerequisites
- AWS CLI v2 + the **Session Manager plugin** (`session-manager-plugin`) on PATH.
- AWS credentials with `ssm:StartSession` on the bastion.
- `mongosh` (for shell) and/or Python `motor`/`pymongo` (for code).

## Fixed facts (verified)
| Thing | Value |
|---|---|
| Bastion EC2 instance | `i-0132f2ba5f7ed8c81` |
| Remote Mongo host:port | `10.0.5.10:27017` |
| Replica set | `tcg-rs` |
| Region | `eu-central-1` (set `AWS_REGION`) |
| Default local port | `27017` (`LOCAL_PORT`) |
| Default auth DB | `admin` (`MONGO_AUTH_SOURCE`) |
| SSM document | `AWS-StartPortForwardingSessionToRemoteHost` |

Source of truth for the command + params: `TCG-software/tcg/core/tunnel.py`; env-var names: `TCG-software/.env.example`.

---

## Step 1 — Open the SSM port-forward tunnel

This is the **exact** invocation TCG-software builds in `tunnel.py` (`SSMTunnel.start`). Run it in its own terminal; it stays open in the foreground.

```bash
export AWS_ACCESS_KEY_ID=...        # from team secrets — do not commit
export AWS_SECRET_ACCESS_KEY=...    # from team secrets — do not commit
export AWS_REGION=eu-central-1

aws ssm start-session \
  --target i-0132f2ba5f7ed8c81 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["10.0.5.10"],"portNumber":["27017"],"localPortNumber":["27017"]}' \
  --region eu-central-1
```

- After "Waiting for connections… / Port 27017 opened", the remote `10.0.5.10:27017` is reachable at `localhost:27017`.
- If local `27017` is taken, change `localPortNumber` (e.g. `27018`) and use that port everywhere below.
- The `--parameters` JSON shape is built verbatim by `tunnel.py`: `{"host":[DB_REMOTE_HOST],"portNumber":[DB_REMOTE_PORT],"localPortNumber":[LOCAL_PORT]}`.

**TCG-software does this automatically.** Set in `.env`:
```ini
SSM_TUNNEL_ENABLED=true
SSM_BASTION_ID=i-0132f2ba5f7ed8c81
DB_REMOTE_HOST=10.0.5.10
DB_REMOTE_PORT=27017
LOCAL_PORT=27017
AWS_REGION=eu-central-1
AWS_ACCESS_KEY_ID=        # fill locally
AWS_SECRET_ACCESS_KEY=    # fill locally
```
On FastAPI startup `tcg/core/tunnel.py` spawns the same `aws ssm start-session`, polls `localhost:LOCAL_PORT` until the TCP connect succeeds, and auto-restarts on drop. Stop = app shutdown (`taskkill /T /F` on Windows to also kill the orphaned `session-manager-plugin`).

---

## Step 2 — Connect with `mongosh`

Tunnel must be up (Step 1). Connect to `localhost`, NOT `10.0.5.10`. Use `directConnection=true` (you're hitting one forwarded member, not the full replica set).

```bash
# Interactive (prompts for password):
mongosh "mongodb://<MONGO_USER>@localhost:27017/?authSource=admin&directConnection=true"

# Or via an env var holding the full URI (recommended — keeps creds out of shell history):
export MONGO_URI='mongodb://<user>:<password>@localhost:27017/?authSource=admin&directConnection=true'
mongosh "$MONGO_URI"
```

Quick checks:
```javascript
show dbs                                   // tcg-instrument, tcg, tcg-raw, tcg-report, tcg-signals, tcg-app-data
use tcg-instrument
db.getCollectionNames()                    // verify FUT_*/OPT_* + INDEX/ETF/FUND/FOREX
db.INDEX.findOne({ _id: "SPX" }, { "eodDatas.YAHOO": { $slice: 1 } })
db["FUT_VIX"].countDocuments()
use tcg-app-data
db["2026-app-data"].find({ type: "indicator" }).limit(3)
```

> The scoped TCG-software write user (`app-writer`) can ONLY touch `tcg-app-data.2026-app-data`; anything else returns `OperationFailure` code 13. Use the admin user for cross-DB browsing.

---

## Step 3 — Connect from Python

TCG-software uses **`motor`** (async, `motor>=3.0`); `pymongo` works the same for sync scripts. URI assembly mirrors `tcg/core/config.py::load_config()` and `tcg/persistence/_client.py`.

URI format (same as the app builds when `SSM_TUNNEL_ENABLED=true`):
```
mongodb://<user>:<password>@localhost:<LOCAL_PORT>/<db>?authSource=admin&directConnection=true
```
User/password are URL-escaped via `quote_plus` in the app.

**Async (motor) — read `tcg-instrument`:**
```python
import os
from urllib.parse import quote_plus
from motor.motor_asyncio import AsyncIOMotorClient

user = quote_plus(os.environ["MONGO_USER"])
pwd  = quote_plus(os.environ["MONGO_PASSWORD"])      # never hardcode
port = os.getenv("LOCAL_PORT", "27017")
uri  = f"mongodb://{user}:{pwd}@localhost:{port}/tcg-instrument?authSource=admin&directConnection=true"

client = AsyncIOMotorClient(uri, directConnection=True, tz_aware=True)
db = client["tcg-instrument"]

async def demo():
    print(await db.list_collection_names())
    doc = await db["INDEX"].find_one({"_id": "SPX"}, {"eodDatas.YAHOO": {"$slice": 1}})
    print(doc)
```

**Sync (pymongo) — quick script:**
```python
import os
from pymongo import MongoClient

uri = os.environ["MONGO_URI"]   # full localhost URI, from env — not committed
c = MongoClient(uri, directConnection=True)
print(c["tcg-instrument"]["FUT_VIX"].estimated_document_count())
```

Client timeouts TCG-software uses (good defaults over a tunnel): `serverSelectionTimeoutMS=30000`, `connectTimeoutMS=60000`, `socketTimeoutMS=300000`, `maxPoolSize=3` (single forwarded port → keep the pool small).

### Reading OHLCV
EOD bars live under `eodDatas.<PROVIDER>` (`YAHOO`, `IVOLATILITY`, `CBOE`, `DERIBIT`, `COINGECKO`, …), each an array of `{date(YYYYMMDD int), open, high, low, close, volume}` sorted ascending. Project out `intradayDatas`/`eodGreeks` (large, unused by the read app):
```python
proj = {"intradayDatas": 0, "eodGreeks": 0}
doc = await db["INDEX"].find_one({"_id": "SPX"}, proj)
bars = doc["eodDatas"]["YAHOO"]
```

---

## Credentials — where they come from (and what NOT to do)
| Need | Env var | Source |
|---|---|---|
| AWS auth for SSM | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` | team secrets store; export locally |
| Mongo read | `MONGO_URI` **or** `MONGO_USER`+`MONGO_PASSWORD` (+`MONGO_AUTH_SOURCE`, `MONGO_DB`) | team secrets store |
| Mongo app-write (scoped) | `MONGO_APP_WRITE_URI` **or** `MONGO_APP_WRITE_USER`+`MONGO_APP_WRITE_PASSWORD` | provisioned by `provision_app_writer.py` (prints nothing) |

⚠ **The Java repo commits plaintext admin URIs in `core.parent/data/.../mongodb-config.properties` (SECURITY TODO).** Do NOT copy those into code, the docs, or this repo — pull live creds from the secrets store and keep them in your local `.env` (gitignored).

## Troubleshooting
| Symptom | Likely cause / fix |
|---|---|
| `port 27017 not reachable within 30s` | tunnel not up / wrong bastion / no SSM perms. Re-run Step 1; check `aws sts get-caller-identity`. |
| `aws: command not found` | install AWS CLI v2 + session-manager-plugin. |
| Port already in use | another tunnel/`session-manager-plugin` running. Kill it (Windows `taskkill /T /F`) or use a different `localPortNumber`. |
| `Authentication failed` | wrong `authSource` (use `admin`) or bad creds. |
| Replica-set/topology errors | add `directConnection=true` — you're connecting to one forwarded member. |
| `OperationFailure` code 13 | you're on the scoped `app-writer` user outside `tcg-app-data.2026-app-data`. Use the admin user. |
