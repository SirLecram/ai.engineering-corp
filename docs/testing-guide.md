# Premium tiers: code overview, setup and testing guide

State as of 2026-10-27. Code is merged on `master` (Libraries, Users) and on the integration branch `feat/premium-tiers` (Api, App, DevOps). Nothing is deployed. The pieces have unit and integration tests, but **no end-to-end run across Gateway, Users services and the app has been done yet** (that is the open Users#8 task).

## 1. What was built

```
Android app --HTTPS--> nginx --> API Gateway (LSC.Gateway) --> Identity (5310)      --> Firebase Admin SDK
   |                     |        tier middleware               Subscriptions (5320) --> RevenueCat REST API
   | RevenueCat SDK      |        (AuditOnly / Enforce)             |                |
   v                     |                                          +--> Postgres 17 (users-db, schemas identity/subscription)
 Google Play             +--/webhooks/revenuecat (only public path to Subscriptions) <-- RevenueCat
```

| Part | Repo / branch | What it does |
|---|---|---|
| IdentityService | Users `master` | Registers users, writes Firebase custom claims `tier`, `tier_exp`, `tier_ver` (version-guarded, merged with existing claims such as `admin`). Internal API guarded by `X-Internal-Key`. CLI `admin grant\|revoke`. |
| SubscriptionService | Users `master` | Receives RevenueCat webhooks (stores, answers 200, processes in background), re-fetches the customer from RevenueCat, computes entitlement, pushes claims to Identity. Endpoints: provision, `me/entitlement`, `sync`. Admin API is NOT built (Users#5). |
| Migrators | Users `master` | One-shot containers running EF Core migration bundles with the high-privilege DB role. Services use a least-privilege role and never migrate. |
| Gateway | Api `feat/premium-tiers` | Validates the Firebase token, enforces the feature->tier map (`streaming.youtube`, `sessions.remote.host` = premium), exposes `/api/v1/features`, `users/me`, `me/entitlement`, `subscriptions/sync`, admin routes. Ships in `AuditOnly` (logs would-be denials, allows). |
| Android app | App `feat/premium-tiers` | Local scoring by default, entitlement cache and feature map, provisioning and sync, 403 handling, RevenueCat purchases (`IPurchasesService`, Fake in Debug), paywall, lock badges on Go Live and Enable remote control, subscription status. |
| Libraries | Libraries `master` | `LSC.Authorization.Tiers(.AspNetCore)` 0.1.0, `LSC.Mediator.Extensions` 0.3.0. |
| DevOps | DevOps `feat/premium-tiers` | nginx + certbot (DuckDNS) config, compose with `users-db`, migrators, services, backups, setup guides `readmes/06-08`. |

## 2. Example execution paths

**A. Free user scores a match (no network).** App starts match -> `LocalScoringBackend` (in memory). No call to the Gateway. `scoring.local` is always unlocked.

**B. Sign-in and provisioning.** Firebase sign-in -> app `UserSessionCoordinator` -> `POST /api/v1/users/me` on the Gateway (Firebase token) -> Gateway calls Identity (create/lookup user) and Subscriptions `provision` (best effort, 3 s) -> returns user + entitlement. App also loads `GET /api/v1/features` (ETag cached, embedded fallback offline) and `GET /api/v1/me/entitlement`.

**C. Purchase unlocks premium.**
1. User taps Go Live (locked badge) -> paywall -> RevenueCat purchase via Google Play.
2. App calls `POST /api/v1/subscriptions/sync` (rate limit 6/min/uid).
3. Subscriptions fetches the customer from RevenueCat, computes tier, calls Identity `PUT /internal/v1/users/{uid}/tier-claims` (version guard; stale = 409 treated as success).
4. Identity writes the Firebase claims. App force-refreshes its ID token, re-reads entitlement, badge disappears.
5. In parallel RevenueCat sends `POST /webhooks/revenuecat` (secret in `Authorization`), stored first, then processed by the background worker with retry/backoff and dead-letter; same result, idempotent by event id.

**D. Locked feature hit directly (Enforce mode).** `POST /api/v1/broadcasts` without premium claim -> Gateway 403 `tier-required` -> app `TierRequiredHandler` runs one sync; if the tier changed it refreshes the token and replays the request once, otherwise it opens the paywall. In `AuditOnly` the Gateway allows the call and only logs/counts `gateway_tier_denied_total{mode="audit"}`.

**E. Manual grant (testing without paying).** RevenueCat dashboard -> Customers -> Grant promotional entitlement `premium` -> webhook or app sync -> claims set. (Gateway admin routes exist, but the Subscriptions admin API behind them is not built yet.) First admin: `docker exec lsc-identity dotnet LSC.Identity.dll admin grant <uid> --reason "<text>"`.

## 3. Prerequisites

**Developer machine**
- .NET SDK 10 (`dotnet --version` 10.x), Docker (Testcontainers + compose), `gh` CLI logged in with `read:packages` (and `write:packages`), Git.
- Android: `dotnet workload install maui-android`, JDK 17, Android SDK 35/36 (needed only to build the app head), an Android device or emulator with Google Play for real purchases.
- NuGet access to the GitHub feed `https://nuget.pkg.github.com/SirLecram/index.json`. The repos' `NuGet.config` reads env var `NUGET_AUTH_TOKEN`:
  `export NUGET_AUTH_TOKEN=$(gh auth token)` (bash) / `$env:NUGET_AUTH_TOKEN = gh auth token` (PowerShell). Other (e.g. corporate) feeds on the machine may break `dotnet restore`; use a scratch `nuget.config` if so.

**GitHub (already done unless noted)**
- Libraries packages readable by Users, Api, App ("Manage Actions access"). **Open:** same for the four Users packages (`LSC.Identity.Dto/.Client`, `LSC.Subscriptions.Dto/.Client`) for Api and App.
- Users repo Actions workflow permissions: read and write.

**Cloud accounts (needed only for real, non-fake runs)**
- Firebase project `livescorecounter-mg` + dedicated service account `lsc-identity` (role Firebase Authentication Admin).
- Google Play Console app with an internal-testing build, a subscription product, license testers.
- RevenueCat project: Android app, entitlement `premium`, offering, public SDK key, secret API key, webhook.
- DuckDNS name `lsc-app` (done), ports 80/443 open in OCI and VM firewall (443 was still timing out), certificate via `scripts/issue-cert.sh` (DevOps readme 06).

Detailed console steps: DevOps `readmes/08-OWNER-ACCOUNT-SETUP.md`.

## 4. Where secrets are needed

| Secret | Used by | Where it goes | Needed for |
|---|---|---|---|
| Postgres role passwords (admin, `identity_svc`, `subscription_svc`, two migrators) | users-db, services, migrators | local: git-ignored `.env`; VM: `scripts/init-users-secrets.sh` generates them (secrets folder per container) | any DB run (local dev uses throwaway values) |
| `Security:InternalServiceKey` (>=32 chars) | Identity, Subscriptions, Gateway | env `Security__InternalServiceKey`; Gateway `Infrastructure__Proxies__<Identity\|Subscriptions>__InternalServiceKey` | service-to-service calls; required in Production |
| Firebase service account JSON | Identity (`Firebase:ServiceAccountPath`) | VM secrets folder file `identity-firebase-sa.json` | real claim writes |
| RevenueCat secret API key (`RevenueCat:SecretApiKey`) | Subscriptions | secrets folder file `RevenueCat__SecretApiKey` | real RevenueCat fetch |
| RevenueCat webhook token (`RevenueCat:WebhookAuthToken`, optional `...Previous`) | Subscriptions + RevenueCat dashboard header | secrets folder; same value in the webhook config; >=32 chars in Production | webhook auth |
| RevenueCat public SDK key (`Billing:RevenueCatPublicApiKey`) | Android app | local `appsettings.Local.json` or release config (public by design, still not committed) | real purchases |
| Test JWKS (`Authentication:TestIssuer:JwksJson`, public) + test private key | Gateway, Identity, Subscriptions in `TestIssuer` mode (one RS256 token passes all three) | generated by `scripts/local-stack/init.sh` into git-ignored `.env` and `.local/test-key.pem` | local testing only; refused in Production |
| DuckDNS token | optional IP-updater cron | `/opt/live-score-counter/secrets/duckdns.env` | only if the VM IP changes |
| `NUGET_AUTH_TOKEN` (GitHub token, `read:packages`) | every restore/Docker build | env var; Docker build uses a BuildKit secret | building |
| Play service account JSON | RevenueCat dashboard only | upload in RevenueCat | RevenueCat reading Play |

No secret is committed anywhere. `Fake`/`TestIssuer` modes exist so nothing above is needed to run the automated tests.

## 5. Testing (works without any real secret)

Set `NUGET_AUTH_TOKEN` first (section 3). Docker must run for the integration tests (they skip silently if it does not).

| What | Command | Notes |
|---|---|---|
| Libraries | `dotnet test src/LSC.Authorization.Tiers/LSC.Authorization.Tiers.slnx`, `dotnet test src/LSC.Mediator.Extensions/LSC.Mediator.Extensions.slnx` | pure unit tests (93 / 75) |
| IdentityService | `cd src/LSC.Identity && dotnet test LSC.Identity.slnx` | ~231 tests, Testcontainers Postgres through the real role bootstrap and migrations; checks the runtime role cannot do DDL |
| SubscriptionService | `cd src/LSC.Subscriptions && dotnet test LSC.Subscriptions.slnx` | ~411 tests incl. webhook idempotency, retry/dead-letter, OpenAPI contract test |
| Gateway | `dotnet test src/LSC.Gateway/LSC.Gateway.slnx` | ~185 tests, real pipeline with `WebApplicationFactory`, test tokens, mocked upstreams (tier 403, AuditOnly, route-bypass variants, rate limit) |
| App | `dotnet test` per project in `tests/` (Domain, Application, Infrastructure, UI.Core, ComponentTests) | ~1,300 tests; paywall states use `FakePurchasesService`. `dotnet build src/LiveScoreCounter.App -f net10.0-android` builds the head |
| Release guard | `dotnet build ... -c Release` with `Billing:Provider` = `Fake` in a shipped appsettings | must fail with `LSCBILL001` |

**Full local stack (recommended for hands-on testing).** Users repo, `docs/local-stack.md`. Commands:
1. `gh repo clone SirLecram/LiveScoreCounter.Api ../LiveScoreCounter.Api && git -C ../LiveScoreCounter.Api checkout feat/premium-tiers` (other path: set `LSC_API_PATH` in `.env`).
2. `scripts/local-stack/init.sh` (PowerShell: `.\scripts\local-stack\init.ps1`) writes `.env` and `.local/test-key.pem`.
3. `export NUGET_AUTH_TOKEN=$(gh auth token)` (PowerShell: `$env:NUGET_AUTH_TOKEN = (gh auth token)`).
4. `docker compose --profile full up --build -d` (if you ran this compose before Users#5, run `docker compose --profile full down -v` first).
5. `scripts/local-stack/smoke.sh` runs the whole walkthrough with assertions (60 checks, both enforcement modes).
6. Tokens: `scripts/local-stack/mint.sh --sub owner-1 [--tier premium --tier-exp <iso>] [--admin]`. Requests: `tools/http/premium-tiers.http` or the Postman collection + `.local/postman.environment.json`.
7. Enforcement mode: `LSC_TIER_ENFORCEMENT_MODE=Enforce docker compose --profile full up -d --no-deps --force-recreate gateway` (or `AuditOnly`).
8. Test-only controls (Development + Fake mode, need `X-Internal-Key`): `PUT/DELETE /dev/fake-revenuecat/subscribers/{uid}`, `POST /dev/fake-revenuecat/webhooks/{uid}`, `GET /dev/fake-firebase/users/{uid}/claims`.
Limits: StreamPlatformManagement, RemoteSessions and the SignalR proxy are not in the stack (a broadcast call that passes the tier gate ends in 400/5xx; the 403 `tier-required` gate is fully shown). Fake Firebase is in memory per process, so the `admin grant` CLI audit row is written but the grant does not reach the running service.

**Manual app testing (no payments):** build Debug with `appsettings.Local.json` `Billing:Provider=Fake` to walk through the paywall states; with a real key and a Play internal-track build, use license testers, or grant a promotional entitlement in RevenueCat.

## 6. Known gaps (be aware when testing)
- No end-to-end test across app, Gateway and Users services (Users#8); the Gateway compose env for the proxies/internal key is not in DevOps compose yet (DevOps#27).
- No CI has run for Api, App, DevOps integration-branch work; the Android build and the real RevenueCat/Play flow were never run on a device.
- Subscriptions admin API, reconciliation job and retention purge are not built (Users#5); Gateway admin routes call them.
- Port 443 and the certificate are not set up; the app already points at `https://lsc-app.duckdns.org/`, so it cannot reach the Gateway until TLS works.
- Gateway is in `AuditOnly`: premium features are not blocked server-side until you set `TierAuthorization__EnforcementMode=Enforce`.
- App leftovers: premium-ended banner, join-entry badge, returning to the paywall after sign-in; the Identity client's provision method returns an empty profile (extra call to `me/entitlement`).

## 7. Step-by-step: from checkpoint to a working test

**Stage 1 - local, no accounts (works today).** Section 5, "Full local stack". Expect `smoke.sh` 60/60. Check both enforcement modes. This validates Gateway + Identity + Subscriptions + database + migrators together with test tokens and a fake RevenueCat.

**Stage 2 - app against the local stack (Debug).** Build the Android Debug head with `Billing:Provider=Fake` (`appsettings.Local.json`), point the Gateway base URL in the same file to your machine (`http://10.0.2.2:5300` for the emulator; cleartext needs `android:usesCleartextTraffic`, see App#148), sign in with a test token flow or Firebase test user. Walk through: local scoring (no network), locked "Go Live" / "Enable remote control" badge, paywall states with the fake service, purchase -> activating -> unlock.

**Stage 3 - deploy to the VM (needs your accounts; order matters).**
1. DNS `lsc-app.duckdns.org` -> VM IP (done), ports 80/443 open in OCI + VM firewall (443 was still timing out), then certificate: DevOps `readmes/06-TLS-DUCKDNS.md`.
2. Merge DevOps `feat/premium-tiers` review PR (#35) FIRST. Then on the VM: `scripts/init-users-secrets.sh`, put owner secrets in the secrets folder (DevOps `readmes/08`, `07`), keep `.env` in fake mode (`LSC_USERS_MODE=Fake`, Staging) for a first boot, `docker compose up -d`, check health. Runbook: `readmes/09-PREMIUM-TIERS-RUNBOOK.md`.
3. Merge Api review PR (#32) (publishes the gateway image; the Gateway needs the two internal keys and `TierAuthorization__AllowAuditOnlyInProduction=true` from compose, otherwise it refuses to start). Keep `AuditOnly`.
4. Merge App review PR (#175) (packages an Android release); install it from the Play internal track.
5. Real mode: add Firebase service account, RevenueCat secret key + webhook token; set `LSC_USERS_MODE=Real`, `LSC_USERS_ENVIRONMENT=Production`; configure the RevenueCat webhook `https://lsc-app.duckdns.org/webhooks/revenuecat` with `Authorization: Bearer <token>`. First admin: `docker exec lsc-identity dotnet LSC.Identity.dll admin grant <uid> --reason "..."`.
6. Test: promotional entitlement in RevenueCat for a tester -> app unlocks; license-tester purchase on the internal track; then switch to `Enforce` per the runbook and re-test the locked flows.

**Troubleshooting.** Gateway refuses to start in Production: missing internal keys or `AllowAuditOnlyInProduction` not set. Services start without DB/RevenueCat config: the secrets directory fix (Users + Api, `/run/secrets`) must be merged first. CI restore 403 in Api/App: grant package access (four Users packages). `http://<vm-ip>/` stops working after DevOps deploy: use the hostname (the transitional port-80 block is being added in the DevOps fix PR).
