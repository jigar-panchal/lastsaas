# Build a SaaS application on LastSaaS

LastSaaS is the **platform**. Your SaaS is **domain logic + UI** on the same codebase (recommended) or a second backend that still uses LastSaaS for auth and billing.

This guide assumes the recommended path: **fork or clone this repo and add product features in-process**.

---

## What you get for free

After `./scripts/setup.sh`, backend + frontend running, and `go run ./cmd/lastsaas setup`:

- Sign up, login, OAuth, MFA, teams, invites
- Root admin at `http://localhost:4280/last`
- Plans, Stripe checkout, credits, invoices (once Stripe keys + webhook are set)
- Branding, API keys, outbound webhooks, health, PM analytics
- Tenant isolation middleware and JWT/API-key auth on every new route you register correctly

You should spend time on **your** objects (e.g. projects, documents, jobs), **your** APIs, and **your** screens — not on a new auth stack.

---

## Recommended workflow

### 1. Make it yours operationally

1. Clone/fork; run `./scripts/setup.sh` (writes `.env`).
2. Change `APP_NAME`, `FROM_EMAIL`, `DATABASE_NAME` (unique per product).
3. Log in as root owner → **Admin → Branding** (name, colors, landing page).
4. **Admin → Plans** — replace the Free plan’s entitlements with keys your product will check (`api_access`, `exports`, `max_projects`, …).
5. Configure Resend, OAuth, and Stripe when you need those channels (see root README).

Keep `CLAUDE.md` (or `MEMORY.md`) so AI agents remember validation, syslog, and deploy rules.

### 2. Add a product data model

Example: a tenant-owned **Project**.

1. Add `backend/internal/models/project.go` with `TenantID`, `CreatedBy`, timestamps, and `validate` tags.
2. Add Mongo JSON Schema in `internal/db/schema.go` and register it in `AllSchemas()`.
3. Add `Projects()` on `db.MongoDB`, indexes (always include `tenantId`), TTL only if the data should expire.
4. Add tests in `internal/validation/validate_test.go`.
5. Run `cd backend && go test ./internal/validation/...`

Every document that customers write **must** include `tenantId` and queries **must** filter on it.

### 3. Add HTTP handlers

Create `backend/internal/api/handlers/projects.go`:

- Read `middleware.GetTenantFromContext` / `GetUserFromContext` / `GetMembershipFromContext`.
- Never take `tenantId` from the JSON body as the source of truth; use context.
- Validate with `validation.Validate`.
- Log significant actions with `syslog.Logger`.
- Emit domain events with `emitter.Emit` if customers should receive webhooks (add a new `events.EventType` and document it).
- Track product usage with `telemetry.Track` and/or `POST`-equivalent in-process calls.

Wire routes in `backend/cmd/server/main.go` on a subrouter that already has:

```go
productAPI := guarded.PathPrefix("/projects").Subrouter()
productAPI.Use(authMiddleware.RequireAuth)
productAPI.Use(tenantMiddleware.RequireTenant)
productAPI.Use(middleware.RequireActiveBilling()) // if paid-only
productAPI.Use(middleware.RequireEntitlement(database, "projects")) // if plan-gated
productAPI.HandleFunc("", projectsHandler.List).Methods("GET")
productAPI.HandleFunc("", projectsHandler.Create).Methods("POST")
// owner/admin-only mutations:
write := productAPI.PathPrefix("").Subrouter()
write.Use(middleware.RequireRole(models.RoleAdmin))
write.HandleFunc("/{id}", projectsHandler.Delete).Methods("DELETE")
```

Numeric entitlements (e.g. max projects) are **not** fully enforced by `RequireEntitlement` (that helper is boolean/presence). Enforce limits in the handler by loading the plan and comparing `plan.Entitlements["max_projects"].NumericValue` to a count query.

### 4. Meter credits (usage-based SaaS)

Two options:

**A. HTTP (good for other languages / scripts):**  
`POST /api/usage/record` with `{ "type": "ai_tokens", "quantity": 10, "metadata": { ... } }`.

**B. In-process (same transaction as your job):**  
Follow the pattern in `handlers/usage.go` (subscription bucket first, then purchased, in a Mongo session). Do not decrement credits without a matching `usage_events` row.

Show remaining credits in the UI (`plansApi.list()` already returns tenant credit fields used by `Layout`).

### 5. Frontend: API + page + nav

1. Add methods on `frontend/src/api/client.ts` (types in `src/types/index.ts`).
2. Add `frontend/src/pages/app/ProjectsPage.tsx`.
3. Register the route inside the authenticated `Layout` in `App.tsx`.
4. Either add a link in `Layout.tsx` **or** add a nav item in **Admin → Branding** (supports entitlement-gated items).

Reuse `components/ui/*` (Button, Card, Modal, …). Respect `useBranding()` / CSS variables.

Call `telemetry` page views if you want funnel coverage beyond auto-instrumentation.

### 6. Gate the UI the same way as the API

Backend enforcement is mandatory. Frontend should hide or disable features using plan entitlements from `plansApi.list()` (see `TestEntitlementsPage.tsx` and `Layout.tsx` credit/team visibility). A determined user can still hit the API — the middleware is the real lock.

### 7. Webhooks for your customers

If your product should notify Zapier/customer backends:

1. Add `EventType` in `internal/events/emitter.go`.
2. `Emit` from the handler with a stable JSON `Data` map (ids, tenantId, timestamps).
3. Document the payload in `handlers/docs.go` / OpenAPI.
4. Operators subscribe in Admin → API → Webhooks.

### 8. Config without redeploy

Add a row to `configstore.SystemDefaults` (or create via Admin → Config) for copy, feature flags, limits. Read with `cfgStore.Get("your.key")` in handlers. Do **not** put secrets there; secrets stay in env/YAML.

### 9. Verify

```bash
cd backend && go build ./... && go test ./...
cd frontend && npx tsc --noEmit
```

Exercise login, tenant switch, your CRUD, a failed entitlement, and insufficient credits.

---

## Product patterns that already exist

| Need | Use |
|------|-----|
| Who is calling? | `GetUserFromContext` |
| Which customer? | `GetTenantFromContext` + `X-Tenant-ID` |
| Permission | `RequireRole(RoleAdmin)` / owner |
| Paid subscriber | `RequireActiveBilling()` |
| Plan feature flag | `RequireEntitlement(db, "key")` + plan entitlements in Admin |
| Usage pricing | Dual credit buckets + `/usage/record` |
| Admin-only ops | Routes under `/api/admin` + `RequireRootTenant` |
| Customer integrations | API keys (`lsk_`) — same `RequireAuth` |
| Outbound notify | `events.Emitter` |
| Analytics | `telemetry.Service` |
| Operator audit | `syslog.Logger` |

---

## Local vs production

**Local:** two terminals (Go `:4290`, Vite `:4280`). Stripe CLI:  
`stripe listen --forward-to localhost:4290/api/billing/webhook`.

**Production (this repo as the whole app):**

```bash
fly deploy -c fly.toml
```

Dockerfile: Go binary + SPA, `LASTSAAS_ENV=prod`, port **8080**. Set Fly secrets (`MONGODB_URI`, JWT, `FRONTEND_URL` = public HTTPS origin, Stripe, Resend, OAuth redirect URLs).

Health check path: `/health`.

---

## Two backends (advanced)

Some products keep LastSaaS as the **identity/billing** process and run a **second** product API.

That only works if:

- The browser still talks to LastSaaS for `/api/auth/*`, `/api/bootstrap/status`, billing, admin.
- The product API validates the **same** JWTs (or calls LastSaaS) and still requires `X-Tenant-ID`.
- Deploy uses a **multi-process** image (platform API + LastSaaS + reverse proxy), not a product-only Dockerfile that swallows `/api` with the SPA.

If you split processes, document it in a root `deploy.md` and never use a bare product `fly deploy` that omits LastSaaS — login will look “broken” (HTML instead of JSON on `/api/auth/*`, redirects to `/setup`). This repository’s default Dockerfile **is** LastSaaS and is correct for the in-process product path.

---

## Checklist before first customers

- [ ] Unique `DATABASE_NAME` and strong JWT secrets
- [ ] Root owner MFA enabled
- [ ] Branding and legal/landing copy
- [ ] Plans + entitlements match gated routes
- [ ] Stripe live webhook with the eight event types in the root README
- [ ] Resend domain authentication
- [ ] OAuth redirect URLs match production `FRONTEND_URL` / API host
- [ ] Product collections indexed by `tenantId`
- [ ] Isolation tests for new APIs
- [ ] Syslog on billing- and data-destructive paths

---

## Where to look in code (cheat sheet)

| Task | Place |
|------|--------|
| Register a route | `backend/cmd/server/main.go` |
| Auth / tenant middleware | `backend/internal/middleware/` |
| New document type | `models/` + `db/schema.go` + `db/mongodb.go` |
| Stripe behavior | `internal/stripe`, `handlers/billing.go`, `handlers/webhook.go` |
| SPA routes | `frontend/src/App.tsx` |
| API client | `frontend/src/api/client.ts` |
| Nav / chrome | `frontend/src/components/Layout.tsx` |
| Agent rules | `CLAUDE.md` |
