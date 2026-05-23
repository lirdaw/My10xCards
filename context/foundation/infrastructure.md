---
project: 10xcards
researched_at: 2026-05-23
recommended_platform: Cloudflare Workers (Static Assets)
runner_up: Vercel
context_type: mvp
tech_stack:
  language: TypeScript
  framework: Astro 6 (+ React islands)
  runtime: Cloudflare Workers (workerd, nodejs_compat)
  database: Supabase (external)
  ai_provider: OpenRouter (external)
---

## Recommendation

**Deploy on Cloudflare Workers with Static Assets.** The MVP is a stateless request/response Astro app with auth (Supabase) and AI generation (OpenRouter) — exactly the workload Cloudflare's edge runtime is optimised for. The user's existing familiarity with Cloudflare breaks the otherwise-tight scoring tie with Vercel and Netlify, the free tier (100K requests/day, 10 ms CPU per request) comfortably covers MVP traffic, and the `@astrojs/cloudflare` adapter is GA with `workerd`-backed local dev that mirrors production. One contract correction is mandatory: `tech-stack.md` records the runtime as `cloudflare-pages`, but the adapter dropped Pages support in v13 — the canonical target is **Workers + Static Assets**, and the bootstrapped scaffold must reflect that before the first deploy.

## Platform Comparison

Each platform is scored Pass / Partial / Fail against the five criteria in `references/agent-friendly-criteria.md`. The criteria are heuristics for "can an agent operate this platform from a terminal without a human clicking" — they are not generic "good platform" axes.

| Platform | CLI-first | Managed/Serverless | Agent-readable docs | Stable deploy API | MCP / Integration |
|---|---|---|---|---|---|
| Cloudflare (Workers + Static Assets) | Pass | Pass | Pass | Pass | Pass |
| Vercel | Pass | Pass | Pass | Pass | Pass |
| Netlify | Pass | Pass | Pass | Pass | Pass |
| Fly.io | Pass | Pass | Pass | Pass | Pass |
| Railway | Pass | Pass | Pass | Pass | Pass |
| Render | Partial | Pass | Pass | Pass | Pass |

### Shortlisted Platforms

#### 1. Cloudflare Workers + Static Assets (Recommended)

Astro's official adapter (`@astrojs/cloudflare` v13.5.4) targets Workers exclusively as of Astro 6; the old `Astro.locals.runtime.env` access pattern is replaced by `import { env } from "cloudflare:workers"`. `wrangler` is mature, scriptable, and covers deploy, rollback (`wrangler deployments list` + `wrangler rollback <id>`), and log tailing (`wrangler tail` with `--status error` and JSON output). The free tier — 100K requests/day, 10 ms CPU per invocation, free static assets — covers the MVP traffic envelope; awaiting an OpenRouter response is wall-clock, not CPU, so AI routes fit the free tier. Supabase works once `nodejs_compat` is enabled and `compatibility_date >= 2024-09-23`. Multiple Cloudflare MCP servers are GA (observability, docs, bindings, builds), and documentation is markdown-sourced on GitHub with `llms.txt`. Status: Astro 6 + adapter v13+ GA, MCP servers GA (checked 2026-05-23).

#### 2. Vercel

`@astrojs/vercel` is current and Astro-recommended (docs updated 2026-03-17). The Hobby tier — 1M function invocations, 100 GB Fast Data Transfer, 60 s max function duration — covers the MVP envelope, with the caveat that Hobby cannot deploy from Git-org repos (personal repo only). Vercel MCP (`mcp.vercel.com`, GA, OAuth) is explicitly supported by Claude Code. Loss vs. Cloudflare: paid-tier jump is $20/seat/month for Pro, password-protecting preview deploys is a $150/month Pro add-on, and image-optimization overages are a known cost trap. Status: adapter and MCP both GA (checked 2026-05-23).

#### 3. Netlify

`@astrojs/netlify` is GA with Astro 5.12+'s Vite plugin emulating functions / edge / blobs / Image CDN locally — no separate `netlify dev` server needed (docs updated 2026-01-30). `netlify deploy` defaults to a **draft preview** URL; production requires explicit `--prod`, the safer default for agent-driven workflows. `@netlify/mcp` is GA, and `docs.netlify.com/llms.txt` is live. Loss vs. Cloudflare: Edge Functions' 50 ms CPU budget is too tight for OpenRouter calls (route AI explicitly through Node serverless functions), password protection on previews is Pro-plan only, and the recent legacy-to-credit-based plan migration makes exact pricing harder to quote. Status: adapter, MCP, Skills all GA (checked 2026-05-23).

#### Honourable mentions (not shortlisted)

- **Fly.io** scores well across all five criteria and has `fly mcp launch` as a first-class agent primitive, but the workload is stateless — Fly's strength (long-lived Machines, multi-region anycast) is operational overhead the MVP doesn't need. No free tier in 2026 ($5 trial credit only) and ~$2–20/month ongoing.
- **Railway** has the strongest agent ecosystem (MCP bundled into the CLI, OAuth remote MCP, native PR preview environments). Loses on cost: no free tier, $5/month Hobby plan + usage. Strong runner-up if the workload ever grows into persistent-connection territory.
- **Render** has a GA MCP server and free template for Astro SSR, but the free Web Service spins down after 15 minutes idle with a 30–60 s cold start — unacceptable for first-time users hitting the login page. Paid Starter is $7/month/service.

## Anti-Bias Cross-Check: Cloudflare Workers + Static Assets

### Devil's Advocate — Weaknesses

1. **Astro 6 + adapter v13+ regime change is recent.** Tutorials, Stack Overflow answers, and likely agent training data still reference `Astro.locals.runtime.env` and Pages-specific build output paths that no longer work. Copy-pasted snippets will compile but fail at runtime in a way that does not surface as an obvious config error.
2. **`nodejs_compat` is mandatory for Supabase.** Without the right `compatibility_date` (≥ 2024-09-23) and the flag set in `wrangler.toml`, the `@supabase/ssr` cookie flow throws "dynamic require of stream" — a stack trace that looks like a code bug, not a config bug.
3. **Workers' execution model caps long LLM responses.** Free tier: 10 ms CPU (fine for I/O-bound LLM waits), 30 s wall-clock. Paid: 50 ms CPU, 30 s wall-clock. Long source-text generations on OpenRouter can exceed 30 s — set explicit timeouts in the client and stream tokens to the browser so the user sees progress before the wall-clock cap hits.
4. **Hybrid prerender + SSR has open Astro+Cloudflare issues** (e.g. GitHub `withastro/astro#15237`). A flashcard app that mixes a marketing landing (static) with auth-gated routes (SSR) sits exactly on that fault line.
5. **Pages still runs but is feature-frozen for Astro projects.** An agent reading older Cloudflare docs and writing against new Workers config will produce code that compiles and fails only at deploy — a class of "looks correct, fails late" trap.

### Pre-Mortem — How This Could Fail

It's late 2026. The 10xCards MVP shipped on Cloudflare Workers in May, hit a couple hundred users by July, and is now in a recurring incident loop. Three things compounded. First, the developer accepted the recommendation to use Workers without realising the `tech-stack.md` correction also meant their bootstrapped project layout was non-canonical — the starter scaffolded against Pages, they shipped against Workers, and the auth cookie/middleware path silently writes to a different cookie namespace in preview vs. production. Users complain about being randomly logged out; nobody traces it back to the runtime swap until October. Second, OpenRouter responses for long source-text pastes routinely exceed 30 seconds; the Workers wall-clock cap truncates them, so users see partial card sets and assume "the AI is bad." Third, by the time they decide to migrate to a longer-lived runtime, they've baked in Cloudflare-specific bindings (KV cache for rate limits, a Durable Object for session pinning added "temporarily") and the migration is no longer a one-day port. They blame the platform; the real cause is having picked an edge runtime for an LLM-streaming product without modelling worst-case generation duration.

### Unknown Unknowns

- **Static asset bytes are free; SSR-rendered pages count against the 100K/day free request budget.** A deck-list page that re-renders on every visit can quietly burn through the daily cap if a single user reloads aggressively. Cache aggressively at the response level for read-heavy pages.
- **`wrangler tail` is the only practical log channel without paid Workers Logs.** Open the terminal before the bug happens — by the time you `tail`, the failing request is already gone. Logpush is paid.
- **Custom domains on Workers route through Cloudflare's DNS.** Domains parked at a non-Cloudflare registrar with custom DNS need a routing chunk in the deploy plan that does not appear in the standard tutorials.
- **Preview-branch URLs on Workers do not honour the same auth-protection options Pages has.** Public preview URLs that hit a live Supabase project will let the world hit the AI generation endpoint. Split secrets into preview-vs-prod from day one and gate preview environments behind Cloudflare Access (or a separate Supabase project).
- **Cloudflare MCP servers are split across multiple URLs** (`observability.mcp.cloudflare.com`, plus docs / bindings / builds endpoints). Agents need OAuth per server or a single scoped API token — a one-token-per-everything default is not provided out of the box.

## Operational Story

- **Preview deploys**: Wrangler creates per-branch preview URLs on `*.<project>.workers.dev`. Fork PRs from external contributors are not auto-deployed (a feature, not a gap — prevents secret exfiltration). Gate preview URLs behind a Cloudflare Access policy (`access.cloudflare.com`) or split preview into a second Workers project that points at a non-production Supabase project.
- **Secrets**: `wrangler secret put <NAME>` writes to Workers Secrets (server-side, never in source). Local dev reads from `.dev.vars` (gitignored). CI reads `CLOUDFLARE_API_TOKEN` from GitHub Actions Secrets; never committed to the repo. Rotate via the Cloudflare dashboard (manual, by design) or `wrangler secret put` (overwrites).
- **Rollback**: `wrangler deployments list` shows recent deploys with IDs; `wrangler rollback <id>` reverts. Time-to-revert is seconds. Supabase migrations don't roll back automatically — if the bad deploy included a schema change, plan the rollback as code revert + Supabase manual SQL.
- **Approval**: Agent may run `wrangler deploy` to preview, `wrangler tail` for logs, `wrangler secret list` (no values shown), `wrangler kv key list`. Human-only: `wrangler deploy --env production`, `wrangler secret put` for primary auth/Supabase keys, deleting a Worker, modifying account-level settings, Supabase project deletion.
- **Logs**: `wrangler tail` (real-time, JSON output, `--status error` filter, no retention) is the free path. Cloudflare Workers Logs (paid) gives 7-day retention and the observability MCP server (`observability.mcp.cloudflare.com`) for structured queries. Agent reads logs via `wrangler tail --format=json` and parses, or via the MCP server if the user has set up the OAuth flow.

## Risk Register

| Risk | Source | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| `tech-stack.md` says `cloudflare-pages`; canonical target is now Workers + Static Assets | Research finding | H | M | Update `tech-stack.md` to `cloudflare-workers` before bootstrapping the scaffold (deploy plan step). |
| Old `Astro.locals.runtime.env` snippets fail silently | Devil's advocate | H | M | Standardise on `import { env } from "cloudflare:workers"`; reject PRs using the old pattern. Add to AGENTS.md. |
| `nodejs_compat` not set → Supabase `@supabase/ssr` throws "dynamic require of stream" | Devil's advocate | M | M | `wrangler.toml`: `compatibility_flags = ["nodejs_compat"]`, `compatibility_date = "2024-09-23"` (or later). Smoke-test login flow on first deploy. |
| OpenRouter long generations exceed Workers 30 s wall-clock | Pre-mortem | M | H | Stream tokens to browser; cap client-side request timeout at 25 s; chunk large source-text pastes server-side. |
| Hybrid prerender + SSR edge cases (Astro #15237) | Devil's advocate | L | M | Default to full SSR for auth-gated routes; static-only for marketing/login pages. Test the boundary before shipping. |
| Free-tier 100K req/day burns through on SSR-heavy traffic | Unknown unknowns | M | L | Cache deck-list responses with `Cache-Control: private, max-age=60`; consider paid Workers ($5/mo) before launch if anticipated traffic exceeds 50K/day. |
| Preview URLs leak AI-generation endpoint to public | Unknown unknowns | M | H | Gate preview URLs behind Cloudflare Access; or scaffold previews against a separate Supabase project with throwaway secrets. |
| `wrangler tail` is the only free log channel — no retention | Unknown unknowns | M | L | Accept for MVP; budget Workers Logs ($) into the first paid month if debugging becomes burdensome. |
| Custom-domain routing requires Cloudflare DNS or zone | Unknown unknowns | L | M | If the user's domain is elsewhere, add a "transfer DNS / add CNAME for Workers route" step to the deploy plan. |
| Cloudflare MCP servers split across URLs; OAuth per server | Unknown unknowns | L | L | Use a single scoped API token (Workers + Pages + Account-Read) for agent-driven MCP access; document the token's scope in the repo's onboarding doc. |
| Edge runtime lock-in if KV / Durable Objects are added "temporarily" | Pre-mortem | L | H | If KV / DO get added, document the migration cost in the same PR. Treat platform-bound storage as a deliberate architecture decision, not a quick-fix. |

## Getting Started

The following commands assume `@astrojs/cloudflare` v13+ (Astro 6) and `wrangler` v4+. Verified against the project's current `tech-stack.md` rather than generic Cloudflare tutorials.

1. **Update `tech-stack.md`**: change `deployment_target: cloudflare-pages` to `deployment_target: cloudflare-workers`. (One line; required because the adapter no longer supports Pages.)
2. **Install the adapter** (if the 10x Astro Starter didn't pre-install it):
   ```sh
   npm install @astrojs/cloudflare wrangler
   ```
3. **Configure `astro.config.mjs`**:
   ```js
   import cloudflare from "@astrojs/cloudflare";
   export default defineConfig({ output: "server", adapter: cloudflare() });
   ```
4. **Create `wrangler.toml`** at the repo root:
   ```toml
   name = "10xcards"
   main = "./dist/_worker.js/index.js"
   compatibility_date = "2024-09-23"
   compatibility_flags = ["nodejs_compat"]
   assets = { directory = "./dist", binding = "ASSETS" }
   ```
5. **Set Supabase + OpenRouter secrets** (run once per environment):
   ```sh
   wrangler secret put SUPABASE_URL
   wrangler secret put SUPABASE_ANON_KEY
   wrangler secret put OPENROUTER_API_KEY
   ```
6. **First deploy**: `npx astro build && wrangler deploy`. Confirm `https://10xcards.<account>.workers.dev` returns the login page. Run `wrangler tail` in a second terminal during the first smoke test.

The actual deploy itself runs through the host's Plan Mode (per the lesson's chain) reading this file and `tech-stack.md` together. The plan is reviewed and corrected before any mutation hits Cloudflare; the approved plan persists at `context/deployment/deploy-plan.md`.

## Out of Scope

The following were not evaluated in this research:
- Docker image configuration (not needed for Workers).
- CI/CD pipeline setup (deferred to a later lesson).
- Production-scale architecture (multi-region active-active, HA, DR — explicitly out of MVP scope).
- Custom analytics / observability pipeline beyond `wrangler tail` and Cloudflare's built-in metrics.
