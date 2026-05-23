---
starter_id: 10x-astro-starter
package_manager: npm
project_name: 10xcards
hints:
  language_family: js
  team_size: solo
  deployment_target: cloudflare-workers
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
---

## Why this stack

10xCards is a solo-built web-app on a tight 3-week after-hours timeline with auth and AI generation as the two forcing constraints. The 10x Astro Starter is the recommended default for (web-app, js) and clears all four agent-friendly gates — TypeScript + Zod (typed), Astro file-based routing (convention-based), Astro + React + Supabase are mainstream in training data (popular in training data), and Astro + Supabase documentation is current and link-able (well-documented). Auth and database arrive out of the box via Supabase; edge deployment via Cloudflare Workers + Static Assets fits the medium-scale target. AI generation (FR-003, FR-004) requires adding an LLM SDK manually — standard practice across all starters in the registry; no starter ships LLM integration first-class. CI runs on GitHub Actions with auto-deploy on merge, matching the solo iteration pace. Bootstrapper confidence is first-class: the CLI is registered and valid, with occasional manual steps possible but mostly smooth scaffolding expected.
