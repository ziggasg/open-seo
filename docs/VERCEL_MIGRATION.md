# Vercel-only migration plan

## Goal

Run the complete OpenSEO application on Vercel without Cloudflare Workers, Workerd, Wrangler, D1, KV, R2, Durable Objects, Cloudflare Workflows, Cloudflare Cron Triggers, or the Celld Docker runtime at production runtime.

This is a platform migration, not a static-site deployment. The target must preserve the existing product workflows and security boundaries.

## Repository baseline

The current application is a TanStack Start application built through Vite, but its production entry point is also a Cloudflare Worker:

- `vite.config.ts` installs `@cloudflare/vite-plugin` and configures the `ssr` Worker environment.
- `src/server.ts` exports a Worker-style `fetch(request, env, ctx)` handler, scheduled handler, Durable Object/Workflow exports, and Cloudflare-specific request routing.
- `wrangler.jsonc` declares D1, KV, R2, Durable Objects, service bindings, Workflows, and cron triggers.
- `wrangler.audit.jsonc` builds the auxiliary audit Worker.
- `src/standalone-server.ts` and `wrangler.standalone.json` bundle Worker classes for Celld. The Docker edition is not a Vercel runtime adapter.
- A PostgreSQL dialect already exists under `src/db/pg`, including a schema and migrations under `drizzle/pg`. The application currently defaults to D1 and selects PostgreSQL only when `DATABASE_PROVIDER=postgres`.

The existing Vercel deployments prove that Vercel can install and build the repository in its current configuration; they do not prove that the Worker bindings or background services work on Vercel.

## Target architecture

| Current Cloudflare capability | Vercel-only target |
| --- | --- |
| Cloudflare Vite plugin / Worker entry | TanStack Start + `nitro/vite`, deployed as Node.js Vercel Functions |
| D1 | PostgreSQL through a Vercel Marketplace integration (Neon is the preferred first candidate); use the existing PostgreSQL schema and migrations |
| KV / OAuth KV | Redis through a Vercel Marketplace integration (Upstash is the preferred first candidate), with explicit TTL and atomic-operation semantics |
| R2 | Vercel Blob, with private/public access rules preserved |
| Cloudflare Workflows | Vercel Workflows for durable, retryable site audits, rank checks, AI visibility and local-rank jobs |
| Cron Triggers | Authenticated Vercel Cron routes; preserve the five-minute and daily schedules subject to the selected Vercel plan's cron limits |
| Durable Objects (SAM chat) | Postgres-backed conversation/session state plus a Vercel-compatible streaming transport; no process-local state |
| Audit scratchpad Durable Object | Postgres-backed audit state or Redis with explicit expiry, selected after workload and consistency review |
| Cross-worker service bindings | Direct server-side module/service boundaries inside the Node.js deployment; no implicit in-memory coupling across function instances |
| Celld crawl gateway | A dedicated Vercel Node.js function/service boundary with strict SSRF and private-network protections |
| Wrangler secrets/bindings | Vercel environment variables and Marketplace-injected credentials |

The target may use Vercel Marketplace integrations for managed Postgres and Redis. These are provisioned and connected through Vercel, but are separate database providers; Vercel does not provide a native D1 or Redis-compatible KV service.

## Implementation phases

### Phase 0 — establish a safe migration branch

- Keep upstream `every-app/open-seo` and the `main` branch unchanged.
- Work on `vercel-migration` in `ziggasg/open-seo`.
- Keep the existing Cloudflare and Docker paths available until Vercel parity is demonstrated.

### Phase 1 — runtime and build adapter

- Add the Nitro Vite plugin and a Vercel-specific build path.
- Ensure the Vercel build does not initialize the Cloudflare Vite plugin, Workerd, Wrangler, or auxiliary Worker build.
- Extract request handling from the Worker-only `fetch(request, env, ctx)` wrapper into a platform-neutral application handler.
- Add a Node.js/Nitro entry that supplies configuration through `process.env`, handles TanStack Start SSR/API/MCP routes, and does not import `cloudflare:workers` at module load.
- Keep the Worker entrypoint as a separate build target for upstream compatibility.

### Phase 2 — database and persistence

- Make PostgreSQL the Vercel deployment's only database provider.
- Reuse `src/db/pg` and `drizzle/pg`; do not route production Vercel traffic to D1.
- Add a deployment-safe migration command and readiness check.
- Audit every D1-specific query, batch, transaction, SQLite function, and raw SQL statement for PostgreSQL parity.
- Implement and test a one-time D1-to-Postgres data migration for existing users, organizations, projects, audit results, reports, rank history, OAuth grants, and settings. Do not point production at an empty database or silently discard existing data.

### Phase 3 — storage, cache and OAuth

- Introduce storage interfaces so application code does not import R2/KV bindings directly.
- Implement Vercel Blob storage, including private object access, downloads, deletion, content types, and lifecycle/retention behavior.
- Implement Redis-backed cache and OAuth key-value storage with expiry, atomic writes, and concurrency tests.
- Preserve OAuth token encryption, rotation, revocation, and cleanup semantics.

### Phase 4 — durable background execution

- Port each Workflow separately: site audit, rank check, AI visibility, and local rank.
- Preserve idempotency, checkpoints, retries, cancellation, progress reporting, billing guards, and stale-job reconciliation.
- Use durable workflow steps rather than detached promises or process-local queues.
- Replace the SAM Durable Object with durable Postgres state and a reconnect-safe streaming interface.
- Move the audit scratchpad out of Durable Objects and prove concurrent audit isolation.

### Phase 5 — cron, crawling and external integrations

- Add authenticated Vercel Cron endpoints for the five-minute maintenance/rank schedule and the daily 03:17 cleanup schedule.
- Make scheduled handlers safe to run more than once and safe under overlapping invocations.
- Port the crawl gateway to Node.js and retain URL/IP validation, redirect re-validation, DNS rebinding defenses, and private/link-local address blocking.
- Verify DataForSEO, OpenRouter, Google OAuth/Search Console/Analytics, billing webhooks, MCP OAuth, MCP streaming, report downloads, and any provider callbacks on Vercel.

### Phase 6 — verification and production cutover

- Run type-checking, lint, unit tests, PostgreSQL integration tests, and the existing relevant Playwright flows.
- Add Vercel deployment smoke tests for SSR, auth/session persistence, API routes, MCP, uploads/downloads, background-job completion, cron authentication, and health checks.
- Test with multiple concurrent Vercel Function instances to catch reliance on local memory or local filesystem state.
- Back up the source database and object storage before migration; verify row counts and representative records after import.
- Deploy to a protected Vercel Preview first. Promote only after the acceptance checklist passes.
- Keep Cloudflare deployment and data backups available for rollback until the Vercel cutover is accepted.

## Non-negotiable acceptance criteria

1. Production runtime has no Cloudflare account dependency and no Worker binding dependency.
2. Authentication, sessions, MCP OAuth, and API keys persist across independent function instances and redeployments.
3. Existing PostgreSQL migrations apply cleanly from an empty database and upgrade a populated database without data loss.
4. Site audits, rank checks, AI visibility, and local rank jobs finish durably even if the initiating HTTP request or function instance ends.
5. SAM chat reconnects and restores conversation history after a new function instance handles the next request.
6. R2-backed assets and exports work through Vercel Blob with equivalent access control.
7. Cron endpoints reject unauthenticated requests and are idempotent.
8. Crawling cannot reach localhost, private, link-local, metadata, or otherwise prohibited network destinations, including through redirects or DNS changes.
9. No production secret is exposed through `VITE_*` or client bundles.
10. A protected Preview passes the functional smoke suite before any production alias is changed.

## Important constraints

- Do not deploy the current `build:standalone` output directly to Vercel. It is packaged for Celld and still expects Worker-compatible bindings.
- Do not simply remove the Cloudflare plugin and assume the app is portable. The current server entry, workflows, storage, auth/OAuth, and background services use Worker-specific APIs.
- Do not switch the production database provider until the PostgreSQL parity suite and data migration are verified.
- Do not remove Wrangler/Cloudflare files until the Vercel target has independent tests and the existing deployment path is no longer needed.

## Current status

This document records the architecture and acceptance criteria only. It does not claim that the application has been ported, that Vercel services have been provisioned, or that a Vercel-only deployment is production-ready. Implementation must proceed phase by phase with build and integration checks after each change.
