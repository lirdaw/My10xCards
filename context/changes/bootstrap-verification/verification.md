---
bootstrapped_at: 2026-05-22T20:51:01Z
starter_id: 10x-astro-starter
starter_name: 10x Astro Starter (Astro + Supabase + Cloudflare)
project_name: 10xcards
language_family: js
package_manager: npm
cwd_strategy: git-clone
bootstrapper_confidence: first-class
phase_3_status: ok
audit_command: npm audit --json
---

## Hand-off

```yaml
starter_id: 10x-astro-starter
package_manager: npm
project_name: 10xcards
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-pages
  ci_provider: github-actions
  ci_default_flow: auto-deploy-on-merge
  bootstrapper_confidence: first-class
  path_taken: standard
  quality_override: false
  self_check_answers: null
  has_auth: true
  has_payments: false
  has_realtime: false
  has_ai: true
  has_background_jobs: false
```

## Why this stack

Astro 6 with React 19 islands is the right call for a flashcard web app with a tight 3-week after-hours timeline. The starter ships Supabase (auth + Postgres) and Cloudflare Pages out of the box, which eliminates the two biggest unknowns for a solo developer: user authentication and deployment. TypeScript and Tailwind 4 are table stakes for a well-scoped MVP. The AI card generation feature will need a separate LLM SDK added post-scaffold (no starter in the registry ships LLM integration first-class), but that addition is straightforward and does not change the stack choice.

## Pre-scaffold verification

| Signal      | Value                                                       | Severity | Notes                                   |
| ----------- | ----------------------------------------------------------- | -------- | --------------------------------------- |
| npm package | not run                                                     | —        | git-clone starter; no create-* package  |
| GitHub repo | przeprogramowani/10x-astro-starter last pushed 2026-05-17   | fresh    | from card.docs_url                      |

## Scaffold log

**Resolved invocation**: `git clone https://github.com/przeprogramowani/10x-astro-starter .bootstrap-scaffold && cd .bootstrap-scaffold && NODE_OPTIONS=--use-system-ca npm install`
**Strategy**: clone a starter repo without keeping its git history
**Exit code**: 0
**Files moved**: ~48 files + 774-package node_modules directory
**Conflicts (.scaffold siblings)**: CLAUDE.md → CLAUDE.md.scaffold
**.gitignore handling**: append-merged (14 unique scaffold lines appended under `# from 10x-astro-starter` separator; .env, .DS_Store, .idea/ de-duped as already present)
**.bootstrap-scaffold cleanup**: left in place — Windows file lock on workerd binary; safe to `Remove-Item -Recurse -Force .bootstrap-scaffold` manually once the lock clears (all project files confirmed in cwd)

**Note on initial attempt**: First run failed with `UNABLE_TO_VERIFY_LEAF_SIGNATURE` during Supabase CLI binary download (Node.js v24.14.0 Windows TLS issue). Resolved by re-running with `NODE_OPTIONS=--use-system-ca`. Clone itself was unaffected; only npm postinstall failed on first attempt.

## Post-scaffold audit

**Tool**: `npm audit --json`
**Summary**: 0 CRITICAL, 1 HIGH, 9 MODERATE, 0 LOW
**Direct vs transitive**: 0/0/2/0 direct of total 0/1/9/0 (CRITICAL/HIGH/MODERATE/LOW)

#### CRITICAL findings

None.

#### HIGH findings

**devalue** v5.6.3–5.8.0
- Advisory: GHSA-77vg-94rm-hx3p
- Description: Svelte devalue DoS via sparse array deserialization — crafted input can cause unbounded memory allocation during deserialization
- CVSS: 7.5 (AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:N/A:H)
- CWE: CWE-770 (Allocation of Resources Without Limits)
- Direct: no (transitive — pulled in by Astro internals)
- Fix available: yes (update to patched version; `npm audit fix` should resolve)

#### MODERATE findings

| Package                | Via                     | Direct | Fix available    |
| ---------------------- | ----------------------- | ------ | ---------------- |
| @astrojs/check         | @astrojs/language-server | yes   | yes (downgrade to 0.9.2, semver-major) |
| @astrojs/language-server | volar-service-yaml    | no     | yes (via @astrojs/check downgrade) |
| @cloudflare/vite-plugin | miniflare, wrangler, ws | no    | yes              |
| miniflare              | ws                      | no     | yes              |
| volar-service-yaml     | yaml-language-server    | no     | yes              |
| wrangler               | miniflare               | yes    | yes              |
| ws                     | GHSA-58qx-3vcg-4xpx (uninitialized memory disclosure, CVSS 4.4) | no | yes |
| yaml                   | GHSA-48c2-rrv3-qjmp (stack overflow via deeply nested YAML, CVSS 4.3) | no | yes |
| yaml-language-server   | yaml                    | no     | yes              |

**Context**: all MODERATE findings trace to dev tooling (Astro language server, wrangler local dev, Cloudflare vite plugin). None are production runtime dependencies — the production bundle that ships to Cloudflare Pages does not include these packages.

#### LOW / INFO findings

None.

## Hints recorded but not acted on

| Hint                    | Value                |
| ----------------------- | -------------------- |
| bootstrapper_confidence | first-class          |
| quality_override        | false                |
| path_taken              | standard             |
| self_check_answers      | null                 |
| team_size               | solo                 |
| deployment_target       | cloudflare-pages     |
| ci_provider             | github-actions       |
| ci_default_flow         | auto-deploy-on-merge |
| has_auth                | true                 |
| has_payments            | false                |
| has_realtime            | false                |
| has_ai                  | true                 |
| has_background_jobs     | false                |

## Next steps

Next: a future skill will set up agent context (CLAUDE.md, AGENTS.md). For now, your project is scaffolded and verified — happy hacking.

Useful manual steps in the meantime:
- `git init` (if you have not already) to start your own repo history.
- Review `CLAUDE.md.scaffold` — the starter ships an AI-rules file with project conventions. Merge it into (or replace) the current `CLAUDE.md`.
- Copy `.env.example` to `.env` and fill in `SUPABASE_URL` + `SUPABASE_KEY` before running `npm run dev`.
- `npm audit fix` to resolve the HIGH `devalue` finding and most MODERATE findings (the `@astrojs/check` downgrade requires `--force` and is optional — it's a lint tool only).
- To clean up: `Remove-Item -Recurse -Force .bootstrap-scaffold` once Windows releases the file lock on the workerd binary.
