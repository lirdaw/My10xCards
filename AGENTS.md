# Repository Guidelines

10xCards is an Astro 6 SSR app: React 19 islands, Tailwind 4, Supabase auth, deployed to Cloudflare Workers. See @README.md for setup, Supabase local stack, and deploy steps.

## Tripwires

- **CI triggers on `master`; this repo's default branch is `main`** (@.github/workflows/ci.yml). Pushes to `main` will not run lint/build. Fix the workflow before treating CI as a merge gate.
- **Secrets only via the Astro env schema.** `SUPABASE_URL` / `SUPABASE_KEY` are declared in `astro.config.mjs` (`env.schema`, `context: "server"`, `access: "secret"`) and imported from `astro:env/server` (see `src/lib/supabase.ts`). Never read them from `import.meta.env.*` — that path exposes them client-side.
- **`output: "server"`** in `astro.config.mjs`: dynamic-by-default. Static pages opt in with `export const prerender = true`. Do not add `prerender = false` to API routes — it's redundant noise.
- **No Next.js directives.** `"use client"` / `"use server"` are meaningless in Astro. Hydrate React islands with `client:load` / `client:idle` / `client:visible` on the JSX tag.
- **Tailwind class composition via `cn()`** from `@/lib/utils` (clsx + tailwind-merge). Never concatenate class strings manually.

## Commands

- `npm run dev` — Astro dev server on Cloudflare workerd runtime.
- `npm run build` — production build.
- `npm run lint` / `npm run lint:fix` — ESLint (typescript-eslint, react-compiler, a11y).
- `npm run format` — Prettier (`prettier-plugin-astro`, `prettier-plugin-tailwindcss`).
- `npx astro sync` — regenerate `.astro/types.d.ts` after env or content-schema changes; CI runs this before lint.

Husky `pre-commit` runs `npx lint-staged`: ESLint `--fix` on `*.{ts,tsx,astro}`, Prettier on `*.{json,css,md}` (@package.json `lint-staged`).

## Layout & conventions

- Path alias `@/*` → `./src/*` (@tsconfig.json). Use it for cross-folder imports; never `../../`.
- `src/pages/` are routes; `src/pages/api/` are server endpoints — uppercase `GET` / `POST` exports, `APIRoute` type (see `src/pages/api/auth/signin.ts`).
- `src/components/ui/` is shadcn/ui (style `new-york`, base `neutral`, icons `lucide`). Add components via `npx shadcn@latest add <name>` — do not hand-write.
- `src/lib/` holds services and helpers; `src/lib/utils.ts` exports `cn()`.
- Auth: `src/middleware.ts` populates `context.locals.user` and gates the `PROTECTED_ROUTES` array.
- TypeScript extends `astro/tsconfigs/strict`. `App.Locals` typed in `src/env.d.ts`.

## Commits

Conventional with a milestone tag, observed in `git log`: `<type>: [m<X>l<Y>] <subject>`. Types seen so far: `chore`, `docs`.

## Deeper context

- @README.md — setup, Supabase local stack, deploy.
- @context/foundation/prd.md — product requirements.
- @context/foundation/tech-stack.md — stack rationale.
