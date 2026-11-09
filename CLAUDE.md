# ShapeShyft Lib

> **Git policy — never auto-commit or auto-push.** Leave work in the working tree.
> Run `git commit`, `git push`, or `gh pr create` **only when the user explicitly
> asks in that turn**. Approval for an earlier change does not
> carry forward, and finishing a task is not permission to commit it.
>
> **Do not use feature branches** — stay on the current branch.

`@sudobility/shapeshyft_lib` — business-logic layer for ShapeShyft: Zustand stores,
manager hooks that wrap `@sudobility/shapeshyft_client`, endpoint templates, and
schema-validation utilities. ESM-only library (no app/runtime of its own).

## Stack

- TypeScript (strict), target ES2020, module ESNext, moduleResolution `bundler`
- Runtime/package manager: **Bun** (do not use npm/yarn/pnpm to install)
- State: Zustand v5 (the only production dependency)
- Build: `tsc`; Test: Vitest

## Commands (verified)

```bash
bun install          # @sudobility/* resolve from GitHub Packages (needs read:packages token)
bun run build        # tsc -p tsconfig.build.json + node scripts/fix-esm-imports.mjs
bun run typecheck    # tsc --noEmit
bun run lint         # eslint src (lint:fix to autofix)
bun run test:unit    # vitest run (single pass — use this, NOT a bare "test")
bun run test:watch   # vitest watch
bun run verify       # typecheck -> lint -> test:unit -> build
```

There is **no `test` or `test:run` script**. CI = single-pass via `test:unit`.

## Architecture

```
shapeshyft_app (frontend)
  └─ shapeshyft_lib (this: stores, managers, templates, utils)
       └─ shapeshyft_client (API hooks)  [peer dep]
            └─ shapeshyft_types (types)
```

Layout under `src/`:

- `index.ts` → re-exports `./business` + `./utils` (the only public surface).
- `business/stores/` — Zustand caches: keys, projects, endpoints, settings,
  analytics (all in-memory, entity/user-scoped) + `providerModelsStore`
  (localStorage, 1h TTL via `isStale()`).
- `business/hooks/` — one manager hook per store wrapping the matching
  `shapeshyft_client` hook; plus `useEndpointTester`, `useBudgetTracker`,
  `useProjectTemplates`, `useEndpointTemplates`.
- `business/templates/` — `endpoint-templates.ts` (9 templates + `applyTemplate`)
  and `binding.ts` (`toBindingFields` / `EndpointBindingInput`).
- `utils/` — `schema-validation.ts`, `test-metrics.ts` (`extractTestMetrics`),
  `persist-storage.ts` (`createSafeJSONStorage`).

## Key patterns

- **Manager hooks prefer server data over cache.** An empty server array means
  "loading", not "no data" — cache is only a fallback; `isCached` flags it.
  Server data is synced into the store via `useEffect`; returns are `useMemo`'d.
- **Stores are global singletons, scoped by key.** Always pass the right scope
  (`entitySlug`, `userId`, or the composite). Call `clearAll()` on logout /
  entity switch.
- **`endpointsStore` uses a composite key** `"entitySlug:projectId"` via an
  internal `makeCacheKey()` — not a bare projectId.
- **Cache entries carry `cachedAt`** (ms epoch) for staleness UI.
- **`applyTemplate(template, projectName, binding, displayName?)`** — `binding`
  is `string | EndpointBindingFields`: a string is treated as a legacy
  `llm_key_id`, an object is passed through (e.g. `{ provider }` for ShapeRouter).
- **Schema validation is intentionally lightweight** — no `$ref`, combinators
  (`allOf`/`oneOf`/`anyOf`), `format`/`pattern`, or conditionals.

## Gotchas

- **Package is published to GitHub Packages, not public npm.** `.npmrc` sets
  `@sudobility:registry=https://npm.pkg.github.com`. `publishConfig.access` is
  `"public"`. Publishing runs in CI (`.github/workflows/ci-cd.yml`,
  `publish-target: github`) on push to main/develop; do not `npm publish` by hand.
- **Build rewrites ESM import specifiers.** `tsc` output alone has extensionless
  relative imports that break under Node ESM; `scripts/fix-esm-imports.mjs` adds
  `.js`. Always run the full `bun run build`, never bare `tsc`, before publishing.
- **Two persist stores (keys differ):** `providerModelsStore` →
  `"shapeshyft-provider-models"`; budget store (`useBudgetStore` in
  `useBudgetTracker`) → `"shapeshyft-budgets"`. Both use `createSafeJSONStorage`,
  which falls back to in-memory when `localStorage` is absent (SSR / Node).
- **Budget periods don't auto-reset** (`'daily'|'weekly'|'monthly'`) — callers
  handle period boundaries.
- **Peer dep on `shapeshyft_client`.** When bumping it, re-verify here; the
  managers depend on its hook shapes.

## Adding things (recipes)

- **Store:** new file in `business/stores/`, `create<State>()` with a
  `cache: Record<key, Entry>`, CRUD + `clearAll()`, set `cachedAt: Date.now()`,
  pick a scope key, wrap in `persist` + `createSafeJSONStorage` only if it must
  survive reloads; export from `stores/index.ts`.
- **Manager hook:** new file in `business/hooks/`, define
  `UseXManagerConfig`/`Return`, wrap the client hook, `useEffect`-sync to store,
  implement server-preferred fallback with `isCached`/`cachedAt`, `useMemo` the
  return; export from `hooks/index.ts`.
- **Template:** add to `endpoint-templates.ts` following `ProjectTemplate`, push
  into `ALL_TEMPLATES`, add tests.

## Downstream

`shapeshyft_app` consumes stores, managers, and templates. After changes:
`bun run verify`, then let CI publish (or local-link via `bun link` for dev).
Part of the ShapeShyft multi-project workspace (sibling repos under the parent dir).
