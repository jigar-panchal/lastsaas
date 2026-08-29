# Architecture

LastSaaS is a production SaaS **platform** (auth, tenancy, billing, admin, branding) that you customize into a **product**. This document describes how the running system is put together.

## Stack

| Layer | Choice | Role |
|-------|--------|------|
| HTTP API | Go 1.25, gorilla/mux | All REST routes under `/api` |
| UI | React 19, TypeScript, Vite 7, Tailwind 4 | SPA; routes in `frontend/src/App.tsx` |
| Database | MongoDB | Documents, indexes, JSON Schema validation, TTL |
| Auth | JWT access + refresh, bcrypt, OAuth, TOTP, magic links | Identity |
| Billing | Stripe Checkout, Portal, webhooks, Tax | Money |
| Email | Resend | Verification, invites, resets, magic links |
| Deploy | Docker multi-stage Alpine image, Fly.io | One container, port 8080 |

Local development runs **two processes**:

- Backend: `go run ./cmd/server` on port **4290** (configurable).
- Frontend: `npm run dev` on port **4280**. Vite **proxies** `/api` to the Go server.

Production runs **one process**. The Dockerfile builds the Go binary and the Vite `dist/` folder. The binary serves the SPA for non-`/api` paths (`spaHandler` in `backend/cmd/server/main.go`).

## Repository layout

```
lastsaas/
  backend/
    cmd/server/main.go     HTTP server: wiring, routes, SPA, shutdown
    cmd/lastsaas/          CLI + MCP stdio server
    config/                YAML templates (dev/prod); secrets via ${ENV}
    internal/
      api/handlers/        HTTP handlers
      middleware/          Auth, tenant, RBAC, rate limit, billing, metrics
      models/              MongoDB document structs + validate tags
      db/                  Connection, collections, indexes, JSON Schema
      auth/                JWT, password, OAuth, TOTP
      stripe/              Stripe SDK wrapper
      email/               Resend templates
      events/              In-process event types → webhook dispatcher
      webhooks/            Outbound HMAC deliveries
      telemetry/           Product analytics SDK + queries
      configstore/         Runtime config (DB, cached 60s)
      syslog/              Operator event log
      health/              Node heartbeat + metrics
      validation/          go-playground/validator
      version/             VERSION file + startup migrations
  frontend/
    src/App.tsx            Routes + BootstrapGuard
    src/api/client.ts      Axios client, token refresh, X-Tenant-ID
    src/contexts/          Auth, Tenant, Branding, Theme
    src/pages/             public / auth / app / admin
  Dockerfile               Production image (Go + SPA)
  fly.toml                 Fly.io: port 8080, /health check
  VERSION                  Semver string baked into the binary
```

There is no separate “product” service in this repo. Your product lives in the same Go process and SPA unless you later split a second backend (see [Build your SaaS](build-your-saas.md#two-backends)).

## Startup sequence

`cmd/server/main.go` does the following in order:

1. Load `.env` (if present) and YAML config for `LASTSAAS_ENV` (`dev` or `prod`).
2. Connect to MongoDB; create indexes; apply JSON Schema (skipped in `test`).
3. Read `VERSION`, run `version.CheckAndMigrate`.
4. Seed `configstore` defaults and start a 60s reload loop.
5. Seed the system **Free** plan if missing.
6. Start syslog (optional DataDog forwarding).
7. Construct JWT, OAuth, email, Stripe, webhook dispatcher, telemetry, health.
8. Register gorilla/mux routes and middleware.
9. Listen; on SIGINT/SIGTERM, drain connections (30s).

Until a root tenant exists, most `/api` routes are blocked by `BootstrapGuard`. Status is public: `GET /api/bootstrap/status`. Initialization is **CLI-only**: `go run ./cmd/lastsaas setup` (creates root tenant + owner). The UI `/setup` page exists for first-run UX; it does not replace the CLI.

## Request lifecycle

```
Browser / API client
        │
        ▼
  CORS + security headers + body size limit + recovery + HTTP metrics
        │
        ▼
  /health          → Mongo ping (no /api prefix)
  /api/...         → Request ID + X-API-Version
        │
        ├── public: bootstrap status, docs, branding assets
        ├── Stripe inbound: POST /api/billing/webhook (signature, not JWT)
        └── BootstrapGuard → remaining APIs
                │
                ├── public auth (login, OAuth, magic link) + rate limits
                ├── RequireAuth (JWT or lsk_ API key)
                │       └── optional RequireTenant (X-Tenant-ID)
                │               └── RequireRole / RequireRootTenant
                │               └── RequireActiveBilling / RequireEntitlement
                └── SPA fallback (production only)
```

### Headers that matter

| Header | Purpose |
|--------|---------|
| `Authorization: Bearer <jwt>` | User session |
| `Authorization: Bearer lsk_...` | API key (hashed in DB) |
| `X-Tenant-ID` | Which workspace the request is for (required on tenant-scoped routes) |
| `X-Request-ID` | Set on every API response |
| `X-API-Version` | From `VERSION` |

The frontend Axios client (`frontend/src/api/client.ts`) attaches both auth and tenant headers. On **401**, it tries `/auth/refresh` once, then redirects to `/login`.

### Context values (Go)

After middleware, handlers read:

- `middleware.GetUserFromContext` — `models.User`
- `middleware.GetTenantFromContext` — `models.Tenant`
- `middleware.GetMembershipFromContext` — role on that tenant

Always scope product queries by `tenant.ID`. Isolation tests live in `backend/internal/api/handlers/isolation_test.go`.

## Multi-tenancy

```
User ──< TenantMembership (role: owner | admin | user) >── Tenant
                                                              │
                                                              ├── Plan (entitlements, credits)
                                                              ├── Stripe customer / subscription
                                                              ├── subscriptionCredits + purchasedCredits
                                                              └── your product documents (you add these)
```

- **Root tenant** (`tenant.IsRoot == true`): operator org. Admin UI is `/last`. Admin APIs require `RequireRootTenant()` plus a minimum role.
- **Customer tenants**: created at signup (and via invitations). Billing, seats, and credits attach here — not to the user.
- A user may belong to **multiple** tenants. The SPA stores the active tenant in `localStorage` (`lastsaas_active_tenant`) and sends it as `X-Tenant-ID`.

Role hierarchy (higher includes lower): `user` < `admin` < `owner`.

Admin console uses the **same** roles on the **root** tenant:

| Root role | Admin API |
|-----------|-----------|
| user | Read-only (`GET /api/admin/...`) |
| admin | Writes: users, tenants, plans, keys, webhooks, … |
| owner | Destructive: delete user, impersonate, branding, subscription overrides |

## Authentication model

Access tokens are short-lived JWTs; refresh tokens are stored (and rotated) in MongoDB. Tokens live in **localStorage** by design (see comment in `AuthContext.tsx`) so the SPA can send `Authorization` headers easily; XSS is mitigated with CSP, React escaping, and sanitization of injected branding HTML (DOMPurify).

API keys (`lsk_` prefix) use SHA-256 hashes. **Admin** keys auto-bind the root tenant. **User** keys still need `X-Tenant-ID`. Last-used timestamps are updated on use.

MFA: after password login, the API may return `mfaRequired` + a short-lived MFA token; `/auth/mfa/challenge` completes login.

## Data layer

Collections are accessed via methods on `db.MongoDB` (`Users()`, `Tenants()`, `Plans()`, …). Two validation layers must stay in sync when you change a model:

1. Go `validate` struct tags (`internal/validation`)
2. MongoDB JSON Schema (`internal/db/schema.go`)

TTL indexes expire refresh tokens, invitations, audit logs, health metrics, telemetry, etc. Do not store product data you need forever in a TTL collection.

Runtime **config vars** (email subjects, feature flags, upgrade-prompt copy) live in `config_vars`, cached in memory, editable in Admin → Config. YAML/env is for **secrets and boot** (Mongo URI, JWT secrets, Stripe keys).

## Frontend architecture

Provider stack (outer → inner): `QueryClient` → `BootstrapGuard` → `BrandingProvider` → `AuthProvider` → `ThemeProvider` → `TenantProvider` → router.

Route groups:

| Path | Audience |
|------|----------|
| `/`, `/p/:slug` | Public landing / custom pages |
| `/login`, `/signup`, OAuth, MFA, magic link | Auth |
| `/onboarding` | First-run after signup |
| `/dashboard`, `/team`, `/plan`, `/settings`, … | Authenticated app (`Layout`) |
| `/last/*` | Root-tenant admin (`AdminLayout`) |

Branding (colors, logo, nav items, landing HTML) is loaded from public `/api/branding` and injected as CSS variables (`BrandingThemeInjector`). Product pages should use those tokens, not hard-coded brand colors, if you want white-label to work.

## Configuration and environments

| Mechanism | Examples |
|-----------|----------|
| `.env` + `backend/config/{dev,prod}.yaml` | `MONGODB_URI`, `JWT_*`, OAuth, Stripe, Resend |
| `LASTSAAS_ENV` | Selects YAML file; Docker sets `prod` |
| Config store (DB) | `app.name`, email templates, billing copy |

`DATABASE_NAME` is the **project identity**. Two apps sharing the same Mongo database name share users and tenants.

## Operational extras

- **Health**: each node heartbeats; metrics every 60s; UI at `/last/health`. Fly checks `GET /health`.
- **Syslog**: `syslog.Logger` for operator-visible events (severity: critical → debug).
- **CLI** (`cmd/lastsaas`): setup, config, users, tenants, doctor, MCP, …
- **MCP**: stdio process that calls the **read-only** admin HTTP API with a root API key.

## What you should not reinvent

Do not add a second login system, a parallel Stripe integration, or a new tenant table. Extend:

- Models + schema + tests
- Handlers + routes in `main.go`
- Pages + `client.ts` methods + `App.tsx` routes
- Entitlements on plans, credits via `/usage/record` or in-process deduction
- Events via `events.Emitter` so outbound webhooks fire automatically
