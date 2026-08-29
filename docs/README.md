# LastSaaS Documentation

This folder explains **how LastSaaS works** and **how to build your own SaaS on top of it**.

Start here if you are new to the repository. The root [README](../README.md) is the marketing overview, feature list, and quick start. These documents go deeper: request flow, data model, each platform feature, and a concrete product-building recipe.

| Document | What it covers |
|----------|----------------|
| [Architecture](architecture.md) | Stack, process model, folders, request lifecycle, tenancy, auth, data stores |
| [Features](features.md) | How every platform capability works (auth, billing, branding, webhooks, admin, MCP, …) |
| [Build your SaaS](build-your-saas.md) | Fork/customize workflow: models, APIs, UI, entitlements, credits, deploy |

## Mental model in one paragraph

LastSaaS is a **single Go HTTP server** plus a **React SPA**. The Go process owns MongoDB, JWT/OAuth, Stripe, email, admin APIs, and (in production) serves the built frontend. A **user** can belong to one or more **tenants** (workspaces). One special **root tenant** is your operator org — its members see `/last` (the admin console). Customer tenants get dashboard, team, billing, and your product pages. You do **not** rewrite auth or billing; you add **product collections, handlers, and pages** that reuse tenant context, plans, credits, and webhooks.

## Typical reading order

1. Skim [Architecture](architecture.md) until the request-lifecycle diagram makes sense.
2. Use [Features](features.md) as a map when you need to change a specific area.
3. Follow [Build your SaaS](build-your-saas.md) when you start adding your product.
