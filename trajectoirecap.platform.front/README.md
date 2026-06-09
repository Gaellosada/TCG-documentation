# trajectoirecap.platform.front

## Purpose
Internal Angular SPA — the main operator/trader web UI for the TCG platform: P&L, portfolio (positions/trades/greeks/risk), strategy signal monitoring (~20 strategies), simulator/backtests, data viewer, vol surface, and admin (users, scheduler, RT-data config, instrument mapping, Bloomberg config, backend logs).

## Status
**LIVE prod.** Evidence: AWS CodeCommit `eu-central-1` remote; consumes prod API `api.platform.trajectoirecap.com`; recent commits add real strategies (Cerulean, Lazuli, Ocean, Short-Simple). Sibling app `trajectoirecap.platform.event.front` is a **separate** Angular app (no code reference between the two).

## Stack
| Item | Value |
|------|-------|
| Language | TypeScript ~4.9.4 |
| Framework | Angular 16.0.4 (Material 16, CDK 16) |
| Build tool | Angular CLI 16 (`@angular-devkit/build-angular` 16) |
| Charts | apexcharts 3.37, echarts 5.4 + echarts-gl 2.0 (via ngx-echarts 16) |
| Crypto | crypto-js 4.1 (AES) |
| Other | rxjs 7.5, moment 2.29, @ngx-pwa/local-storage 16, zone.js 0.13 |
| Tests | Karma 6.3 + Jasmine 4.0 (`ng test`); no e2e configured |
| Node types | `@types/node` ^12 (build was generated under older Node) |

Note: repo root `README.md` is stale Angular-CLI boilerplate claiming **CLI 13.3.3** — `package.json` is the truth (Angular 16). 152 component dirs, 19 services, ~710 files.

## Deployment
- **No deploy config committed in this repo.** No Dockerfile, no `buildspec.yml`, no nginx conf, no k8s/helm/kustomize manifests anywhere in the tree (verified). Build output is a static bundle: `ng build` → `dist/trajectoirecap.platform.front/` (production config, `outputHashing: all`, budgets initial 2mb warn/4mb error).
- Per the data audit, **EKS hosts only the web app(s)** (no k8s CronJobs). This repo is the static-asset producer; the actual Docker/nginx/EKS deploy machinery lives **outside this repo** (likely a separate CI/infra repo or CodePipeline) — `TODO: locate deploy pipeline; not in source here`.
- `fileReplacements` swaps `environment.ts` → `environment.prod.ts` for the production build.

## Interactions
Single backend dependency — the **`trajectoirecap.platform.parent`** Spring Boot `svc` module (REST), reached via `environment.apiUrl`:
- Prod: `https://api.platform.trajectoirecap.com`
- Dev: `http://localhost:8080`

Endpoint confirmation: frontend paths match parent `svc` controllers exactly (e.g. `@PostMapping("/token")`, `/rt/list-single`, `/scheduler/update`; controllers `OceanController`, `LazuliController`, `NavyShortController`, `SignalsController`, …).

Endpoint families called (all prefixed by `apiUrl`):
| Path prefix | Use |
|-------------|-----|
| `POST /token`, `POST /refresh-token` | auth / JWT refresh |
| `/pnl/*`, `/pnl-resume`, `/pnl-split/*` | P&L (+ `/download/*` CSV) |
| `/ptf/*`, `/trade/*`, `/sm/override-px`, `/greeks/delta/*` | portfolio viewer |
| `/curve`, `/market-timer[-short|-ns]`, `/put-arbitrage`, `/cobalt`, `/azure`, `/ocean`, `/lazuli`, `/cerulean`, `/short-simple`, `/long-simple`, `/navy-long`, `/navy-short`, `/hvol`, `/midnight`, `/flows`, `/signals` | per-strategy signals |
| `/sapphire/order/*` | sapphire orders/positions |
| `/scheduler/*`, `/user/*`, `/rt/*`, `/instrument-mapping/*` | admin |
| `GET /{strategy}/stream/{token}` | **SSE** (EventSource) live updates — ~10 strategy tabs + `/rt/stream` |

**Auth model:** login `POST /token` with HTTP **Basic** header (`window.btoa(login:password)`) → returns `{token, refreshToken, role, mail}`. `token`/`user`/`role`/`mail` stored in `localStorage`; `refreshToken` stored **AES-encrypted** (crypto-js, see secret below). HTTP interceptor (`interceptor.interceptor.ts`, registered `multi:true` in `app.module.ts`) attaches `Authorization: Bearer <token>` to every request except `/token`+`/refresh-token`; on `401` it calls `/refresh-token` once and retries, else routes to `/login`. Route guards: `AuthGuard`, `LoginGuard`, `AdminGuard` (admin routes need both Auth + Admin).

## Data
**No direct datastore access.** This is a pure browser client — zero Mongo/Postgres connections, no DB drivers in `package.json`. All data flows through the parent REST API above. Client-side persistence is `localStorage` only (`token`, `refreshToken` [AES], `user`, `role`, `mail`).

| Store | Direction |
|-------|-----------|
| MongoDB | none (indirect via parent API) |
| `dwh` Postgres | none (indirect via parent API) |
| Browser `localStorage` | read/write: auth token, encrypted refresh token, user/role/mail |

## Run locally
- Install: `npm install`
- Start dev server: `npm start` (= `ng serve --host=127.0.0.1`, port **4200**). Default serve config is `development`.
- Build prod bundle: `npm run build`
- Unit tests: `npm test` (`ng test`, Karma/Chrome)
- **Config keys** (`src/environments/environment.ts` / `.prod.ts`): `apiUrl`, `tcgSecret`, `tcgIv`, `production`. No `proxy.conf.json`; the dev build hits `apiUrl` (`http://localhost:8080`) directly, so the parent `svc` backend must run locally on 8080 (and allow CORS) for end-to-end dev.

## Gotchas
- **HARDCODED SECRET (SECURITY TODO):** AES key `tcgSecret` and IV `tcgIv` are committed in plaintext in **both** `src/environments/environment.ts` and `src/environments/environment.prod.ts`. They encrypt the stored `refreshToken`. Repo is public — these must be rotated/removed. (Value intentionally not reproduced here.)
- **Stale root README:** claims Angular CLI 13.3.3; actual is Angular 16. Ignore it.
- **Deploy not in repo:** Dockerfile/nginx/EKS manifests are elsewhere — don't expect `ng build` alone to deploy.
- **Prod host has no hyphen:** prod = `api.platform.trajectoirecap.com`. A commented-out line in `environment.ts` shows a hyphenated `api.platform.trajectoire-cap.com` — that variant is **not** the one used.
- **IV is a fixed 16-byte literal** (value in `environment*.ts`, not reproduced here) reused for all AES ops — weak by design.
- **Dead code in source:** commented-out `VixFlowsComponent` route, disabled retry block in the interceptor, `079d56d Disable Azure` commit — several strategy tabs toggle on/off via commits, so the live route/strategy set drifts from what's in the tree.
- **French comments** throughout (e.g. login/interceptor) — codebase is bilingual.
- Strategies are **hardcoded per-component** (one tab/table/chart component set per strategy, ~code-name palette: azure, cobalt, cerulean, lazuli, navy, ocean, midnight, sapphire…); adding a strategy means new components + a new parent controller, not config.
