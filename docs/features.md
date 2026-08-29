# Features — how they work

This is a map of platform capabilities: what they do, where they live, and how they interact. For adding *your* product, see [Build your SaaS](build-your-saas.md).

---

## 1. Bootstrap and first owner

**Purpose:** Prevent an empty database from accepting signups until an operator exists.

**How:** `GET /api/bootstrap/status` reads `system_config.initialized`. The frontend `BootstrapGuard` sends everyone to `/setup` until that flag is true. Creating the root tenant is `lastsaas setup` (CLI), which writes the root tenant, owner user, membership, and sets initialized.

**Files:** `handlers/bootstrap.go`, `cmd/lastsaas` `setup`, `frontend/src/pages/BootstrapPage.tsx`

---

## 2. Authentication and identity

**Email/password:** Register hashes passwords with bcrypt. Login is timing-safe (dummy compare on unknown emails). Failed attempts can lock the account (config/rate limits).

**Email verification / password reset:** Tokens stored hashed where applicable; emails via Resend. If Resend is unset, those flows degrade (warn on startup).

**JWT:** Access token in `Authorization`; refresh token rotated on `/auth/refresh`. Password change revokes sessions. Logout revokes the presented refresh token.

**OAuth (Google, GitHub, Microsoft):** Only registered if env credentials exist. `GET /api/auth/{provider}` starts the flow; callback exchanges code, links or creates a user, then the SPA uses `/auth/exchange-code` (or redirect) to obtain JWTs. `GET /api/auth/providers` tells the UI which buttons to show.

**Magic link:** `POST /api/auth/magic-link` emails a link; `/auth/magic-link/verify` issues JWTs.

**MFA (TOTP):** Setup/verify/disable under `/api/auth/mfa/*`. Recovery codes are high-entropy. TOTP secrets can be encrypted at rest when `WEBHOOK_ENCRYPTION_KEY` is set (shared AES key used for webhook secrets too).

**Sessions:** `/api/auth/sessions` lists refresh-token sessions; delete one or all.

**Passkeys / SSO:** Models exist (`webauthn_credential`, `sso_connection`). Health checks register when config flags `auth.passkeys.enabled` / `auth.sso.enabled` are true. Treat these as optional/platform flags, not the primary login path unless you enable them in config.

**Files:** `internal/auth/*`, `handlers/auth.go`, `pages/auth/*`, `contexts/AuthContext.tsx`

---

## 3. Multi-tenancy, teams, RBAC

**Memberships:** `tenant_memberships` unique on `(userId, tenantId)`.

**Invites:** Admin+ posts `/api/tenant/members/invite`. Email + token; accept via `/api/auth/accept-invitation`. Seat limits come from the plan (`userLimit`, per-seat min/max).

**Roles:**

| Action | Minimum role |
|--------|----------------|
| List members, activity | any member |
| Invite / remove | admin |
| Change roles, transfer ownership, tenant settings | owner |
| Checkout / cancel subscription | owner |

**Activity log:** Per-tenant audit-style entries (`GET /api/tenant/activity`).

**Onboarding:** `/auth/complete-onboarding` after first login; UI route `/onboarding`.

**Files:** `handlers/tenant.go`, `models/membership.go`, `pages/app/TeamPage.tsx`

---

## 4. Billing, plans, credits, invoices

**Plans:** Stored in MongoDB, managed in Admin → Plans. Fields include monthly price, annual discount %, flat vs per-seat, included/min/max seats, monthly usage credits, reset vs accrue, bonus credits, trial days, `entitlements` map.

A system **Free** plan is seeded (`planstore.Seed`). Stripe Products/Prices are created at **checkout time**, not in the Stripe Dashboard.

**Checkout:** Tenant **owner** `POST /api/billing/checkout` → Stripe Checkout Session. Success/cancel URLs are `/billing/success` and `/billing/cancel`.

**Portal:** `POST /api/billing/portal` → Stripe Billing Portal (cards, invoices).

**Inbound webhooks:** `POST /api/billing/webhook` (no JWT). Signature via `STRIPE_WEBHOOK_SECRET`. Handlers update `billingStatus`, subscription IDs, credits, financial transactions. Events include checkout completed, invoice paid/failed, subscription updated/deleted, refunds, disputes. Processing is idempotent (`webhook_events` unique IDs).

**Credits (two buckets):**

- `tenant.subscriptionCredits` — from the plan; reset or accrue on billing cycle.
- `tenant.purchasedCredits` — from credit bundles.

`POST /api/usage/record` (auth + tenant + `RequireActiveBilling`) deducts **subscription first**, then purchased, in a MongoDB transaction, and writes `usage_events`. Insufficient credits → error. `GET /api/usage/summary` for dashboards.

**Credit bundles:** Admin CRUD; customers `GET /api/credit-bundles` and checkout like subscriptions.

**Promotions:** Admin creates Stripe coupons/promotion codes; Checkout can accept them.

**Invoices:** Sequential numbers; HTML/JSON plus PDF (`gofpdf`) on `/api/billing/transactions/{id}/invoice`.

**Enforcement:**

- `RequireActiveBilling()` — blocks if status is past_due / canceled / unpaid (root and `billingWaived` skip). Status `none` (never subscribed) is allowed so free-tier APIs still work.
- `RequireEntitlement(db, "feature_key")` — boolean (and presence) check on the tenant’s plan. Root / waived skip.

**Trial abuse:** Tracked via `trialUsedAt` and related checks so a user/tenant cannot infinitely re-trial.

**Files:** `handlers/billing.go`, `handlers/webhook.go`, `handlers/plans.go`, `handlers/bundles.go`, `handlers/usage.go`, `internal/stripe`, `middleware/tenant.go`

---

## 5. White-label branding

**Public reads (no auth):** `/api/branding`, assets, media, `/api/branding/page/{slug}`.

**Writes:** Root **owner** only — theme colors, fonts, logos, landing HTML, custom pages (`/p/{slug}`), CSS/head injection, nav items (can be entitlement-gated), auth copy, dashboard HTML, favicon, OG image.

Frontend `BrandingProvider` + `BrandingThemeInjector` apply CSS variables. Custom HTML is sanitized with DOMPurify before inject.

**Files:** `handlers/branding.go`, `pages/admin/BrandingPage.tsx`, `pages/public/*`

---

## 6. API keys

Created in Admin → API (`POST /api/admin/api-keys`). Raw key shown **once**. Auth middleware treats `Bearer lsk_...` as a key lookup. Events: `api_key.created` / `api_key.revoked`.

---

## 7. Outgoing webhooks

Operators configure URLs + event filters in Admin. Secrets are `whsec_` prefixed, HMAC-SHA256 signed, optionally encrypted at rest.

In-process `events.Emitter` (`internal/events`) is implemented by `webhooks.Dispatcher`. Handlers call `Emit` on user/tenant/billing/security actions; the dispatcher POSTs JSON to subscribers and stores deliveries.

Event type constants live in `internal/events/emitter.go` (users, tenants, members, billing, API keys, plus `system.initialized`).

**Files:** `handlers/webhooks.go` (CRUD/test), `internal/webhooks/dispatcher.go`

---

## 8. Admin console (`/last`)

Root-tenant members only (`AdminLayout`). Surfaces:

| Area | What it drives |
|------|----------------|
| Dashboard | Counts, health snapshot |
| Users / tenants | Search, status, plan assign, CSV export |
| User profile | Memberships, impersonate, delete preflight |
| Root members | Invite operators |
| Plans / bundles / promotions | Catalog |
| Financial | All-tenant transactions, revenue/ARR/DAU/MAU |
| PM | Funnel, KPIs, retention, engagement, custom events |
| Health | Nodes, charts, integrations (Mongo, Stripe, Resend, OAuth, DataDog, …) |
| Logs | Syslog search |
| Config | Runtime vars |
| API | Keys + outbound webhooks |
| Branding / announcements | White-label + in-app banners |
| Messages | Operator → user inbox |
| About | Version |

Impersonation issues a short-lived JWT with `impersonatedBy`; UI shows `ImpersonationBanner`.

---

## 9. Customer self-service

Dashboard, team, plan picker, buy credits, settings (profile, security, MFA, sessions, billing tab), activity, data export, account deletion, messages inbox, announcements.

`/test-entitlements` is a **root-tenant** helper to try plan flags — not a customer feature.

---

## 10. Product analytics and telemetry

**Auto-tracked:** registration, verification, login, checkout, subscription activate/cancel, plan change (wired in auth/billing handlers).

**Go SDK:** `telemetry.Track`, `TrackBatch`, `TrackPageView`, `TrackCheckoutStarted`, `TrackLogin` — buffered writes, no HTTP hop.

**HTTP:**

- `POST /api/telemetry/track` — anonymous page views, IP rate limit
- `POST /api/telemetry/events` and `/batch` — JWT + tenant

TTL ~365 days. Admin PM pages query aggregations (funnel, KPIs, cohorts). Optional DataDog observer on each event.

**Event definitions:** Admin can define named events / Sankey views for the PM UI.

**Files:** `internal/telemetry/service.go`, `handlers/telemetry.go`, `handlers/pm.go`, `pages/admin/PMPage.tsx`

---

## 11. System health, syslog, DataDog

**Health service:** Registers the node, heartbeats ~30s, samples CPU/mem/disk/HTTP/Mongo/Go runtime ~60s. HTTP middleware records latency percentiles. Admin charts filter by node and range (1h–30d). Integrations run connectivity checks.

**Syslog:** Structured operator log with injection-pattern detection. Admin → Logs.

**DataDog (optional):** If `DATADOG_API_KEY` is set, syslog, health snapshots, and telemetry can forward. Health dashboard shows a DataDog check.

---

## 12. Messaging and announcements

**Messages:** Per-user inbox (`/api/messages`). Admin send; CLI `send-message`. Unread badge in `Layout`.

**Announcements:** Tenant-visible banners; dismiss stored in localStorage.

---

## 13. Built-in API docs

Public:

- `GET /api/docs` — HTML
- `GET /api/docs/markdown`
- `GET /api/docs/openapi.json`

Generated from handler documentation code — keep it updated when you add endpoints.

---

## 14. CLI and MCP

**CLI** (`go run ./cmd/lastsaas <cmd>`): setup, start/stop/restart (process helpers), change-password, send-message, transfer-root-owner, config get/set, version, status, logs, users, tenants, health, stats, doctor, financial, db, mcp.

**MCP:** `lastsaas mcp` with `LASTSAAS_URL` + `LASTSAAS_API_KEY` (admin authority). Stdio, **read-only** tools (dashboard, users, tenants, financials, logs, health, plans, webhooks, PM metrics, …). Safe for connecting Claude to production **reads**.

Manifests: `manifest.json`, `server.json`, `smithery.yaml`, `glama.json`.

---

## 15. Security controls (platform)

- Security headers (CSP, HSTS, frame options, …)
- Distributed rate limits (Mongo-backed) on auth, invites, usage, telemetry, CSV export
- 1MB body cap on API
- Regex escaping on search (NoSQL injection)
- CSV formula sanitization
- Stripe + outbound webhook signatures
- Trusted client IP: `Fly-Client-IP` in production (not raw `X-Forwarded-For`)

---

## 16. Versioning

`VERSION` file is compiled into the binary (`-ldflags` in Docker). Startup migrations live in `internal/version`. Admins may receive in-app messages after upgrades. Changelog: [VERSIONS.md](../VERSIONS.md).
