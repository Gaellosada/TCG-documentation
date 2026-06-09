# trajectoirecap.platform.event.front

## Purpose
Single-page investor event registration form. Presents an investor pitch letter (April 2023 breakfast at Mandarin Hotel Geneva) and collects attendee contact details. One-time event tool; not part of the core trading platform.

## Status
**UNKNOWN (likely inactive/archived)** — the home page content is hardcoded to "April 2023" event; no Dockerfile, no buildspec, no k8s manifest, and no CI/CD pipeline found in this repo. No deploy config references this repo anywhere in `prod-things/`. The backend endpoint it calls (`/event-register` on `api.platform.trajectoire-cap.com`) is still exposed by the live `svc` service (confirmed in `SecurityConfig.java`), but this Angular app itself has no evidence of active deployment.

## Stack

| Item | Value |
|------|-------|
| Language | TypeScript |
| Framework | Angular 15.1 (`@angular/core: ^15.1.0`) |
| Build tool | Angular CLI 15.1.6 (`@angular-devkit/build-angular: ^15.1.6`) |
| Angular Material | 15.2.3 |
| RxJS | 7.8.x |
| TypeScript | 4.9.4 |
| Test runner | Karma 6.4 + Jasmine 4.5 |

## Deployment

**No deploy config found in this repo.** No Dockerfile, no buildspec, no k8s manifest, no AWS CodeBuild pipeline reference. Deploy mechanism is UNKNOWN.

- `ng build` produces `dist/trajectoirecap.platform.event.front/` (static files, production config is the default)
- The `svc` backend serving `/event-register` runs on EKS via `svc-app-deployment` (ECR image `tcg-svc:latest`, port 8080), built by `buildspec-svc.yml` in `trajectoirecap.platform.parent`. That service is separate — this Angular app would need its own hosting (S3 + CloudFront or similar) which is not documented.

## Interactions

| Direction | Target | Details |
|-----------|--------|---------|
| CALLS | `trajectoirecap.platform.parent/svc` | `POST https://api.platform.trajectoire-cap.com/event-register` — unauthenticated (`.permitAll()` in `SecurityConfig.java:55`) |

- No shared code or module dependency with `trajectoirecap.platform.front` (sibling repo running Angular 16 — completely separate app)
- No authentication required for the registration endpoint

## Data

| Direction | DB | Collection | Fields written |
|-----------|-----|-----------|---------------|
| WRITES (via `svc` backend) | `tcg` (Mongo) | `eventRegister` | `mail`, `firstName`, `lastName`, `firm`, `phone`, `registerDate` (set server-side to Europe/Paris `LocalDateTime`) |

- On registration: (1) saves to Mongo `tcg.eventRegister`; (2) sends HTML email to `thai.phan@trajectoirecap.com`, `event@trajectoirecap.com`, `cyril.beriot@trajectoirecap.com` with full registrant list; (3) sends confirmation email to registrant. Email handled by `MailService` in `svc`.
- No direct DB access from Angular — all writes go through `svc` API.

## Routes

| Path | Component |
|------|-----------|
| `/` (redirect) | → `/register` |
| `/register` | `RegisterComponent` — form with firstName, lastName, firm, phone, mail |
| `/register-seccess` | `RegisterSuccessComponent` — success message (note: typo in route name is in code) |
| `/home` | `HomeComponent` — investor pitch letter (unused as default route; accessible directly) |

## Run Locally

```bash
npm install
npm start          # ng serve --host=127.0.0.1 → http://127.0.0.1:4200
# Dev server uses environment.development.ts: apiUrl = http://localhost:8080
# Requires local svc instance or proxy to hit /event-register
```

**Key env/config:**

| File | Key | Dev value | Prod value |
|------|-----|-----------|-----------|
| `src/environments/environment.development.ts` | `apiUrl` | `http://localhost:8080` | — |
| `src/environments/environment.ts` | `apiUrl` | `https://api.platform.trajectoire-cap.com` | same |

Note: `environment.ts` is the **production** file (Angular default config = production). For dev, the `--configuration development` flag swaps in `environment.development.ts`.

No secrets in source code found.

## Gotchas

- **Route typo**: success redirect goes to `/register-seccess` (two `c`s) — this is the actual registered route; do not "fix" without updating both `AppRoutingModule` and `RegisterComponent`.
- **Hardcoded event content**: `HomeComponent` HTML contains the April 2023 event details (date, venue, fund performance numbers) as static text — not data-driven.
- **`environment.ts` is prod config by default**: `angular.json` sets `defaultConfiguration: "production"` for build, so `ng build` (without flags) uses `environment.ts` pointing to `https://api.platform.trajectoire-cap.com`. Running `ng serve` alone uses development config (swaps to `environment.development.ts`).
- **No Dockerfile / deploy pipeline**: this repo cannot be deployed via the same CodeBuild/ECR/EKS path as other platform components without adding infra.
- **`/event-register` endpoint is public** on the live `svc`: no auth token required. Any client can POST to it.
- Audit (`w1-mongodb.md`) lists `eventRegister` collection under `tcg` DB with description "Job/event bookkeeping" — this is the investor event registration collection; unrelated to EventBridge job tracking.
