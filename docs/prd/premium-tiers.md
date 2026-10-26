# PRD: LiveScoreCounter Premium / Free Tiers

| Field | Value |
|---|---|
| Source issue | SirLecram/LiveScoreCounter.App#27 - "Implement a way to grant premium/free access and configure access to the app based on the tier the user has - license, pricing" |
| Tracking branch | `feat/2-premium-tier-specs` |
| Status | Draft v1.1 - written in autonomous mode (no interactive owner interview; see section 11). v1.1: WearOS app removed from scope per owner instruction. |
| Owner | Marcel Grzes (Product Owner / Admin) |
| Billing provider | RevenueCat (entitlement id `premium`) |
| Identity provider | Firebase Authentication |
| Affected repos | `LiveScoreCounter.Users` (NEW: IdentityService, SubscriptionService), API Gateway, `LiveScoreCounter.App` (Android phone app only), `LiveScoreCounter.Libraries` (shared tier-authorization NuGet), `LiveScoreCounter.DevOps` (docs only) |

---

## 1. Summary

LiveScoreCounter is an Android .NET MAUI scoreboard app. Its backend services (API Gateway, Stream Platform Mgmt, Remote Sessions) run behind nginx on an Oracle VM and use Firebase Auth. Today every signed-in user has access to every feature. This PRD adds two tiers:

- **Free**: local scoring only. Works offline and never needs the backend.
- **Premium** (paid subscription through Google Play, billed via RevenueCat): YouTube streaming/broadcast and remote sessions/pairing.

Gating is driven by a **configurable feature -> minimum-tier map**. The tier assignments above are the defaults; the map is not hardcoded. RevenueCat is the source of truth for purchases. The new **SubscriptionService** turns RevenueCat state into an internal entitlement record. The new **IdentityService** mirrors that record into Firebase custom claims. The **API Gateway** enforces the map on each request using only the verified ID-token claims. The Android phone app enforces the same map in the UI and shows a paywall for locked features.

The WearOS companion app is **out of scope** for this feature (section 10).

## 2. Goals and Success Metrics

### 2.1 Goals
- G1. Monetize premium features with a Google Play subscription and no manual work per purchase.
- G2. Enforce on the server. A modified or rooted client must not be able to use premium backend features.
- G3. Change which tier unlocks which feature by editing config and restarting the service, without code changes.
- G4. Never degrade the free experience: local scoring works offline, signed out and with expired tokens.
- G5. Give the owner a safe, audited way to grant and revoke premium manually (support, promotions, testers).
- G6. Build and test everything without real RevenueCat or Firebase credentials (fakes), and keep it ready for real credentials.

### 2.2 Success Metrics

| ID | Metric | Target | Measured by |
|---|---|---|---|
| M1 | Purchase -> premium unlocked in app (end-to-end) | p95 <= 15 s, p99 <= 60 s | App telemetry event `premium_unlocked` minus `purchase_completed` |
| M2 | RevenueCat webhook received -> Firebase claim updated | p95 <= 10 s | SubscriptionService metric `entitlement_sync_latency_seconds` |
| M3 | Webhook events eventually processed (after retries) | >= 99.9 %; dead-letter count 0 per week | `webhook_events_total{status}` |
| M4 | Unauthorized premium access (security test suite, section 9.3) | 0 passing bypass cases | CI security test suite |
| M5 | Paywall view -> purchase conversion | Baseline recorded in first 30 days; target set by owner afterwards | RevenueCat dashboard + app events |
| M6 | Free -> Premium conversion within 30 days of sign-up | Baseline in first 60 days (placeholder target 2 %) | RevenueCat + IdentityService sign-up dates |
| M7 | Refund rate | < 5 % of purchases | RevenueCat |
| M8 | Regressions in free local scoring | 0 (existing scoring test suite stays green; offline test AC-APP-01) | CI |

## 3. Personas

| Persona | Description | Primary need |
|---|---|---|
| P1 Free Scorer | Parent or amateur referee keeping score on one phone, often without connectivity in sports halls. | Scoring always works, with no nagging when offline. |
| P2 Premium Broadcaster | Club volunteer streaming matches to YouTube from a phone, controlling score from a second phone. | Pay once per period, unlock instantly, and have nothing break mid-match. |
| P3 Session Joiner | Helper who joins a premium broadcaster's session from another phone using a pairing code. May be a free user. | Join without buying anything. |
| P4 Lapsed Subscriber | Former premium user whose subscription expired, was cancelled or refunded. | Clear messaging and an easy resubscribe path. |
| P5 Owner/Admin | Product owner handling support, testers, promotions and refunds. | Grant or revoke premium per user with an audit trail. |
| P6 Operator | Runs the Docker stack on the Oracle VM. | Health checks, logs, metrics, alerts, documented config and secrets. |

## 4. Tiers, Pricing and Feature Matrix

### 4.1 Tiers
- Tier ordering: `free` (rank 0) < `premium` (rank 1). The model supports N ranked tiers. Only these two ship in v1.
- A user with no entitlement, an unknown tier or an expired tier is `free`.

### 4.2 Pricing and products (configuration, not code)
- RevenueCat entitlement id: `premium`. RevenueCat offering: `default`.
- Google Play subscription products: `lsc_premium_monthly` and `lsc_premium_annual`. Both attach to entitlement `premium`.
- Prices, regional pricing, free trials and intro offers are set **only** in Google Play Console / RevenueCat. The app always shows the localized price strings returned by the RevenueCat offering. No price appears in code or app config.
- A free trial, if the owner configures one, counts as `premium` for gating. No special code path is required.

### 4.3 Feature matrix (default map)

| Feature key | Description | Default min tier | Enforced where |
|---|---|---|---|
| `scoring.local` | Local scoreboard, scoring, reset, local overlay preview | `free` | Client only (never touches backend) |
| `streaming.youtube` | Create a YouTube broadcast, start an RTMP stream, "Go Live" | `premium` | Gateway (entry routes of Stream Platform Mgmt) + client UI |
| `sessions.remote.host` | Create a remote session, generate a pairing code, pair a controller phone | `premium` | Gateway (entry routes of Remote Sessions) + client UI |
| `sessions.remote.join` | Join an existing session by pairing code from a phone and send score updates as a controller | `free` | Gateway + client UI |

The owner chose "premium = remote sessions/pairing". It is enforced on the **host** (`sessions.remote.host`): the person who creates the session and pairing code pays, and joiners (P3) do not. Section 11, D7, explains why. To require premium for joiners too, change `sessions.remote.join` to `premium` in config.

## 5. Architecture Overview (normative)

```
Android app ──(RevenueCat SDK, appUserID = Firebase UID)──> Google Play / RevenueCat
     │                                                          │ webhook (HTTPS, Authorization header)
     │ Firebase ID token (claims: tier, tier_exp)               ▼
     ▼                                              nginx ──> SubscriptionService ──(REST, refetch)──> RevenueCat API
   nginx ──> API Gateway (verifies JWT + feature map) ──> Stream Platform Mgmt / Remote Sessions
                                                             SubscriptionService ──(internal HTTP)──> IdentityService ──(Admin SDK)──> Firebase custom claims
```

- **SubscriptionService** (`LiveScoreCounter.Users`): receives webhooks, keeps an inbox and an entitlement store, refetches RevenueCat subscriber state, exposes the "me", sync and admin APIs, and runs the reconciliation job.
- **IdentityService** (`LiveScoreCounter.Users`): user registry (Firebase UID, status, timestamps) and the only component that writes Firebase custom claims. It merges claims and keeps the claim audit log.
- **API Gateway**: verifies the Firebase JWT and applies the feature map to configured routes through the shared NuGet `LSC.Authorization.Tiers` (in `LiveScoreCounter.Libraries`). It is the single enforcement point for premium backend features.
- Contracts: OpenAPI 3.1 specs `docs/contracts/identity-service.yaml`, `docs/contracts/subscription-service.yaml` and the feature-map JSON schema `docs/contracts/feature-tier-map.schema.json`. Their creation is a requirement of this PRD (FR-CON-01).

## 6. Functional Requirements

Notation: each requirement `FR-x` has testable acceptance criteria `AC-x`. "MUST" is mandatory for v1.

### 6.1 Feature -> tier map (FR-MAP)

- **FR-MAP-01** The map MUST be a JSON configuration section `TierAuthorization` with: `tiers` (name -> rank), `features` (key -> minTier), and `routes` (list of `{ method, pathPattern, feature }`). It is loaded through the .NET options pattern with validation at startup.
  - AC: Changing `streaming.youtube` to `free` in config and restarting the Gateway lets a free user's `POST` to a streaming entry route succeed, with no code change or rebuild (integration test).
  - AC: Invalid config makes the service **fail to start** with a descriptive error. Invalid means a feature references an undefined tier, a route references an undefined feature, or two tiers share a rank (unit test).
- **FR-MAP-02** Only routes listed in `routes` are tier-gated. Other authenticated routes keep their current authorization. Gating applies to **entry operations**: create broadcast, start stream, create session, generate pairing code, join session. **Continuation operations** stay authorized for the authenticated owner/participant of an already-existing resource: score updates, statistics, stop/end, leave.
  - AC: A user whose tier expires after creating a session can still post score updates to and end that session (integration test).
- **FR-MAP-03** Fail closed. A gated route whose feature resolves to no valid tier is denied for everyone except `admin`, and the denial is logged at Error level.
- **FR-MAP-04** The Gateway MUST expose `GET /api/v1/features` (authenticated). It returns the tiers and feature map (not the route list) with an `ETag`. The app ships an embedded default copy (same content as section 4.3), fetches the server copy at start and on foreground (max once per 15 min), and caches it.
  - AC: The app uses the server map when available, the cached map when offline, and the embedded map on first launch offline. An unknown feature key in the app is treated as locked.

### 6.2 Entitlement model and claims (FR-ENT)

- **FR-ENT-01** SubscriptionService stores one `Entitlement` per Firebase UID with: `tier`, `expiresAt` (UTC, null = no expiry), `source` (`store` | `promotional` | `none`), `willRenew`, `store`, `productId`, `periodType` (`normal` | `trial` | `intro`), `billingIssueDetectedAt`, `revenueCatLastSyncedAt`, `version` (optimistic concurrency).
- **FR-ENT-02** RevenueCat subscriber state is the source of truth. The local entitlement is always **recomputed from a fresh `GET /v1/subscribers/{app_user_id}` call**, never from webhook payload deltas. The user is `premium` iff RevenueCat entitlement `premium` exists and its `expires_date` is null or in the future.
- **FR-ENT-03** IdentityService MUST write these Firebase custom claims:
  - `tier`: `"premium"`, or the claim is removed for free users.
  - `tier_exp`: Unix seconds = RevenueCat `expires_date` + `ExpiryBufferHours` (default 6, configurable). Omitted when there is no expiry.
  - `tier_ver`: the entitlement `version`, a monotonic integer.
  - AC: Setting tier claims MUST **merge** with existing claims (for example `admin`) and never drop them (unit test with a fake Admin SDK).
  - AC: Claims payload stays < 1000 bytes (Firebase limit), asserted in a unit test.
  - AC: A claim write carrying a `tier_ver` lower than the currently stored version is rejected as stale (no overwrite).
- **FR-ENT-04** The Gateway treats a request as `premium` iff the verified ID token has `tier == "premium"` and (`tier_exp` is absent or `tier_exp > now - 300 s` clock skew). Otherwise the request is `free`. No network call is made on the request path.
- **FR-ENT-05** A denied request returns **HTTP 403** with RFC 9457 Problem Details: `type = "https://livescorecounter.app/problems/tier-required"`, `feature`, `requiredTier`, `currentTier`. A missing or invalid token still returns 401.
  - AC: The app maps this 403 to the paywall for that feature (UI test with a mocked HTTP handler).
- **FR-ENT-06** `GET /api/v1/me/entitlement` returns `{ tier, expiresAt, willRenew, source, periodType, billingIssue, managementUrl, features: { key: bool } }` from the SubscriptionService store, never from the token.

### 6.3 User provisioning (FR-USR)

- **FR-USR-01** After every successful sign-in or sign-up, the app calls `POST /api/v1/users/me` (idempotent). IdentityService upserts the user (`uid`, `createdAt`, `lastSeenAt`, `status = active`). SubscriptionService ensures a RevenueCat customer exists for the UID, using the `GET subscriber` call, which creates it.
  - AC: Calling it N times creates exactly one user record (integration test).
- **FR-USR-02** RevenueCat unavailability during provisioning MUST NOT block the user. The user is `active` with tier `free`, and provisioning is retried by the reconciliation job. (This deviates from the diagram note "create in RevenueCat before user active"; see D10.)
- **FR-USR-03** The app MUST call `Purchases.LogIn(firebaseUid)` after Firebase sign-in and `Purchases.LogOut()` on sign-out. Anonymous RevenueCat purchases are not allowed: the paywall requires a signed-in user.
  - AC: A webhook whose `app_user_id` starts with `$RCAnonymousID:` is stored, marked `ignored_anonymous`, acknowledged with 200, and does not change any claim.

### 6.4 Webhook ingestion (FR-WH)

- **FR-WH-01** Endpoint `POST /webhooks/revenuecat` on SubscriptionService, exposed through nginx on that exact path only.
- **FR-WH-02 Security.**
  - The request MUST carry `Authorization: Bearer <secret>`. The secret is configured in the RevenueCat dashboard and in the `RevenueCat__WebhookAuthToken` secret. It is compared in constant time.
  - Missing or wrong secret -> 401, nothing persisted, warning logged (without the header value).
  - Body > 256 KB -> 413. Non-JSON or schema-invalid body -> 400.
  - nginx rate limit is 20 req/s with burst 50 on this path.
  - The service MUST refuse to start in `Production` if the token is empty or shorter than 32 characters.
  - AC: Each case above is covered by an integration test.
- **FR-WH-03 Environment.** Events with `environment == "SANDBOX"` are processed only when `RevenueCat__AcceptSandbox == true` (default true in Development/Staging, false in Production). Otherwise they are stored as `ignored_sandbox` and acknowledged with 200.
- **FR-WH-04 Idempotency.** The inbox table has a unique key on `event.id`.
  - A duplicate delivery returns 200 and is not reprocessed (metric `status="duplicate"`).
  - AC: Posting the same event 5 times (sequentially and concurrently) produces exactly 1 inbox row and at most 1 claim write.
- **FR-WH-05 Acknowledge fast, process async.** The handler persists the raw event and returns 200 within p95 500 ms. Processing then runs in a background worker: refetch RevenueCat -> recompute entitlement -> request claim update from IdentityService -> write audit.
  - Processing for the same `app_user_id` is serialized (per-user lock or optimistic concurrency with retry).
- **FR-WH-06 Retry and dead-letter.** Failures with the RevenueCat API, IdentityService or Firebase are retried with exponential backoff (1 s, 5 s, 30 s, 2 min, 10 min, 30 min, 1 h, 3 h; 8 attempts). After that the event becomes `dead_letter`, an Error log is emitted and `webhook_dead_letter_total` is incremented.
  - AC: A dead-lettered event can be replayed through an admin endpoint (FR-ADM-04).
- **FR-WH-07 Event coverage.** Every RevenueCat event type is accepted. Because the logic is "refetch and recompute" (FR-ENT-02), these need no type-specific branching: `INITIAL_PURCHASE`, `RENEWAL`, `PRODUCT_CHANGE`, `CANCELLATION`, `UNCANCELLATION`, `NON_RENEWING_PURCHASE`, `EXPIRATION`, `BILLING_ISSUE`, `SUBSCRIPTION_PAUSED`, `SUBSCRIPTION_EXTENDED`, `TEMPORARY_ENTITLEMENT_GRANT`, `REFUND_REVERSED`, `TEST`.
  - `TRANSFER`: refetch **both** `transferred_from` and `transferred_to` users.
  - `TEST`: logged and acknowledged, no claim change.
  - Unknown future types: stored, refetch performed if `app_user_id` is known, acknowledged with 200.
  - Each type has a fixture-based test.
- **FR-WH-08 Audit.** Every entitlement change appends a row to `entitlement_audit`: uid, old/new tier, old/new expiresAt, cause (`webhook:<eventType>:<eventId>`, `sync`, `reconcile`, `admin:<adminUid>`), timestamp, correlation id. The audit table is append-only (no update/delete in code).

### 6.5 Lifecycle flows (FR-FLOW)

Each flow below is an end-to-end component test using the fake RevenueCat client and the fake Firebase Admin client.

| ID | Flow | Trigger | Required result |
|---|---|---|---|
| FR-FLOW-01 | Grant (purchase) | `INITIAL_PURCHASE` webhook **or** app `POST /api/v1/subscriptions/sync` | Entitlement `premium`, claims set, audit row. Whichever arrives first wins; the second is a no-op (same version). |
| FR-FLOW-02 | Renewal | `RENEWAL` | `expiresAt` and `tier_exp` extended. No interruption: the gateway never sees an expired `tier_exp` thanks to the 6 h buffer. |
| FR-FLOW-03 | Cancel (auto-renew off) | `CANCELLATION` (reason != `CUSTOMER_SUPPORT`) | Stays `premium` until `expiresAt`. `willRenew=false`. App shows "Premium until <date>". |
| FR-FLOW-04 | Expiry | `EXPIRATION` or reconciliation finds `expiresAt` in the past | Tier `free`, `tier`/`tier_exp` claims removed. |
| FR-FLOW-05 | Refund / revoke | `CANCELLATION` with `cancel_reason == CUSTOMER_SUPPORT` (Play refund) | RevenueCat refetch shows no active entitlement -> `free` immediately. Revocation latency is bounded by ID-token lifetime (<= 60 min, see D5). |
| FR-FLOW-06 | Billing issue / grace | `BILLING_ISSUE` | Premium kept while RevenueCat reports the entitlement active (Google grace period). `billingIssue=true` -> app shows a "Fix payment" banner with the Play management link. |
| FR-FLOW-07 | Pause | `SUBSCRIPTION_PAUSED` | Premium until the current `expiresAt`, then `free` (RevenueCat state drives it). |
| FR-FLOW-08 | Uncancel / resubscribe | `UNCANCELLATION`, `INITIAL_PURCHASE` after lapse | `willRenew=true` / `premium` restored. |
| FR-FLOW-09 | Promotional grant | Admin API (FR-ADM-01) or RevenueCat dashboard grant | Entitlement `premium`, `source=promotional`, expiry per duration (lifetime -> no `tier_exp`). |
| FR-FLOW-10 | Product change | `PRODUCT_CHANGE` (monthly <-> annual) | Tier unchanged; `productId`/`expiresAt` updated per refetch. |
| FR-FLOW-11 | Missed webhook | Reconciliation job | Every `ReconcileIntervalMinutes` (default 360) the job refetches RevenueCat for (a) users with `premium` and `expiresAt < now + 24 h`, (b) users with failed provisioning, (c) users with `revenueCatLastSyncedAt` older than 7 days and tier `premium`. Drift is corrected and audited with cause `reconcile`. |

- **FR-FLOW-12 In-progress activities.** Expiry or revocation MUST NOT terminate an already running stream or session server-side; FR-MAP-02 continuation operations stay allowed. The app MUST NOT stop a running stream when the tier changes. It shows a non-blocking banner "Premium ended - this session continues; new sessions require Premium".

### 6.6 App-triggered sync (FR-SYNC)

- **FR-SYNC-01** `POST /api/v1/subscriptions/sync` (authenticated, own UID only) synchronously refetches RevenueCat, recomputes the entitlement, updates claims and returns the FR-ENT-06 body.
  - p95 <= 3 s.
  - Rate limit 6 requests/min per UID; excess -> 429 with `Retry-After`.
- **FR-SYNC-02** The app calls sync after: a successful purchase, a successful restore, a RevenueCat SDK `CustomerInfo` update whose `premium` active state differs from the cached state, and the app returning to foreground when the cached `expiresAt` has passed. After a sync that changed the tier, the app MUST force-refresh the Firebase ID token (`forceRefresh: true`) before calling any gated route.

### 6.7 Android phone app: paywall and gating UX (FR-APP)

- **FR-APP-01** An `IEntitlementService` (Application layer interface, Infrastructure implementation) exposes `CurrentTier`, `IsFeatureUnlocked(featureKey)` and `EntitlementChanged`. ViewModels use only this service, never RevenueCat types directly.
- **FR-APP-02** An `IPurchasesService` abstraction wraps RevenueCat. It has two implementations:
  - `RevenueCatPurchasesService` (real).
  - `FakePurchasesService`, selectable by config `Billing:Provider = Fake`. Allowed only in Debug builds and enforced by a build-time check. It simulates success, user-cancel, pending, error and already-owned outcomes.
- **FR-APP-03 Locked entry points.** "Go Live / Start broadcast" and "Create remote session / Pair device" show a lock badge when the feature is locked. Tapping opens the paywall; it does not show a disabled state with no explanation.
- **FR-APP-04 Paywall screen** shows:
  - The premium benefits list, generated from feature keys with min tier `premium`.
  - Packages from the RevenueCat `default` offering with localized price and period.
  - Buttons: Subscribe, Restore purchases, and Close.
  - Links to Terms and Privacy Policy, plus the Google Play auto-renew disclosure text.
- **FR-APP-05 Paywall states.**

| State | Behaviour |
|---|---|
| Not signed in | Sign-in screen first, then return to the paywall. |
| Loading offerings | Skeleton; Subscribe disabled. |
| Offerings unavailable / offline | Message "Can't load plans. Check connection." + Retry. No crash. |
| Purchase in progress | All buttons disabled; back navigation blocked until the result. |
| User cancelled | Stay on paywall, no error toast. |
| Pending payment (Play pending) | Message "Payment pending - Premium unlocks when confirmed." No unlock. |
| Success | Spinner "Activating..." -> sync (FR-SYNC-02) -> token refresh -> close paywall and continue to the originally requested feature. If activation exceeds 30 s: "Purchase received, activation is taking longer - we'll unlock automatically", and background retry with backoff for up to 10 min. |
| Store error | Show RevenueCat's readable error + Retry. |
| Purchase owned by another account | "This subscription belongs to another LiveScoreCounter account. Sign in with that account." (see D9) |

- **FR-APP-06 Restore.** Restore calls RevenueCat `RestorePurchases` then sync. If nothing is restored, show "No active subscription found for this Google account."
- **FR-APP-07 Manage subscription.** The Settings/Account screen shows tier, renewal or expiry date, billing-issue banner, and "Manage subscription", which opens the RevenueCat `managementURL` (Google Play subscriptions). There is no in-app cancel.
- **FR-APP-08 Offline and expired token.**
  - Cached entitlement (`tier`, `expiresAt`, `lastVerifiedAt`) lives in SecureStorage.
  - On start the UI uses the cached value, then refreshes in the background.
  - If cached `expiresAt` < device time, the UI treats the user as `free`.
  - Local scoring (`scoring.local`) MUST work with no network, no signed-in user and an expired or unrefreshable token.
  - When offline, premium entry points show "Requires internet connection" instead of the paywall, because premium features need the network.
  - The paywall is shown only when the server confirms `free` (or a 403 tier-required response is received).
- **FR-APP-09 Server is authoritative.** If the UI thinks a feature is unlocked but the server returns 403 tier-required, the app runs sync once and retries the request once. If it is still 403, the app shows the paywall.
- **FR-APP-10 Sign-out.** Sign-out clears the cached entitlement and feature map overrides and calls `Purchases.LogOut()`. The next user starts from server state.

### 6.8 Admin / manual grant (FR-ADM)

- **FR-ADM-01** `POST /api/v1/admin/users/{uid}/grants` with body `{ duration, reason }`.
  - `duration` is one of RevenueCat promotional durations: `daily`, `three_day`, `weekly`, `monthly`, `two_month`, `three_month`, `six_month`, `yearly`, `lifetime`.
  - `reason` is 5-500 characters.
  - The handler calls the RevenueCat promotional entitlement API for entitlement `premium`, then syncs (FR-SYNC-01).
- **FR-ADM-02** `POST /api/v1/admin/users/{uid}/grants/revoke` with `{ reason }` revokes **promotional** entitlements only. Store purchases cannot be revoked here (use a Play Console refund). The response states whether the user is still premium through a store purchase.
- **FR-ADM-03** `GET /api/v1/admin/users/{uid}/entitlement` returns the entitlement, the last 50 audit rows and the last 20 webhook events for that UID.
- **FR-ADM-04** `POST /api/v1/admin/webhooks/{eventId}/replay` re-queues a `dead_letter` or `failed` event. `POST /api/v1/admin/users/{uid}/resync` forces a refetch.
- **FR-ADM-05** Every admin endpoint requires the Firebase custom claim `admin == true`; otherwise -> 403. The admin claim is bootstrapped only through an IdentityService CLI command run inside the container (`dotnet IdentityService.dll admin grant <uid>` / `admin revoke <uid>`). There is no HTTP endpoint that grants `admin`.
  - Every admin action is audited with the admin UID and reason.
  - AC: A non-admin token gets 403 on every admin route (parameterized test).
- **FR-ADM-06** There is no admin UI in v1. Swagger UI is enabled only in Development. A DevOps doc gives `curl` examples.

### 6.9 Contracts, delivery and environments (FR-CON / FR-DEL)

- **FR-CON-01** OpenAPI 3.1 specs for every new endpoint (sections 6.1-6.8) and the feature-map JSON schema MUST be committed under `/docs/contracts/` before implementation PRs merge. Webhook payload fixtures are stored as test resources.
- **FR-DEL-01** New repo `LiveScoreCounter.Users` contains:
  - .NET (same major version as the App, .NET 10), Clean Architecture per service (Domain / Application / Infrastructure / Api) and EF Core.
  - PostgreSQL by default (see A4). One database with separate schemas `identity` and `subscription`; each service owns its schema.
- **FR-DEL-02** Fakes:
  - `IRevenueCatClient` -> `FakeRevenueCatClient` (in-memory, scriptable subscriber states).
  - `IFirebaseAdmin` -> `FakeFirebaseAdmin` (in-memory claims).
  - Selected by `RevenueCat__Mode=Fake` and `Firebase__Mode=Fake`. Fake mode is refused at startup when `ASPNETCORE_ENVIRONMENT=Production`.
- **FR-DEL-03** Tests: unit tests for Domain/Application, component tests for flows FR-FLOW-01..12 using fakes and Testcontainers PostgreSQL, and contract tests that check the API against the OpenAPI specs. Line coverage >= 80 % for the Domain and Application projects.
- **FR-DEL-04** CI (GitHub Actions) runs on PR: restore, build, test, publish coverage, and `docker build` for both services. Images are tagged with commit SHA. **No deployment step** to the live VM.
- **FR-DEL-05** Docker: one multi-stage Dockerfile per service, non-root user, `HEALTHCHECK`. A `docker-compose.yml` for local dev includes PostgreSQL and both services in fake mode.
- **FR-DEL-06** DevOps docs (PR to `LiveScoreCounter.DevOps`, docs and compose snippets only) cover:
  - Service registration per guide 05-MANAGING-SERVICES.
  - nginx locations: `/webhooks/revenuecat` -> SubscriptionService; `/api/v1/users`, `/api/v1/me`, `/api/v1/subscriptions`, `/api/v1/admin` -> through the API Gateway.
  - Secrets list, RevenueCat dashboard setup (webhook URL + auth header, entitlement, offering, transfer behaviour), and the switch from fakes to real credentials.
  - The architecture diagram update.

## 7. User Stories (Given-When-Then)

**US-01 Free scoring offline (P1)**
Given I am not signed in and the device is in airplane mode,
When I open the app and score a match,
Then scoring, reset and the local scoreboard work fully, and no paywall, sign-in prompt or network error is shown.

**US-02 Discover premium (P1)**
Given I am signed in, online and on tier free,
When I tap "Go Live",
Then the paywall opens with premium benefits and the localized monthly and annual prices.

**US-03 Purchase and instant unlock (P2)**
Given I am on the paywall, signed in,
When I complete the Google Play purchase,
Then within 15 s (p95) the paywall closes, "Go Live" proceeds, and the backend accepts my create-broadcast request.

**US-04 Webhook arrives before or after sync (system)**
Given a purchase was completed,
When the `INITIAL_PURCHASE` webhook and the app's sync call arrive in either order or concurrently,
Then the user ends up `premium` with exactly one effective entitlement version change and one audit row per actual change.

**US-05 Cancel keeps access until period end (P2)**
Given I am premium and turn off auto-renew in Google Play,
When the `CANCELLATION` webhook is processed,
Then I remain premium until the paid period ends, and the Account screen shows "Premium until <date>, will not renew".

**US-06 Expiry (P4)**
Given my subscription period ended without renewal,
When the `EXPIRATION` webhook is processed or reconciliation runs,
Then my claims are removed. Within at most 60 min plus the 6 h buffer after `expiresAt`, gated entry routes return 403 tier-required, and the app shows the paywall on locked features.

**US-07 Refund (P4/P5)**
Given I am premium and receive a refund via Google Play,
When the `CANCELLATION` (CUSTOMER_SUPPORT) webhook is processed,
Then my entitlement becomes free immediately in SubscriptionService and claims are removed, and new gated requests are denied once my current ID token is refreshed (<= 60 min).

**US-08 Mid-match expiry (P2)**
Given I am live streaming with a remote session,
When my premium expires or is revoked,
Then the stream and session continue until I end them, and I see a non-blocking banner. Starting a new stream or session requires premium.

**US-09 Phone joins premium host (P3)**
Given a premium host created a session and shows a pairing code,
When a free user joins with the code from their phone,
Then the join succeeds and score updates from the controller phone are accepted.

**US-10 Free host tries to create a session (P1)**
Given I am free,
When I try to create a remote session or pair a controller phone,
Then the paywall opens, and if the call reaches the backend it returns 403 tier-required.

**US-11 Admin promotional grant (P5)**
Given I have the admin claim,
When I POST a `monthly` grant with a reason for a UID,
Then that user becomes premium for one month, the audit log records my UID and reason, and the user's app unlocks after its next sync or token refresh.

**US-12 Forged webhook (attacker)**
Given an attacker sends a well-formed RevenueCat payload without the correct Authorization header,
When it reaches `/webhooks/revenuecat`,
Then it gets 401, nothing is persisted and no claim changes.

**US-13 Tampered client (attacker)**
Given a modified app that unlocks all UI locally while the token has no `tier` claim,
When it calls a gated entry route,
Then the Gateway returns 403 tier-required.

**US-14 Config change (P5/P6)**
Given the operator changes `sessions.remote.join` to `premium` in Gateway config and restarts,
When a free user tries to join a session,
Then the request returns 403 tier-required, and the app (after fetching `/api/v1/features`) shows the join entry point as locked.

**US-15 Billing issue (P2)**
Given my renewal payment failed and Google grace period is active,
When `BILLING_ISSUE` is processed,
Then I remain premium during grace and see a "Fix payment" banner linking to Google Play.

## 8. Acceptance Criteria Checklist

- [ ] AC-MAP-01 Feature map changes via config + restart only; invalid config fails startup (FR-MAP-01).
- [ ] AC-MAP-02 Only listed entry routes gated; continuation ops work after expiry (FR-MAP-02, FR-FLOW-12).
- [ ] AC-MAP-03 Unresolvable feature -> deny + Error log (FR-MAP-03).
- [ ] AC-MAP-04 `/api/v1/features` served with ETag; app uses server -> cache -> embedded fallback (FR-MAP-04).
- [ ] AC-ENT-01 Entitlement always recomputed from RevenueCat refetch (FR-ENT-02).
- [ ] AC-ENT-02 Claims merged (admin preserved), < 1000 bytes, stale `tier_ver` rejected (FR-ENT-03).
- [ ] AC-ENT-03 Gateway decides from token only, honours `tier_exp` with 300 s skew (FR-ENT-04).
- [ ] AC-ENT-04 403 Problem Details with `feature`, `requiredTier`, `currentTier` (FR-ENT-05).
- [ ] AC-USR-01 `POST /users/me` idempotent; RevenueCat outage does not block (FR-USR-01/02).
- [ ] AC-USR-02 Anonymous RevenueCat IDs ignored (FR-USR-03).
- [ ] AC-WH-01 401/413/400 for bad auth/size/body; prod refuses weak or empty token (FR-WH-02).
- [ ] AC-WH-02 Sandbox events honoured per environment flag (FR-WH-03).
- [ ] AC-WH-03 Duplicate event id -> 1 row, <= 1 claim write, including concurrent delivery (FR-WH-04).
- [ ] AC-WH-04 Ack p95 <= 500 ms; per-user serialized processing (FR-WH-05).
- [ ] AC-WH-05 Backoff, dead-letter after 8 attempts, replay works (FR-WH-06, FR-ADM-04).
- [ ] AC-WH-06 Fixture test per event type incl. TRANSFER both users, TEST, unknown type (FR-WH-07).
- [ ] AC-WH-07 Append-only audit for every entitlement change (FR-WH-08).
- [ ] AC-FLOW-01..12 Component tests pass for every flow in section 6.5.
- [ ] AC-SYNC-01 Sync p95 <= 3 s, 6/min/UID rate limit, 429 with Retry-After (FR-SYNC-01).
- [ ] AC-SYNC-02 App force-refreshes ID token after tier change before calling gated routes (FR-SYNC-02).
- [ ] AC-APP-01 Local scoring fully functional offline, signed out, expired token (FR-APP-08, US-01).
- [ ] AC-APP-02 Every paywall state in FR-APP-05 covered by ViewModel tests using `FakePurchasesService`.
- [ ] AC-APP-03 Fake billing provider impossible in Release builds (FR-APP-02).
- [ ] AC-APP-04 403 tier-required -> one sync+retry -> paywall (FR-APP-09).
- [ ] AC-APP-05 Restore, Manage subscription, billing banner, "Premium until" shown correctly (FR-APP-06/07, FR-FLOW-03/06).
- [ ] AC-APP-06 Sign-out clears entitlement cache and calls `Purchases.LogOut` (FR-APP-10).
- [ ] AC-APP-07 No changes in the WearOS app project (diff check in PR review).
- [ ] AC-ADM-01 Grant/revoke/inspect/resync/replay work and are audited with admin UID and reason (FR-ADM-01..04).
- [ ] AC-ADM-02 Non-admin -> 403 on all admin routes; no HTTP path grants admin (FR-ADM-05).
- [ ] AC-CON-01 OpenAPI 3.1 specs + feature-map schema in `/docs/contracts/`; contract tests pass (FR-CON-01).
- [ ] AC-DEL-01 CI green: build, tests, coverage >= 80 % Domain/Application, Docker images built; no deploy step (FR-DEL-03/04).
- [ ] AC-DEL-02 `docker compose up` runs both services in fake mode locally; fake mode refused in Production (FR-DEL-02/05).
- [ ] AC-DEL-03 DevOps docs PR covers nginx, secrets, RevenueCat dashboard setup, credential switch, diagram (FR-DEL-06).
- [ ] AC-NFR-* All non-functional requirements in section 9 verified.

## 9. Non-Functional Requirements

### 9.1 Performance and capacity
- NFR-01 Gateway tier check adds p95 <= 2 ms (in-memory claim + map evaluation, no I/O).
- NFR-02 `GET /api/v1/me/entitlement` p95 <= 300 ms. `POST /subscriptions/sync` p95 <= 3 s. Webhook ack p95 <= 500 ms.
- NFR-03 Capacity target on the single Oracle VM: 50 webhook events/s burst for 1 min with no loss, and 200 RPS on gated gateway routes. Verified by a k6 (or equivalent) script committed to the repo, run manually and not in CI.

### 9.2 Reliability
- NFR-04 Webhook processing is durable: an event acknowledged with 200 is persisted before the response and survives a service restart (test: kill the worker mid-processing, restart, event completes).
- NFR-05 If SubscriptionService or IdentityService is down, the Gateway keeps authorizing from tokens. Free and existing-premium users are unaffected until token refresh. Gated features never depend on the Users services being reachable at request time.
- NFR-06 Health endpoints `/health/live` and `/health/ready` (ready checks DB; RevenueCat and Firebase reported as degraded, not unhealthy).

### 9.3 Security
- NFR-07 Secrets are supplied only via environment variables or Docker secrets and are never committed:
  - RevenueCat secret API key (v1, server only).
  - Webhook auth token.
  - Firebase service-account JSON.
  - Internal service key.
  - DB password.
  - CI MUST run secret scanning (gitleaks or GitHub secret scanning).
  - The app contains only the RevenueCat **public** Android SDK key.
- NFR-08 SubscriptionService -> IdentityService calls are on the internal Docker network only (IdentityService is not exposed via nginx). They are authenticated by the `X-Internal-Key` header, compared in constant time.
- NFR-09 The Gateway validates Firebase ID tokens: signature (Google JWKS, cached), `iss`, `aud` = project id, `exp`, `iat`, `sub` non-empty. Tokens are never logged.
- NFR-10 Security test suite (CI) covers at least:
  - Forged webhook.
  - Replayed webhook.
  - Token without claim on gated route.
  - Token with `tier=premium` but expired `tier_exp`.
  - Token signed by the wrong key.
  - Non-admin on admin routes.
  - Sync for another UID (impossible by design: the UID is taken from the token).
  - Oversized payload.
- NFR-11 Data minimization: SubscriptionService stores UID, product, dates, store and event payloads, and no email or name. RevenueCat subscriber attributes are not populated with PII. Raw webhook payloads are purged after 90 days (scheduled job), and audit rows are kept 2 years.

### 9.4 Observability
- NFR-12 Structured logging (Serilog, JSON) with correlation id propagated across Gateway -> services and including the webhook `event.id`. Logs contain UID but never tokens, the webhook secret or full payment details.
- NFR-13 Metrics (Prometheus format at `/metrics`, internal only):
  - `webhook_events_total{type,status}`, `webhook_dead_letter_total`, `webhook_processing_seconds`.
  - `entitlement_sync_latency_seconds`, `entitlement_changes_total{from,to,cause}`.
  - `revenuecat_api_requests_total{status}`, `firebase_claim_writes_total{status}`.
  - `gateway_tier_denied_total{feature}`, `reconcile_drift_corrected_total`.
- NFR-14 Alert conditions are documented in DevOps docs:
  - Dead-letter > 0.
  - Webhook 401 rate > 10/min (possible attack or misconfiguration).
  - RevenueCat API error ratio > 20 % over 10 min.
  - Reconcile drift > 5 per run.
  - Wiring alerts into a pager is out of scope.

### 9.5 Compliance
- NFR-15 The paywall complies with Google Play subscription policy: clear price, period, auto-renew and cancellation disclosure, and the trial terms shown when a trial exists.
- NFR-16 Terms of Service and Privacy Policy URLs are configurable app settings. Content is supplied by the owner (open item O3).

## 10. Out of Scope (v1)

- **WearOS app** (owner instruction). There are no code, UI, test or contract changes in the WearOS project: no paywall, no tier messaging, no entitlement cache.
  - Server-side gating (Gateway) is client-agnostic. Any WearOS call to a gated entry route is subject to the same map, but WearOS behaviour is neither specified nor tested in v1.
  - A follow-up issue should define WearOS behaviour (see O6).
- iOS / App Store, web or Stripe purchases. One-time "lifetime" store products (lifetime is possible only via admin promotional grant).
- More than two tiers in the product. Per-feature add-on purchases. Usage-based limits (for example "3 free streams/month").
- Admin web UI or dashboard. Customer self-service refunds. In-app cancellation.
- Deployment to the live Oracle VM, production RevenueCat/Firebase credential provisioning, and production DNS/TLS changes (docs only).
- Account deletion and GDPR erasure flow across Firebase/RevenueCat (tracked as follow-up issue; required before public launch in the EU, see O5).
- Family sharing, team/club shared subscriptions, seat-based licensing.
- Killing in-progress streams or sessions on expiry. Instant (< token lifetime) revocation through refresh-token revocation or a deny-list.
- Paywall A/B testing (RevenueCat Experiments), localization beyond the app's existing languages, push notifications about billing.
- Changes to the YouTube OAuth or RTMP pipeline beyond adding the gate in front of "Go Live".

## 11. Assumptions & Decisions

### 11.1 Decisions (made autonomously unless marked as owner instruction)

| ID | Decision | Rationale |
|---|---|---|
| D1 | RevenueCat is the single source of truth. Each event triggers a **refetch** of `GET /v1/subscribers/{id}`, and the payload is not applied as a delta. | Removes webhook ordering problems, makes idempotency trivial and covers all event types (including future ones) with one code path. |
| D2 | Claim schema is `tier` (string) + `tier_exp` + `tier_ver` instead of the boolean `premium` claim proposed in the issue #27 plan. | The configurable map needs ranked tiers. `tier_exp` lets the Gateway expire access without a webhook, and `tier_ver` blocks stale writes. Deliberate deviation from the issue plan. RevenueCat entitlement id stays `premium`. |
| D3 | Only IdentityService writes Firebase claims. SubscriptionService asks it to. | Matches the architecture diagram ("Subscription Service requests permission changes"; "Identity Service: roles/claims management") and gives one place for claim merging. |
| D4 | The API Gateway is the enforcement point. It decides from the token only, through a shared NuGet `LSC.Authorization.Tiers`. | Matches the diagram ("API Gateway ... validates subscription status"), adds no per-request network calls, and Users-service outages don't affect streaming. |
| D5 | Revocation lag (refund/admin revoke) is accepted as <= ID-token lifetime (60 min). Natural expiry gets a 6 h `tier_exp` buffer. | Avoids forcing re-login on every device (refresh-token revocation) and avoids a deny-list. The buffer absorbs renewal-webhook delays. The commercial risk (<= 1 h of streaming after a refund) is negligible. |
| D6 | Gating applies to entry operations only; running streams and sessions are never killed. | Cutting a live match broadcast is the worst UX for P2. Continuation is cheap. |
| D7 | Remote sessions are gated on the host (`sessions.remote.host`); joining (`sessions.remote.join`) is free by default. | The paying broadcaster owns the session, and helpers should not each need a subscription. Config can flip it (US-14). |
| D8 | App-triggered `POST /subscriptions/sync` in addition to webhooks. | Meets M1 (<= 15 s unlock) without depending on webhook latency. |
| D9 | RevenueCat transfer behaviour is "Keep with original App User ID". Purchases require a signed-in Firebase user (no anonymous purchases). | Prevents one Play purchase from unlocking many accounts. Support cases go through admin promotional grants. |
| D10 | Provisioning does not block on RevenueCat. The user is active as `free` immediately and the RevenueCat customer creation is retried. | Free scoring needs no backend, and a third-party outage must not block sign-up. Deviation from the diagram note "create in RevenueCat before changing user to active". |
| D11 | Admin grants use RevenueCat promotional entitlements, not a local override table. | Keeps one source of truth (D1), and the RevenueCat SDK in the app sees the grant too. |
| D12 | Admin role is the Firebase claim `admin`, bootstrapped only by CLI inside the container. | No HTTP privilege-escalation path. |
| D13 | Prices live only in Play Console / RevenueCat. Products are monthly + annual. | Price changes need no release. "License" in issue #27 means the entitlement/licence-to-use, not the source code licence. |
| D14 | PostgreSQL with one DB and per-service schemas for `LiveScoreCounter.Users` (see A4). | Low operational cost on a single VM while keeping ownership boundaries. |
| D15 | The feature map lives in Gateway config and is published to the app via `/api/v1/features`. The app embeds a fallback copy. | One authoritative definition; the app works offline and on first launch. |
| D16 | **Owner instruction:** the WearOS app is ignored for this feature. Client scope is the Android phone app only. | Explicit owner direction during drafting. Server-side gating is unchanged and client-agnostic. |

### 11.2 Assumptions (to be validated)

| ID | Assumption | Impact if false |
|---|---|---|
| A1 | The issue #27 comment plan matches the caller's description (RevenueCat, entitlement `premium`, webhook -> backend -> Firebase claim, `LogIn(firebaseUid)`). The issue could not be read directly in this run (no shell/`gh` access). | Re-check the issue and reconcile D2 if the comments specify more. |
| A2 | All premium backend entry operations, including Remote Sessions SignalR negotiate/hub connect for session creation and pairing, are routed through the API Gateway. | If nginx routes any premium endpoint directly to a service, that service MUST also reference `LSC.Authorization.Tiers` and apply the same policy. This becomes a mandatory task. |
| A3 | The API Gateway is a .NET service that can consume a NuGet from `LiveScoreCounter.Libraries`. | Otherwise reimplement FR-ENT-04/05 natively in the gateway, with the same tests. |
| A4 | No existing shared DB engine is mandated on the VM; PostgreSQL is acceptable. | Swap the EF Core provider; the schema design is unchanged. |
| A5 | There is no official RevenueCat .NET MAUI SDK. The app will use a .NET Android binding of `purchases-android` (pinned version, same approach as `RootEncoder.Binding`) or a maintained community binding, behind `IPurchasesService`. **A spike is required first** (timebox 2 days). | This is the highest delivery risk. If it fails, fallback is Google Play Billing directly plus RevenueCat REST receipt posting (more work; D1 is unchanged). |
| A6 | Firebase ID tokens have a 60 min lifetime and the app uses the Firebase SDK's token refresh. | D5 bound changes accordingly. |
| A7 | Gating `sessions.remote.host` at the Gateway does not break existing WearOS usage in a way the owner cares about for v1 (for example, a free host can no longer create sessions the watch joins). | If the WearOS flow must keep working for free users, temporarily set `sessions.remote.host` to `free` in config (no code change) until O6 is resolved. |
| A8 | Separate Firebase projects / RevenueCat environments exist (or will) for dev and prod. Until the owner supplies credentials, everything runs in fake mode. | Sandbox flag (FR-WH-03) handles a shared RevenueCat project. A shared Firebase project needs care with test claims. |
| A9 | Users are currently able to use local scoring without signing in. | If sign-in is currently mandatory, AC-APP-01's "signed out" clause is downgraded to "signed in with an expired token". |

### 11.3 Open items for the owner (non-blocking for implementation with fakes)

- O1 Actual prices, trial length (if any) and countries.
- O2 Confirm D7 (joiners free) and D5 (<= 60 min refund lag).
- O3 Terms of Service and Privacy Policy URLs and content.
- O4 Final values for success-metric targets M5/M6 after baseline.
- O5 Schedule the account-deletion/GDPR follow-up before public paid launch.
- O6 Create a follow-up issue defining WearOS behaviour under tiers (messaging on 403 tier-required, whether the watch counts as a "controller device" for gating).
