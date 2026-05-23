# Cloudflare Workers Deploy Plan — 10xCards MVP

## Context

This plan ships the current 10xCards build (Astro 6 + Supabase auth + protected `/dashboard`) to **Cloudflare Workers with Static Assets**, per the recommendation in `context/foundation/infrastructure.md`. The scaffold is more deploy-ready than the research file assumed — `@astrojs/cloudflare` v13.5.0, `wrangler` v4.90.0, `wrangler.jsonc`, Supabase middleware (`src/middleware.ts`), and Supabase SSR client (`src/lib/supabase.ts`) are all wired. The plan closes the three pre-flight gaps (`tech-stack.md` still says `cloudflare-pages`, CI triggers on `master` instead of `main`, `wrangler.jsonc` lacks `account_id`), runs an end-to-end smoke test, and configures **Cloudflare Workers Builds** (Cloudflare's native git-integrated CI) so future pushes to `main` auto-deploy.

**Auto-deploy mechanism:** Workers Builds — NOT GitHub Actions. Cloudflare watches the GitHub repo, runs the build on its own infrastructure, and deploys on every push to the configured production branch. The existing `.github/workflows/ci.yml` stays in the repo but is repurposed to a **PR-time lint + build check only** (no deploy step). This keeps PR review fast and decouples "did the code compile" from "did Cloudflare ship it".

**Out of scope this deploy** (confirmed with user): AI generation (no OpenRouter wiring yet), custom domain (use `*.workers.dev`), preview/staging environment (single prod Worker). These exclusions are written down so they don't drift back in mid-execution.

**Reader assumption:** the developer is learning JS/TS via the 10xDevs course. Stack-specific terms get a one-line plain-language gloss inline.

---

## External integrations (what credential lives where)

| Service | Used for | Credentials live where |
|---|---|---|
| **Cloudflare account** | Hosts the Worker (Astro app + static assets) | `wrangler login` OAuth on your laptop. No long-lived API token required (Workers Builds runs inside Cloudflare). |
| **Cloudflare Workers Builds** | Native git-integrated CI: auto-deploys on push to `main` | Dashboard-only setup. Authorizes against GitHub via OAuth (the human clicks "Authorize Cloudflare" in GitHub). No CI secrets land in the repo. |
| **GitHub Actions** | PR-time lint + build check ONLY (no deploy) | `SUPABASE_URL` + `SUPABASE_KEY` as GitHub Actions Secrets (build-time env-schema validation). |
| **Supabase** | Auth + Postgres | Local dev: `.dev.vars` (gitignored). Workers runtime: Cloudflare Workers Secrets via `wrangler secret put`. Cloudflare build-time: Workers Builds → **Build Variables**. GitHub PR build-time: GitHub Actions Secrets. Same values in all four places. Never in source. |

---

## Research notes (verified 2026-05-23)

- `@astrojs/cloudflare` v13 dropped **Cloudflare Pages** support. Workers + Static Assets is the only target. The current `wrangler.jsonc` `main` value (`"@astrojs/cloudflare/entrypoints/server"`) is correct for v13; the snippet in `infrastructure.md` Getting Started (`./dist/_worker.js/index.js`) is stale and should NOT be followed.
- `nodejs_compat` is already set in `wrangler.jsonc` line 6. Without it, Supabase SSR throws `dynamic require of 'stream'` at runtime — an error that looks like a code bug.
- `Astro.locals.runtime.env` was removed in adapter v13. Current code reads secrets via `astro:env/server` (see `src/lib/supabase.ts` and `astro.config.mjs` env schema) which is the supported path and sidesteps Astro issue #15237 (hybrid prerender + SSR + `cloudflare:workers` env import).
- **Cloudflare Workers Builds** (Cloudflare's native CI for Workers): default deploy command is `npx wrangler deploy`, default production branch is `main`, runs Node 22.16.0 in the build image, and is configured via `dash.cloudflare.com → Workers & Pages → <Worker> → Settings → Builds`. The Worker name in the dashboard must match `wrangler.jsonc` `name` exactly, or builds fail.
- Workers Builds keeps **Build Variables** (build-time, available to `npm run build`) separate from **Workers Secrets** (runtime, available to the deployed Worker). They are two different config surfaces and BOTH need the Supabase values.
- Secrets set once via local `wrangler secret put` persist on the Worker — Workers Builds does NOT re-PUT them on every deploy.

---

## Prerequisites — one-time setup before Phase 0

**Goal:** Have all the tools, accounts, and external projects in place so Phase 0 can start cleanly. Everything below is a one-time setup — you do not repeat any of it on subsequent deploys.

### P.1 — Local toolchain

What you need on your laptop. Skip any you already have.

- [ ] **Node.js 22 LTS** (matches the `node-version: 22` in `.github/workflows/ci.yml` and the Workers Builds default).
  - **Windows:** `winget install OpenJS.NodeJS.LTS` *(PowerShell)*, or download the `.msi` from `https://nodejs.org/en/download/`.
  - **macOS:** `brew install node@22` *(Homebrew)*, or the installer from nodejs.org.
  - **Linux:** use nvm (`https://github.com/nvm-sh/nvm`) → `nvm install 22 && nvm use 22`.
  - **Verify:** `node --version` returns `v22.x.x`; `npm --version` returns `10.x.x` or higher.

- [ ] **Git** (already required because this is a git repo; check only if you're on a fresh machine).
  - **Verify:** `git --version` returns any recent version (≥ 2.30). Install from `https://git-scm.com/downloads` if missing.

- [ ] **A modern desktop browser** (Chrome, Firefox, Edge, or Safari). Needed for OAuth login flows in Phase 2 (`wrangler login`) and Phase 5 (Cloudflare GitHub App authorization).

- [x] **`wrangler` CLI — no separate install needed.** It's listed in `devDependencies` of `package.json` (`wrangler@^4.90.0`) and gets installed when you run `npm install` in Phase 1. Every command in this plan uses `npx wrangler …`, which runs the project-local copy.
  - **Verify after Phase 1's `npm install`:** `npx wrangler --version` returns `wrangler 4.x.x`.

### P.2 — External accounts

- [x] **Cloudflare account** — free tier covers everything in this plan; no card required.
  - Sign up: `https://dash.cloudflare.com/sign-up` (email + password).
  - Verify your email when the confirmation arrives.

- [x] **Supabase account** — free tier is sufficient (500 MB DB, 50K monthly active users, no card).
  - Sign up: `https://supabase.com/dashboard/sign-in`. "Continue with GitHub" is the simplest option since you already have a GitHub account.

- [x] **GitHub repository pushed to a remote.** Workers Builds in Phase 5 connects to a github.com repo, so the code must already be on GitHub.
  - **Verify:** in the repo directory, `git remote -v` shows an `origin` pointing at `https://github.com/<you>/<repo>.git` or `git@github.com:...`.
  - **If empty:** create a new repo at `https://github.com/new` (do NOT initialize it with a README — the local repo already has one), then:
    ```
    git remote add origin https://github.com/<your-username>/My10xCards.git
    git branch -M main
    git push -u origin main
    ```

### P.3 — Provision a Supabase project

- [x] **Create the project.**
  - Supabase dashboard → **New project**.
  - **Name:** `10xcards` (or your preference; not load-bearing).
  - **Region:** pick the one closest to your users. For Polish/European users → `Central EU (Frankfurt)`. *Why this matters: Supabase isn't multi-region on the free tier, and you can't change region later without recreating the project. ~200 ms round-trip from a misaligned region.*
  - **Database password:** generate a strong one and save it in your password manager. You don't need it day-to-day, but you cannot recover it later.
  - Wait ~2 minutes for provisioning. Status indicator goes from "Setting up project..." to "Active".

- [ ] **Confirm Email/Password auth is enabled.**
  - Dashboard → **Authentication → Providers → Email**. Should be **ON** by default for new projects. Toggle on if not.
  - **Confirm email** sub-toggle — decide now:
    - **ON (recommended):** users must click an email link before they can sign in. Phase 4's smoke test will include a real email round-trip — use a mailbox you can access.
    - **OFF (faster iteration):** signup → immediate signin, no email check. OK for early dev; re-enable before launch.

- [ ] **Configure Site URL and Redirect URLs** (so Supabase knows where to send users after email confirm / password reset).
  - Dashboard → **Authentication → URL Configuration**.
  - **Site URL:** `http://localhost:8787` for now (this is the URL `wrangler dev` uses in Phase 1).
  - **Redirect URLs (allow list):** add `http://localhost:8787/**`. You'll add the production `https://10xcards.<account>.workers.dev/**` here AFTER Phase 4 once you know the exact subdomain.
  - Save.

- [ ] **Locate the credentials you'll need in Phase 3** (don't copy them yet — Phase 3 has the human-only paste step).
  - Dashboard → **Project Settings → API**.
  - `Project URL` → used as `SUPABASE_URL`. Looks like `https://abcdefghijklmnop.supabase.co`.
  - `anon public` key → used as `SUPABASE_KEY`. Long base64-looking string, labelled `anon` / `public`.
  - The `service_role` key is on the same page, **labelled with a red warning** — never use it. It bypasses Row-Level Security and gives full DB access.

### P.4 — Pre-flight verification checklist

Before starting Phase 0, confirm all of these are true:

- [ ] `node --version` → `v22.x.x`.
- [ ] `git --version` → any recent version (≥ 2.30).
- [ ] `git remote -v` → shows a GitHub remote on `origin`.
- [ ] You can log in to `https://dash.cloudflare.com` and see your Cloudflare account dashboard.
- [ ] You can log in to `https://supabase.com/dashboard` and your Supabase project shows status **Active**.
- [ ] You know where the Supabase **Project URL** and **anon public** key are (no need to copy them yet — Phase 3 owns that).
- [ ] You can read your **Cloudflare Account ID** from the dashboard sidebar (Phase 0 needs it).

**If any of these are false, fix them before proceeding.** Phase 0 onwards assumes all prerequisites are met.

### P.5 — What is NOT a prerequisite

Calling these out so you don't go looking for them:

- **No Docker.** Workers don't run in a container; the local runtime is `workerd` (started by `npx wrangler dev`).
- **No Cloudflare API token** at this stage. Workers Builds authorizes via OAuth in Phase 5; the local `wrangler` uses `wrangler login` in Phase 2.
- **No domain name purchase.** This plan ships to `*.workers.dev` (the free Cloudflare-managed subdomain).
- **No OpenRouter / Anthropic account.** AI generation is out of scope for this deploy.
- **No paid Supabase plan.** Free tier covers MVP.

---

## Phase 0 — Pre-flight contract corrections

**Goal:** Fix the three small files that are wrong before any Cloudflare API call.

- [ ] **Agent:** Update `context/foundation/tech-stack.md` line 8: `deployment_target: cloudflare-pages` → `deployment_target: cloudflare-workers`. *Why: Pages and Workers are two different Cloudflare products; adapter v13 dropped Pages support.*
- [ ] **Agent:** Update `.github/workflows/ci.yml` lines 5 and 7: both `master` → `main`. As-shipped the workflow never fires (repo default is `main`).
- [ ] **Agent:** Update `wrangler.jsonc` line 3: `"name": "10x-astro-starter"` → `"name": "10xcards"`. This becomes the Worker's identifier and the `*.workers.dev` subdomain.
- [ ] **Human:** Log into `https://dash.cloudflare.com` and copy your **Account ID** (right sidebar of Account Home).
- [ ] **Agent:** Add `"account_id": "<paste-account-id>"` as a top-level field in `wrangler.jsonc` (after `"name"`). Account ID is not a secret; it's safe to commit.

**Verify:** `npx wrangler types` exits 0 (config parses cleanly). `git diff` shows exactly three files changed.

**If it fails:** "Account ID not found" later → the pasted value doesn't match the account `wrangler login` is bound to. Reopen the dashboard, pick the right account.

---

## Phase 1 — Local sanity check (Workers runtime via `workerd`)

**Goal:** Prove the app builds and runs against Cloudflare's local runtime before touching production.

- [ ] **Agent:** `npm install`
- [ ] **Agent:** `npm run build` — runs `astro build`. Expected: a `dist/` directory containing the Worker bundle and static assets. Must finish without `nodejs_compat`-related errors.
- [ ] **Agent:** `npx wrangler dev` — starts the Worker on `workerd` locally (the same runtime Cloudflare uses in production). Open `http://localhost:8787/auth/signin` in a browser; the signin page must render with no 500. (Auth itself won't work yet — secrets land in Phase 3.)

**Verify:** Signin page returns 200 at `http://localhost:8787/auth/signin`. `wrangler dev` terminal shows no red errors.

**If it fails:**
- Build error `dynamic require of 'stream'` → the `nodejs_compat` flag isn't being picked up. Confirm `wrangler.jsonc` line 6 is exactly `"compatibility_flags": ["nodejs_compat"]` and `compatibility_date` is present. If still failing, delete `node_modules/` and `.astro/`, reinstall, rebuild.
- Signin page blank → a React island failed to hydrate. Browser console will tell you. Don't ship until the page renders.

---

## Phase 2 — Cloudflare account login (HUMAN-ONLY phase)

**Goal:** Authenticate the local `wrangler` CLI to your Cloudflare account so Phases 3 and 4 can set secrets and do the manual first deploy. *No long-lived API token is needed for CI/CD* — Workers Builds (Phase 5) authorizes via OAuth in the dashboard, not via a token committed to the repo.

- [ ] **Human:** `wrangler login` — opens a browser, OAuth into Cloudflare. Bound to your laptop. Run once per machine.
- [ ] **Human:** Verify with `wrangler whoami` — output shows your email and the account ID (must match the `account_id` you added in Phase 0).

**Verify:** `wrangler whoami` prints the expected account.

**If it fails:**
- Multiple Cloudflare accounts on the same email → `wrangler` may pick the wrong one. Set `CLOUDFLARE_ACCOUNT_ID` env var explicitly, or re-run `wrangler login` and select the right account.

**Note — optional API token for manual deploys from a non-primary machine:** if you ever want to deploy from CI other than Workers Builds, or from a colleague's laptop, you can create a scoped token at `dash.cloudflare.com/profile/api-tokens` (template: **Edit Cloudflare Workers**, restricted to your account). Not needed for this plan.

---

## Phase 3 — Secrets for production

**Goal:** Get Supabase credentials onto the Worker (server-side) and into local dev.

- [ ] **Human:** Supabase dashboard → your project → Project Settings → API. Copy **Project URL** and **anon public key**. (Critically: NOT the `service_role` key, which is labelled with a red warning.)
- [ ] **Agent:** Create `.dev.vars` at repo root with the keys (values left as placeholders for the human):
  ```
  SUPABASE_URL="https://<your-project>.supabase.co"
  SUPABASE_KEY="<your-anon-public-key>"
  ```
- [ ] **Human:** Paste the real values into `.dev.vars`.
- [ ] **Agent:** `git status` — confirm `.dev.vars` does NOT appear (it's gitignored in `.gitignore`).
- [ ] **Human:** Set the production Workers Secrets (first-time `secret put` is human-only per CLAUDE.md):
  ```
  npx wrangler secret put SUPABASE_URL
  npx wrangler secret put SUPABASE_KEY
  ```
  Paste each value at the prompt.
- [ ] **Human:** Add the same values as **GitHub Actions Secrets** (Repo → Settings → Secrets and variables → Actions → New repository secret). These are needed at build time for the PR-check workflow's env-schema validation in `astro.config.mjs`:
  - `SUPABASE_URL`
  - `SUPABASE_KEY`
- [ ] *(Cloudflare Workers Builds **Build Variables** for the same values are set in Phase 5 when we wire up Workers Builds. Don't try to set them now — the Worker has to exist first.)*

**Verify:** `npx wrangler secret list` shows both names (no values — secrets are never displayed). `npx wrangler dev` and a Supabase signin with a known-good account succeeds locally.

**If it fails:**
- Signin returns "Invalid credentials" for known-good account → wrong Supabase project copied. Re-check.
- ANY signin succeeds → you pasted the `service_role` key by mistake. **Rotate the Supabase project keys** in the Supabase dashboard and redo this phase. This is a security incident.

---

## Phase 4 — First production deploy (manual, before CI is wired)

**Goal:** Ship to `*.workers.dev`, smoke-test the full auth loop end-to-end, prove rollback works.

- [ ] **Agent:** `npm run build`
- [ ] **Agent:** `npx wrangler deploy` — uploads the Worker + assets. ~30 seconds. Output ends with `https://10xcards.<account>.workers.dev`.
- [ ] **Agent:** In a second terminal, `npx wrangler tail --format=pretty` — leave open during smoke test.
- [ ] **Human:** Smoke test in a real browser (not curl — Supabase needs cookies):
  - Visit `https://10xcards.<account>.workers.dev/` → landing page renders.
  - Click **Sign up**, register a throwaway email.
  - Confirm via Supabase email (check spam).
  - Sign in.
  - `/dashboard` renders.
  - Sign out → back to signin.
- [ ] **Human:** Watch the `wrangler tail` terminal during each click — expect 200/302, no red error rows.

**Verify:** `npx wrangler deployments list` shows the deployment with a recent timestamp. End-to-end smoke passes. `wrangler tail` shows no unhandled exceptions.

**If it fails — rollback:**
- `npx wrangler deployments list`, copy the previous deployment ID, `npx wrangler rollback <id>`. Rollback is seconds.
- First deploy has no previous version → fix locally and re-run `wrangler deploy`. If you need to fully abort, `npx wrangler delete 10xcards` removes the Worker.

---

## Phase 5 — Wire Cloudflare Workers Builds auto-deploy on `main`

**Goal:** Configure Cloudflare Workers Builds (Cloudflare's native CI) so every `push` to `main` triggers a build + deploy on Cloudflare's infrastructure. Leave the GitHub Actions workflow in place as a PR-time lint+build check only.

### 5a — Adjust the PR-only GitHub Actions workflow

The Phase 0 step already fixed the trigger from `master` to `main`. The existing job (lint + build) is now correctly scoped for PR-time validation. No deploy step is added. The workflow's role is: "did the code compile and lint cleanly before we merge". The deploy is Cloudflare's job after merge.

- [ ] **Agent:** Confirm `.github/workflows/ci.yml` after the Phase 0 edit triggers on `push: branches: [main]` and `pull_request: branches: [main]`, runs `npm ci → npx astro sync → npm run lint → npm run build` with `SUPABASE_URL`/`SUPABASE_KEY` env from GitHub Actions Secrets, and has **no** `cloudflare/wrangler-action` step. Just verify; don't add anything.

### 5b — Connect the repo to Workers Builds (HUMAN, dashboard-only)

Workers Builds setup is dashboard-only — no `wrangler.jsonc` field controls it. The Worker created in Phase 4 must exist before this step.

- [ ] **Human:** `https://dash.cloudflare.com` → **Workers & Pages** → click the `10xcards` Worker (created in Phase 4).
- [ ] **Human:** **Settings → Builds → Connect**. Authorize the **Cloudflare GitHub App** when prompted (one-time GitHub OAuth — pick the org / personal account that owns the repo and grant access to just the `My10xCards` repo, NOT "All repositories").
- [ ] **Human:** Select the repository (your `My10xCards` repo).
- [ ] **Human:** Configure build settings:
  - **Production branch:** `main` *(default; should already be filled in)*
  - **Build command:** `npm run build`
  - **Deploy command:** *(leave empty to use the default `npx wrangler deploy`)*
  - **Root directory:** *(leave empty / `/`)*
  - **Build comments on pull requests:** ON (Cloudflare posts deploy URLs as PR comments — helpful even without a preview env, since the build still runs on PR commits to give you a pass/fail signal).
- [ ] **Human:** Save and exit.

### 5c — Add Build Variables (build-time Supabase env)

The `npm run build` step Cloudflare runs needs the same Supabase env vars the local build needs. These are **separate from Workers Secrets** (which are runtime).

- [ ] **Human:** Settings → **Build → Build Variables and Secrets → Add variable**. Add:
  - `SUPABASE_URL` *(type: variable — value is the project URL, not secret-sensitive)*
  - `SUPABASE_KEY` *(type: **secret** — encrypted at rest; same anon public key)*
  - Save.

> **Why both Build Variables AND Workers Secrets exist for the same value:** the Astro build process validates the env schema during `npm run build` (Cloudflare's CI step), so the values need to be readable then. At request-time, the deployed Worker reads them from `astro:env/server`, which resolves them from Workers Secrets. Two surfaces, same value. This is unavoidable until either Astro's env schema is loosened to skip build-time check or Cloudflare unifies the two surfaces.

### 5d — Trigger the first auto-deploy and verify

- [ ] **Human:** Push a one-line README change (or empty commit `git commit --allow-empty -m "chore: trigger first Workers Builds run"`) to `main`.
- [ ] **Human:** In the Cloudflare dashboard → Worker → **Builds**, watch the run progress. Expected stages: *Initializing → Cloning → Building → Deploying → Success*. Total: ~2-4 minutes.
- [ ] **Human:** Visit `https://10xcards.<account>.workers.dev/` — signin page renders, served by the Workers-Builds-produced deploy. (Compare to Phase 4 which was a manually-pushed deploy.)
- [ ] **Agent:** `npx wrangler deployments list` — shows a new deploy newer than the Phase 4 one. The source line should indicate it came from a CI run (different "deployed by" entry).

**Verify:**
- Cloudflare dashboard → Builds tab → most recent build status = Success.
- A new entry in `wrangler deployments list`.
- The `*.workers.dev` URL still works end-to-end (smoke test again, or at least signin page renders).
- GitHub repo → Settings → Integrations → Cloudflare GitHub App shows the integration is active.

**If it fails:**
- **Build fails at `npm run build`** with "Missing env SUPABASE_URL" → Build Variables not set or set under the wrong Worker. Re-check step 5c.
- **Build succeeds but deploy errors with "Worker name mismatch"** → Cloudflare dashboard's Worker name (`10xcards`) doesn't match `wrangler.jsonc` `name` field. Both must be exactly `10xcards`. (You set both in Phase 0; double-check no rename happened.)
- **Build never triggers on push** → the Cloudflare GitHub App didn't get access to the repo. GitHub → Settings → Integrations → Applications → Cloudflare → grant repo access.
- **Build triggers on every branch push (unwanted previews)** → only the production branch should auto-deploy. In dashboard → Builds → confirm "Non-production branch deploys" is OFF (or set to manual). For MVP we want main-only deploys.
- **`*.workers.dev` suddenly 500 after a CI deploy** → roll back FIRST via `npx wrangler rollback <previous-id>` (run from your laptop). Then fix forward via a new commit to `main`. **Do not** push a "revert" commit to `main` and hope it deploys — between push and Cloudflare's build completing, production stays broken. Roll back first.

---

## Phase 6 — Risk-register follow-through

**Goal:** Close each High/Medium row from `infrastructure.md`'s risk register, or explicitly defer.

- [ ] `tech-stack.md` says `cloudflare-pages` → **Closed** (Phase 0).
- [ ] `nodejs_compat` not set → **Closed** (already in `wrangler.jsonc`, proven by Phase 1 build).
- [ ] Old `Astro.locals.runtime.env` snippets → **Closed**. Add to `AGENTS.md` Tripwires: *"Read server secrets via `astro:env/server`; never `Astro.locals.runtime.env` (removed in adapter v13) and never `import.meta.env.SUPABASE_*` (leaks client-side)."*
- [ ] Hybrid prerender + SSR (#15237) → **Closed**. Add to `AGENTS.md` Tripwires: *"Don't `import { env } from 'cloudflare:workers'` — use `astro:env/server`. The first form has open bugs around hybrid prerender + SSR."*
- [ ] `wrangler tail` is the only log channel → **Accepted for MVP**. Add to `AGENTS.md` Tripwires: *"Logs: `npx wrangler tail`. No retention — open the tail BEFORE reproducing a bug."*
- [ ] **Deferred:** OpenRouter wall-clock 30s — not in this deploy. Revisit when AI feature ships.
- [ ] **Deferred:** Free-tier 100K req/day budget — add `Cache-Control: private, max-age=60` to `/dashboard` GET responses in a follow-up PR. Not blocking first deploy.
- [ ] **N/A:** Preview URL leakage (no previews scoped).
- [ ] **N/A:** Custom-domain DNS routing (no custom domain).
- [ ] **N/A:** Edge runtime lock-in via KV/DO (none used).

---

## Edge-case support steps

Each one has a recognisable signature and a one-line fix.

1. **`wrangler deploy` aborts with "Could not find account_id"** → Phase 0 step skipped. Add `"account_id"` field to `wrangler.jsonc`.
2. **Signin succeeds locally but next page shows logged out on `*.workers.dev`** → cookie domain mismatch. In `src/lib/supabase.ts`, do NOT set an explicit `domain` in cookie options; let the browser default to the request host. Verify the cookie has `HttpOnly`, `Secure`, `SameSite=Lax`.
3. **Runtime error `dynamic require of 'stream'`** → `nodejs_compat` not active on the deployed Worker. Re-check `wrangler.jsonc`, redeploy. If still failing, wipe `node_modules/` + `.astro/` + `dist/`, reinstall, rebuild, redeploy.
4. **One specific page 500s with `env is undefined`, the rest work** → Astro issue #15237 family. A `prerender = true` page is reading server secrets. Either drop the prerender flag, or move the secret-reading into an API route the page calls.
5. **`wrangler deploy` hangs ~30s and times out** → re-run; `wrangler deploy` is idempotent. If it consistently times out, check `dist/` for files > 25 MB (the per-file cap on the Static Assets binding).
6. **Workers Builds deployed but local `wrangler deploy` warns "version conflict"** → your local state is stale, not ahead. `npx wrangler deployments list` is the source of truth. Pull `main`, then redeploy only if you actually have a change.
7. **Workers Builds queued / "Initializing" doesn't progress** → temporary platform delay. Check `https://www.cloudflarestatus.com/` for incidents. If clean, cancel the build in the dashboard and re-trigger via `git commit --allow-empty -m "retry build" && git push`.
8. **Cloudflare GitHub App revoked or repo access removed** → Workers Builds stops triggering silently (no deploys on `main` pushes). Fix: GitHub → Settings → Integrations → Applications → Cloudflare → re-grant repo access. Then re-confirm from the Worker's Builds tab in Cloudflare dashboard.
9. **Build Variable typo (e.g. `SUPABASE_URI` instead of `SUPABASE_URL`)** → build fails with env-schema error. Astro's error message names the missing key; fix in Cloudflare dashboard → Settings → Build → Build Variables. Build Variables update does NOT auto-retrigger the build — push an empty commit or click "Retry build" in the dashboard.

10. **`wrangler` fails locally with `UNABLE_TO_VERIFY_LEAF_SIGNATURE` / "Failed to fetch auth token: TypeError: fetch failed"** → Node's bundled CA store doesn't trust the cert presented by `api.cloudflare.com`. Usually caused by antivirus HTTPS scanning, corporate VPN, or endpoint protection re-signing TLS with a custom root CA that's in the Windows cert store but not in Node's bundle. **Fix (Node 22+):** make Node use the system CA store via `NODE_OPTIONS=--use-system-ca`. Persistent one-shot from PowerShell: `setx NODE_OPTIONS "--use-system-ca"` — takes effect in **new** terminals only (existing sessions keep the old env). This affects only LOCAL `wrangler` use; Workers Builds in Phase 5 and GitHub Actions runners are unaffected (they don't sit behind the same TLS interceptor).

---

## Critical files

- `wrangler.jsonc` — add `account_id`, rename to `10xcards`.
- `.github/workflows/ci.yml` — branch fix `master`→`main` ONLY. No deploy step (Workers Builds owns deploys).
- `context/foundation/tech-stack.md` — fix `deployment_target` value.
- `AGENTS.md` — add three Tripwires (Phase 6).
- `.dev.vars` — create (gitignored).
- `src/middleware.ts` and `src/lib/supabase.ts` — read-only references; already wired correctly, no changes.
- `astro.config.mjs` — read-only reference; env schema (`envField` for `SUPABASE_URL`/`SUPABASE_KEY`) drives local dev, GitHub Actions PR build, AND Workers Builds build requirements.

---

## End-to-end verification

After Phase 5 completes, the following should all be true:

1. `https://10xcards.<account>.workers.dev/auth/signin` returns 200 with a working signin form.
2. Signing up a new email → confirming → signing in → seeing `/dashboard` → signing out works in a real browser.
3. `npx wrangler deployments list` shows at least two deploys: the manual one from Phase 4 and the Workers-Builds-driven one from Phase 5 (different "deployed by" entry on the latter).
4. `npx wrangler tail --format=pretty` against a fresh signin shows JSON log rows with 200/302 status, no exceptions.
5. `npx wrangler rollback <previous-id>` returns the previous version within seconds (verify by visiting the URL and confirming the rolled-back state, then rollback to current).
6. Pushing a no-op change to `main` triggers (a) GitHub Actions lint+build = green AND (b) Cloudflare Workers Builds = Success → a new deploy lands on `*.workers.dev`.
7. `npx wrangler secret list` shows `SUPABASE_URL` and `SUPABASE_KEY` (names only).
8. Cloudflare dashboard → Worker → **Builds** tab shows the recent successful build, sourced from the GitHub repo's `main` branch.
9. Opening any PR against `main` triggers the GitHub Actions lint+build job and (per Workers Builds settings) a Cloudflare PR comment with the build status. Production is unaffected by PR builds.
