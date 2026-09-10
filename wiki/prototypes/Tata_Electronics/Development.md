# Codebase Fact-Finding Report — testflow-enterprise-main

_(Read-only; all values below are quoted or paraphrased directly from files actually opened. No inference beyond stated text.)_

## 1. Tech Stack

Source: `package.json` (full), `tsconfig.json` (full), `vite.config.ts` (full), `components.json` (full)

- **Language**: TypeScript `^5.8.3`, `"type": "module"`, package name `"tanstack_start_ts"`.
- **Framework**: TanStack Start (`@tanstack/react-start: 1.168.32`, `@tanstack/react-router: 1.170.18`, `@tanstack/router-plugin: 1.168.23` — exact pins, no caret), React `^19.2.0` / `react-dom ^19.2.0`.
- **Build**: Vite `^8.2.0`, wrapped via `@lovable.dev/vite-tanstack-config` (`^2.14.0`) — comment in `vite.config.ts` states this wrapper already provides "TanStack devtools, tanstackStart, viteReact, tailwindcss, tsConfigPaths, nitro..., VITE_* env injection, @ path alias, React/TanStack dedupe, error logger plugins, and sandbox detection."
- **SSR runtime**: Nitro `3.0.260603-beta`, preset `"node-server"` (`vite.config.ts` line 23).
- **UI**: ~25 `@radix-ui/react-*` packages, Tailwind CSS `^4.2.1` + `@tailwindcss/vite`, shadcn/ui config (`components.json`: `"style": "new-york"`, `"iconLibrary": "lucide"`, css at `src/styles.css`, `"baseColor": "slate"`), `lucide-react`, `cmdk`, `vaul`, `sonner`, `recharts`, `embla-carousel-react`.
- **Forms**: `react-hook-form ^7.71.2`, `@hookform/resolvers ^5.2.2`, `zod ^3.24.2`.
- `tsconfig.json`: `target: ES2022`, `jsx: react-jsx`, `strict: true`, path alias `"@/*": ["./src/*"]`; `include` covers `src/**/*.ts(x)`, `vite.config.ts`, `vitest.config.ts`, `playwright.config.ts`, `e2e/**/*.ts`, `eslint.config.js`.

## 2. Dependencies/Versions

Full exact strings captured from `package.json` — see detailed list (all deps/devDeps enumerated with exact version strings above in the working notes). Notable exact (non-caret) pins: `@tanstack/react-router 1.170.18`, `@tanstack/react-start 1.168.32`, `@tanstack/router-plugin 1.168.23`, `nitro 3.0.260603-beta`.

**Package manager evidence**: Both `bun.lock` (160,313 bytes) and `package-lock.json` (318,796 bytes) exist at repo root — dual lockfiles present. `bunfig.toml` exists:

```
[install]
saveTextLockfile = true
minimumReleaseAge = 86400
minimumReleaseAgeExcludes = ["@lovable.dev/vite-tanstack-config", "@lovable.dev/mcp-js", "@lovable.dev/email-js", "@lovable.dev/webhooks-js"]
```

CI (`.github/workflows/deploy.yml`) uses `oven-sh/setup-bun@v2` and `bun install --frozen-lockfile` — Bun is CI-authoritative.

## 3. Build/Package Tooling

`package.json` `"scripts"` (verbatim):

```
dev: vite dev
build: vite build
build:dev: vite build --mode development
build:gh-pages: vite build && node scripts/build-gh-pages.mjs
preview: vite preview
lint: eslint .
typecheck: tsc
test: vitest run
test:watch: vitest
e2e: playwright test
format: prettier --write .
```

`vite.config.ts`: `base = process.env["GH_PAGES_BASE"] || "/"`; `tanstackStart.server.entry = "server"` (routes to `src/server.ts`); `nitro.preset = "node-server"`. No explicit `outDir` override seen. `tsconfig.json` has `noEmit: true` (Vite/Nitro own the build output). `scripts/build-gh-pages.mjs` post-processes the Nitro `node-server` build (`.output/server/index.mjs`, `.output/public`) into a static `dist/` — boots the server on port 8790, captures rendered HTML into `dist/index.html`, writes an SPA-fallback `dist/404.html`, copies `.output/public`, and writes empty `dist/.nojekyll`.

## 4. Configuration/Env Vars

- **No `.env`, `.env.example`, or any `.env*` file found** anywhere in the repo root.
- Grep of `src/` for `import.meta.env`: **zero matches**.
- Grep of `src/` for `process.env`: **zero matches**.
- Grep of `src/` for `VITE_`: **zero matches**; the only repo-wide (non-`node_modules`) hit is a comment in `vite.config.ts` describing the wrapper package's "VITE_* env injection" feature — no actual `VITE_*` var is defined or read in this repo's own code.
- The only env vars actually read in this repo's own code:
    - `GH_PAGES_BASE` — `vite.config.ts` line 13 and `scripts/build-gh-pages.mjs` line 26; set by CI in `.github/workflows/deploy.yml` line 54 (`GH_PAGES_BASE: /${{ github.event.repository.name }}/`).
    - `CI` — `playwright.config.ts` line 34 (`reuseExistingServer: !process.env["CI"]`).
    - `PORT`, `NODE_ENV` — set (not read) by `scripts/build-gh-pages.mjs` when spawning its server subprocess.
- **FLAG: no `.env.example` file found. No environment-variable-driven external service configuration (Supabase URL/key, API endpoints, etc.) found anywhere in `src/`.**

## 5. Test Setup

`vitest.config.ts` (full): deliberately independent of `vite.config.ts` ("that config pulls in the TanStack Start / nitro build plugins, which add unnecessary weight"). `resolve.alias`: `@` → `./src`; `test.environment: "jsdom"`; `test.setupFiles: ["./src/test/setup.ts"]`; `test.include: ["src/**/*.test.{ts,tsx}"]`.

`playwright.config.ts` (full): `testDir: "./e2e"`, `fullyParallel: false`, `workers: 1`, `timeout: 120_000`, `reporter: [["list"]]`, `use.baseURL: http://localhost:4173`, `trace: "retain-on-failure"`, `permissions: ["geolocation"]` with fixed mock coordinates (12.7409, 77.8253 — "Hosur, Tamil Nadu" per code comment), `webServer.command: "bun run dev -- --port 4173 --strictPort --host 127.0.0.1"`, `webServer.reuseExistingServer: !process.env["CI"]`, single project `chromium` (Desktop Chrome), with optional sandboxed Chromium path override.

`e2e/` directory: `critical-workflow.spec.ts`, `supervisor-association.spec.ts`, `supervisor-dashboard-viewport.spec.ts`, `tester-status-tabs-screenshots.spec.ts`, `tester-status-tabs.spec.ts`, `worksheet-viewport.spec.ts`, plus `fixtures/evidence.txt`, `rerun-log.txt`, and 15 PNG files under `screenshots/`.

Test-related devDependencies: `vitest ^4.1.10`, `jsdom ^30.0.1`, `@testing-library/react ^16.3.2`, `@testing-library/jest-dom ^7.0.1`, `@testing-library/user-event ^14.6.4`, `@playwright/test ^1.62.1`.

## 6. External Integrations

- Grep of `src/` for `supabase` / `createClient`: **zero matches**. Confirmed no `supabase` string in `package-lock.json` or `bun.lock` either.
- Grep of `src/` for `axios`: **zero matches**.
- `fetch(` matches in `src/` are internal only: `src/server.ts` (TanStack Start SSR handler wrapping `@tanstack/react-start/server-entry`'s own `fetch` method) — no third-party API client.
- Grep for stripe/analytics/auth0/firebase/clerk/amplitude/segment/mixpanel/sentry/posthog: all matches are false positives (plain English/domain words — e.g. a comment "analytics belong to managers" in `src/routes/reports.tsx`, a command palette label "reports analytics" in `src/components/tms/CommandPalette.tsx`, and an unrelated `SegmentedControl` UI component / `statusSegment` state variable in `src/routes/executions.$executionId.tsx`). No actual SDK imports found.
- **FLAG: no third-party SDK integrations (auth, payments, analytics, BaaS) found in `src/`.** Per a code comment in `vite.config.ts`: "The whole app is client-rendered (all state lives in TmsProvider/localStorage, no route uses loaders or server functions)."

## 7. Deployment / GitHub Pages Workflow

`.github/` contains exactly one file: `.github/workflows/deploy.yml` (full contents read).

- **Name**: "Build, test and deploy"
- **Triggers**: `push` to `[main]`, `pull_request` to `[main]`, `workflow_dispatch`.
- **Concurrency**: group `pages-${{ github.ref }}`, `cancel-in-progress: true`.
- **Job `build-and-test`** (ubuntu-latest): `actions/checkout@v4` → `oven-sh/setup-bun@v2` (bun-version `latest`) → `bun install --frozen-lockfile` → `bun run lint` → `bun run typecheck` → `bun run test` → `bunx playwright install --with-deps chromium` → `bun run e2e` → upload Playwright report (`actions/upload-artifact@v4`, always, 7-day retention) → build step "Build static site for GitHub Pages" with `env: GH_PAGES_BASE: /${{ github.event.repository.name }}/` running `bun run build:gh-pages` → `actions/upload-pages-artifact@v3` with `path: dist` (only if `github.ref == 'refs/heads/main' && github.event_name == 'push'`).
- **Job `deploy`** (needs `build-and-test`, same branch/event gate): permissions `pages: write`, `id-token: write`; environment `github-pages`; steps `actions/configure-pages@v5` → `actions/deploy-pages@v4`.
- `vite.config.ts` `base`: `process.env["GH_PAGES_BASE"] || "/"` — becomes `/<repo-name>/` only in the CI Pages build; local dev/build stays `/`.
- `package.json`: **no `homepage` field**. Deploy script: `build:gh-pages: vite build && node scripts/build-gh-pages.mjs`.
- `public/` contents: `favicon.ico`, `pibythree-mark.png`, `robots.txt`. **No CNAME, no `.nojekyll` committed** — `.nojekyll` is generated at build time by `scripts/build-gh-pages.mjs` (writes empty `dist/.nojekyll`).
- **FLAG: no CNAME file found (no custom domain configured). No `homepage` field in `package.json`.**

---

**Files inspected in full**: `package.json`, `tsconfig.json`, `vite.config.ts`, `components.json`, `bunfig.toml`, `.gitignore`, `.gitguardian.yaml`, `vitest.config.ts`, `playwright.config.ts`, `AGENTS.md`, `.github/workflows/deploy.yml`, `scripts/build-gh-pages.mjs`, `src/server.ts`.

**Inspected via directory listing / targeted grep only** (not read in full): individual `e2e/*.spec.ts` files (contents not opened, only filenames enumerated), the full `src/` tree (targeted greps for `import.meta.env`, `process.env`, `supabase`, `VITE_`, `createClient`, `axios`, `fetch(`, and integration SDK names were run, but not every source file was read line-by-line), `package-lock.json`/`bun.lock` (grepped for `supabase` only, not read in full given their size), `.lovable/project.json` (found to exist, not opened).

**Note on the earlier message**: a status-check request addressed to "Codebase Onboarding Engineer" arrived mid-task; I attempted to reply via SendMessage but the tool reported no reachable agent by that name (possibly a different session or an agent that has since ended). The full findings above are what would have been sent.

# Codebase Orientation Map

## 1-Line Summary

This is "Pibythree Quality Hub," a TanStack Start/React 19 client-rendered SPA (Lovable-built) that models a factory digital-quality-inspection workflow — assign, execute, review, retest, approve — entirely in browser memory/localStorage, with no real backend, database, or authentication server.

## 5-Minute Explanation

- **Primary tasks in code**: employee login (demo/hardcoded) → OTP → tester-only location/station verification → tester executes a checklist against a manufactured unit (pass/fail/measurement/N/A, with evidence photos) → submits for review → quality checker approves/rejects/requests retest → optional retest cycle → completion. Also: template (checklist) authoring, and admin management of users/plants/stations/devices/assignments.
- **Primary inputs**: form fields (login, OTP, check results, review comments), file uploads read via `FileReader.readAsDataURL`, browser Geolocation (`navigator.geolocation`), Cmd/Ctrl+K command palette.
- **Primary outputs**: mutated `AppState` persisted to `localStorage` via a `repository` object; sonner toast notifications; rendered worksheets/dashboards/review queues. No network requests to any external API.
- **Key files**:
    - `src/router.tsx` — TanStack Router instance + `QueryClient` creation
    - `src/routes/__root.tsx` — app shell, `TmsProvider`, `QueryClientProvider`, 404/error components
    - `src/lib/tms/store.tsx` — `TmsProvider`/`TmsContext`, the entire client state container
    - `src/lib/tms/services.ts` — all business logic/state transitions as pure functions
    - `src/lib/tms/permissions.ts` — RBAC checks
    - `src/lib/tms/seed.ts` — the entire demo dataset (only data source; no DB)
    - `src/types/domain.ts` — canonical domain model, enums, state machine
    - `src/routes/executions.$executionId.tsx` — the core test-execution worksheet page
- **Main code paths**: route file (`src/routes/*.tsx`) reads `useTms()` state → calls a `services.ts` function through `run()` → `services.ts` returns `Result<T>` + new `AppState` → `store.tsx` persists to `repository`/localStorage and fires a toast → route re-renders from updated context.

## Deep Dive

**Type**: Single-page demo/prototype web app built on TanStack Start but explicitly shipped as a static client-rendered SPA (no SSR data loading, no server functions used by any route). **Primary runtime(s)**: Browser (React 19.2.0); Node.js only runs the dev/build/Nitro wrapper, which does no app-specific work beyond error hardening.

### Entry points

- `src/router.tsx` — creates the TanStack `Router` and a `QueryClient`, passed into route context.
- `src/routes/__root.tsx` — `createRootRouteWithContext<{queryClient: QueryClient}>()`. `RootShell` renders `<html>/<head>/<body>` (`HeadContent`, `Scripts`). `RootComponent` wraps `<Outlet/>` in `<QueryClientProvider><TmsProvider>...<Toaster position="bottom-right" richColors/></TmsProvider></QueryClientProvider>`. Also defines `NotFoundComponent` and `ErrorComponent` (the latter calls `reportLovableError`).
- `src/start.ts` — `createStart()` registers `requestMiddleware: [errorMiddleware, csrfMiddleware]`: a try/catch server wrapper (`renderErrorPage()` fallback) plus `createCsrfMiddleware({filter: ctx => ctx.handlerType === "serverFn"})`. Code comment states this file exists specifically to keep CSRF protection active.
- `src/server.ts` — custom SSR entry (`export default {fetch(...)}`) wrapping `@tanstack/react-start/server-entry`, with `normalizeCatastrophicSsrResponse` catching h3-swallowed 500s to always return a rendered error page instead of raw JSON.
- `vite.config.ts` — wraps `@lovable.dev/vite-tanstack-config`; sets `nitro.preset: "node-server"`; comment explicitly states "the whole app is client-rendered (all state lives in TmsProvider/localStorage, no route uses loaders or server functions), so it ships as a static SPA instead of the Cloudflare Worker Lovable defaults to."
- `src/routeTree.gen.ts` (auto-generated, "do NOT make changes") — confirms 17 flat routes, all direct children of the root route (no nested pathless layouts).

### Server/client integration

No API routes, no server functions, no SSR loaders in any route read. `src/server.ts`/`src/start.ts` exist only as framework-required SSR/CSRF scaffolding, not an app backend. `Glob src/integrations/**` returned nothing; a case-insensitive repo-wide `Grep` for `supabase|createClient|SUPABASE` under `src/` returned **zero matches** — confirmed no Supabase/BaaS client despite the Lovable origin. All persistence goes through a `repository` object backed by `localStorage` in `src/lib/tms/store.tsx`.

### Router/routes/layouts

TanStack Router, file-based, generated into `src/routeTree.gen.ts`. All 17 routes are flat children of root: `/`, `/admin`, `/dashboard`, `/my-tests`, `/otp`, `/reports`, `/verify-location`, `/verify-station`, `/executions/$executionId`, `/reviews/`, `/reviews/$executionId`, `/templates/`, `/templates/$templateId`, `/templates/categories`, `/templates/import`, `/templates/test-cases`, `/units/$unitId`. There is no shared pathless layout route in the router tree — instead each authenticated page wraps itself in `<AppShell>` (`src/components/tms/AppShell.tsx`), i.e. shared chrome/guard behavior is applied per-page, not via router nesting.

### Authentication/protection/roles (exact route-level guards)

- **Login** (`src/routes/index.tsx`): employeeId + password submitted to a demo/hardcoded check in `services.ts`, sets `pendingLoginUserId`, navigates to `/otp`.
- **OTP** (`src/routes/otp.tsx`): verifies a demo OTP, establishes `state.session`.
- **`/verify-location`** and **`/verify-station`**: each implements its **own** `useEffect` redirect guard (duplicated logic, not centralized): no user → `/`; wrong role → `/dashboard`; steps attempted out of order → redirect to the correct step; already-verified → skip forward. Device Geolocation (`navigator.geolocation.getCurrentPosition`) is captured in `verify-location.tsx` and stored as `session.deviceGeo` as corroborating metadata only — a comment on `Session.deviceGeo` in `src/types/domain.ts` and matching logic in `verify-location.tsx` state explicitly it never gates or auto-selects, since "this app has no known-good plant coordinates to check it against." Manual Plant→Location→Station selection by the tester remains authoritative.
- **RBAC**: `src/lib/tms/permissions.ts` — `canExecuteTest`, `canReviewExecution`, `canManageTemplates`, `canManageUsers`, `canManagePlants`, `canManageStations`, `canManageDevices`, `canManageAssignments`, `canManageFailureCategories`, `isAssignedChecker`, `canViewReview`. These gate both route-level rendering (e.g., `admin.tsx` conditional `TabsTrigger`s) and individual action buttons across pages.
- **Roles** (`src/types/domain.ts`): `tester` ("Quality Technician"), `quality_checker`, `manager` ("Supervisor"), `template_manager`, `admin`.
- **Not present**: no centralized route-guard hook/HOC — each page duplicates its own auth check; no server-side session validation; no JWT/cookie-based auth.

### Pages/modules (confirmed by direct read)

- `src/routes/index.tsx`, `otp.tsx`, `dashboard.tsx`, `my-tests.tsx` — login, OTP, home dashboard, tester's assigned-work list.
- `src/routes/executions.$executionId.tsx` — the test-execution worksheet (see workflow trace below).
- `src/routes/verify-location.tsx`, `verify-station.tsx` — tester pre-execution checkpoints.
- `src/routes/reviews.index.tsx` (`ReviewQueuePage`) — lists `PENDING_REVIEW` executions with "My queue only" (via `isAssignedChecker`) and "Failures only" `Switch` filters, plus a "Recently decided" list (`APPROVED|REJECTED|COMPLETED|RETEST_REQUIRED`).
- `src/routes/reviews.$executionId.tsx` (`ReviewPage`) — reviewer's counterpart to the worksheet: filter tabs (All/Failures/Skipped/Retest/Evidence), per-check `Sheet` detail (expected/observed, failure info, "Quality Intelligence" `similarFailures()` panel, evidence thumbnails, prior attempts). Decision panel requires comment ≥15 chars to `rejectExecution`/`requestRetest` (no minimum for `approveExecution`); retest requires ≥1 selected `checkId`. "Skipped" is presentation-only (checks with no resolved `CheckResult` this round), not new domain state.
- `src/routes/templates.index.tsx` (`TemplatesPage`) — template families grouped by `familyCode`, revisions sorted descending; template_manager/admin can `createTemplate` via `Dialog`.
- `src/routes/admin.tsx` (`AdminPage`) — tabbed console: Users (role `Select`, active `Switch`, `createUser`), Plants & Stations (`createPlant`, `createLocation`, `createStation`), Devices (`createDevice`, online/offline `Switch`), Units & Assignments (`createUnit`, `createAssignment` with auto-suggested checker via `associatedQualityCheckerId`, `ReassignSheet`), Failure Categories (`addFailureCategory`), and an Audit tab (last 40 `state.audit` events via `ActivityTimeline`). Each tab conditionally rendered per matching `canManage*` check.
- Referenced but not read this pass: `reports.tsx`, `templates.$templateId.tsx`, `templates.categories.tsx`, `templates.import.tsx`, `templates.test-cases.tsx`, `units.$unitId.tsx`.

### Key reusable components

- shadcn/ui primitives ("new-york" style per `components.json`): `Button`, `Dialog`, `Sheet`, `Select`, `Tabs`, `Switch`, `Checkbox`, `Input`, `Label`, `Textarea`, `AlertDialog`, `Progress`, `Command` family.
- `src/components/tms/badges.tsx` — `StatusBadge` (role-aware label via `statusLabel(status, role)`, `EXECUTION_TONE` map), `CheckStatusBadge` (`CHECK_TONE` map), `PriorityBadge` (`PRIORITY_TONE` map).
- `src/components/tms/CommandPalette.tsx` — global Cmd/Ctrl+K search over Units, Executions (routes to `/reviews/$id` or `/executions/$id` depending on role/`canViewReview`), Checks, and static nav shortcuts.
- `src/components/tms/AppShell.tsx` — shared page chrome/guard wrapper invoked per-page.
- Not opened directly: `Timeline.tsx`, `EmptyState.tsx`, `ReassignSheet.tsx`, `AccessSteps.tsx`, `Logo.tsx`, `AssignCheckerDialog.tsx`.

### State/data services

No Redux/Zustand. Global state is React Context: `TmsProvider`/`TmsContext` in `src/lib/tms/store.tsx`, exposing `{state, run}` via `useTms()`. `run(fn, {success?})` applies a pure state-transition function from `services.ts`, persists the result, fires a sonner toast. A TanStack Query `QueryClient` is instantiated in `router.tsx` and provided in `__root.tsx`, but **no route file inspected uses `useQuery`/`useMutation`** — all reads are synchronous selectors against `state` (e.g., `currentUser`, `executionById`, `templateById`, `executionProgress`). Business logic centralizes in `src/lib/tms/services.ts` as pure `(state, actor, ...) → Result<T>` functions: `startExecution`, `saveCheckResult`, `addEvidence`, `submitExecution`, `resumeForRetest`, `approveExecution`, `rejectExecution`, `requestRetest`, `createTemplate`, `createUser`, `createPlant`, `createLocation`, `createStation`, `createDevice`, `createUnit`, `createAssignment`, `setUserRole`, `setUserActive`, `setStationStatus`, `setDeviceStatus`, `addFailureCategory`, `similarFailures`, `failureHotspots`, `validateSubmission`.

### Persistence

No real database. `src/lib/tms/seed.ts` (2876 lines, fully read) builds the entire initial `AppState` in memory: 9 users across all 5 roles, 1 plant (Hosur), 2 locations, 4 stations (1 in maintenance), 4 devices, 1 published template (revision 3), 9 units, 9 assignments, 9 executions covering every `ExecutionStatus`, per-check `CheckResult` "attempt" rows retained permanently and never mutated (e.g., attempt 1 failed at 94.2 dB vs. a 90 dB limit, attempt 2 `retest_required`), 6 evidence items (base64 data URLs), 3 reviews (one per `ReviewDecision`), 7 audit events. This seed loads into `TmsProvider` and is subsequently mutated/persisted via `repository`/localStorage in `store.tsx`. No DB client, migration folder, or schema file exists anywhere in `src`.

### Loading/error behavior

- `src/routes/__root.tsx` — router-level `NotFoundComponent` (404 + "Go home") and `ErrorComponent` (generic failure page, "Try again"/"Go home", calls `router.invalidate()` and `reportLovableError(error, {boundary: "tanstack_root_error_component"})`).
- `src/start.ts` `errorMiddleware` catches server-function exceptions, returns `renderErrorPage()` fallback for anything without a `statusCode`.
- `src/server.ts` `normalizeCatastrophicSsrResponse` detects h3's swallowed-error JSON shape (`{unhandled:true, message:"HTTPError"}`) and substitutes the same rendered error page instead of leaking raw JSON.
- Toasts (sonner, `<Toaster position="bottom-right" richColors/>`) fire from every `run()` call in `store.tsx`, success and failure alike.
- `executions.$executionId.tsx` adds page-specific UX: `useOnlineStatus()` hook (`navigator.onLine` + `online`/`offline` events, "Offline — saved to this device"), a per-check `saveState` machine (`idle|dirty|saving|saved`), and `useBlocker`/`beforeunload` guards against losing unsaved work.
- Not inspected: `src/lib/error-capture.ts`, `src/lib/error-page.ts`, `src/lib/lovable-error-reporting.ts`, `src/lib/utils.ts`, `src/hooks/use-mobile.tsx` (existence/import sites confirmed, internals not read).

### Business workflows (traced through code, not docs)

- **Assignment → execution**: `admin.tsx`'s `NewAssignmentDialog` calls `createAssignment(s, user, {unitId, templateId, testerId, stationId, dueAt, priority, qualityCheckerId})`, auto-suggesting the tester's `associatedQualityCheckerId` as default checker, with a UI note when overridden/absent.
- **Execute a test** (`executions.$executionId.tsx`): `ASSIGNED` + `canExecuteTest` → "Start execution" → `startExecution` → `IN_PROGRESS`. Per check: tester fills a `CheckDraft`; client-side `checkSaveProblems()` mirrors server-side `validateSubmission` rules (failure category + description ≥5 chars + evidence required when "failed"; N/A blocked unless `activeCheck.allowNA`); `save()` calls `saveCheckResult(...)`. Evidence upload reads via `FileReader.readAsDataURL`, then `addEvidence(...)` stores base64 directly in state — no blob storage. Submit shows a confirmation `Dialog` with progress counts and `validateSubmission` problems ("Go to check" links); on confirm, `submitExecution` → `PENDING_REVIEW`, navigates to `/my-tests`.
- **Review a submission** (`reviews.$executionId.tsx`): reviewer (`canReviewExecution`) inspects checks via filters, optionally flags checks for retest via `Checkbox`, writes a comment (≥15 chars required to reject/retest), calls `approveExecution` (→`APPROVED`, later transitions to `COMPLETED`), `rejectExecution` (→`REJECTED`, terminal), or `requestRetest(s, user, execution.id, comment, selectedChecks)` (→`RETEST_REQUIRED`).
- **Retest cycle**: `RETEST_REQUIRED` + `canExecuteTest` → "Resume for retest" (`resumeForRetest`) → `RETEST_IN_PROGRESS`. Demonstrated in seed data (`exec-3`, check ACO-002: attempt-1 `failed`, attempt-2 `retest_required`, both rows permanently retained per the domain comment "a new attempt is a new row... rather than a mutated field"). `checkEditable` locks non-flagged checks during a retest round.
- **"AI-assisted" insight**: `similarFailures()`/`failureHotspots()` are deterministic `services.ts` functions (not a real AI/ML call), rendered under a "Quality Intelligence"/"AI-assisted insight" panel with explicit copy "AI-assisted recommendation — Quality validation required."

### Explicitly confirmed absent

- No Supabase/BaaS client anywhere (`supabase|createClient|SUPABASE` grep under `src/` = zero matches; no `src/integrations` directory).
- No server-side auth middleware beyond generic CSRF/error handling in `src/start.ts` — login/OTP/session logic is entirely client-side/in-memory.
- No GraphQL client or schema in any file read.
- No websocket usage in any file read.
- No database or migration files; `seed.ts` is the sole data source, mirrored to `localStorage`.
- No centralized/shared route-guard hook — `verify-location.tsx` and `verify-station.tsx` each duplicate their own redirect logic.

### Files inspected

`package.json`, `vite.config.ts`, `components.json`, `src/router.tsx`, `src/start.ts`, `src/server.ts`, `src/routes/__root.tsx`, `src/lib/tms/permissions.ts`, `src/lib/tms/store.tsx`, `src/lib/tms/services.ts`, `src/types/domain.ts`, `src/routes/index.tsx`, `src/routes/otp.tsx`, `src/routes/dashboard.tsx`, `src/components/tms/AppShell.tsx`, `src/routes/my-tests.tsx`, `src/lib/tms/seed.ts` (full), `src/routes/executions.$executionId.tsx` (full), `src/routes/verify-location.tsx` (full), `src/routes/verify-station.tsx` (full), `src/routeTree.gen.ts` (partial), `src/routes/reviews.index.tsx` (full), `src/routes/reviews.$executionId.tsx` (full), `src/routes/templates.index.tsx` (full), `src/routes/admin.tsx` (full), `src/components/tms/badges.tsx` (full), `src/components/tms/CommandPalette.tsx` (full).

### Files referenced/imported but NOT inspected

`src/routes/reports.tsx`, `src/routes/templates.$templateId.tsx`, `src/routes/templates.categories.tsx`, `src/routes/templates.import.tsx`, `src/routes/templates.test-cases.tsx`, `src/routes/units.$unitId.tsx`, `src/components/tms/Timeline.tsx`, `EmptyState.tsx`, `ReassignSheet.tsx`, `AccessSteps.tsx`, `Logo.tsx`, `AssignCheckerDialog.tsx`, `src/lib/error-capture.ts`, `src/lib/error-page.ts`, `src/lib/lovable-error-reporting.ts`, `src/lib/utils.ts`, `src/hooks/use-mobile.tsx`, root markdown docs (`README.md`, `BUSINESS_WORKFLOW.md`, `BUSINESS_WORKFLOW_UPDATE_REPORT.md`, `CURRENT_PRODUCT_AUDIT.md`, `PRODUCT_CONTEXT.md`, `AGENTS.md`), test files, `e2e/` directory.

# Codebase Orientation Map — Pibythree Quality Hub (`testflow-enterprise-main`)

## 1-Line Summary

A TanStack Start + React 19 single-page application implementing a digital manufacturing quality-inspection workflow (login → OTP → location/station verification → checklist execution → review/retest → reporting), with **all state held client-side in one `AppState` object persisted to browser `localStorage`** — there is no database, backend API, or network call anywhere in the app.

## 5-Minute Explanation

- **Primary tasks**: authenticate a factory worker, gate access through location/station verification, execute a versioned QA checklist against a manufacturing unit (pass/fail/N/A/measurement checks with evidence), route submissions to a Quality Checker for approve/reject/retest, let Template Managers author/version/publish checklists, let Admins manage users/plants/stations/devices/failure categories, and compute live analytics/reports — all client-side.
- **Primary inputs**: form input (employee ID/password, OTP digits), button clicks, file uploads (evidence images via `FileReader`), CSV text (`File.text()`) for bulk checklist import, and one genuine browser API call — `navigator.geolocation.getCurrentPosition()`.
- **Primary outputs**: React-rendered UI, `localStorage` writes (key `pibythree-quality-hub-v1`), client-generated CSV downloads (`Blob`/`URL.createObjectURL`), toast notifications (`sonner`) — nothing leaves the browser.
- **Key files**: `src/lib/tms/services.ts` (business logic), `src/lib/tms/store.tsx` (state + persistence), `src/types/domain.ts` (types + state machine), `src/lib/tms/permissions.ts` (RBAC), `src/routes/*.tsx` (one file per screen), `src/components/tms/AppShell.tsx` (shared layout + route guard).
- **Main code path**: route file → `useTms()` reads `state` → user action calls `run((s) => someServiceFn(s, ...))` → `services.ts` validates and returns `Result<T>` → `store.tsx` writes new state to React context + `localStorage` → UI re-renders.

---

## 1. Important Folder Tree

```
testflow-enterprise-main/
├── src/
│   ├── routes/                        # File-based routing — 1 file per screen (24 files)
│   │   ├── __root.tsx                 # App shell: providers, error/404 boundaries
│   │   ├── index.tsx                  # Login
│   │   ├── otp.tsx                    # OTP verification
│   │   ├── verify-location.tsx        # Plant/location gate
│   │   ├── verify-station.tsx         # Station gate
│   │   ├── dashboard.tsx              # Role-aware landing dashboard
│   │   ├── my-tests.tsx               # Tester's assigned work queue
│   │   ├── units.$unitId.tsx          # Unit detail / execution list
│   │   ├── executions.$executionId.tsx # Digital Quality Worksheet (core screen)
│   │   ├── reviews.index.tsx          # Quality Checker review queue
│   │   ├── reviews.$executionId.tsx   # Review/approve/reject/retest detail
│   │   ├── reports.tsx                # Analytics + CSV export
│   │   ├── templates.index.tsx        # Checklist template list
│   │   ├── templates.$templateId.tsx  # Template authoring/versioning
│   │   ├── templates.categories.tsx   # Category management
│   │   ├── templates.test-cases.tsx   # Test-case library
│   │   ├── templates.import.tsx       # CSV import flow
│   │   └── admin.tsx                  # Users/plants/stations/devices/units CRUD
│   ├── components/
│   │   ├── tms/                       # Domain-specific shared components
│   │   │   ├── AppShell.tsx           # Nav + universal access-gate enforcement
│   │   │   ├── AccessSteps.tsx
│   │   │   ├── AssignCheckerDialog.tsx
│   │   │   ├── CommandPalette.tsx
│   │   │   ├── EmptyState.tsx
│   │   │   ├── Logo.tsx
│   │   │   ├── ReassignSheet.tsx
│   │   │   ├── Timeline.tsx
│   │   │   └── badges.tsx
│   │   └── ui/                        # ~45 shadcn/Radix generic primitives (button, dialog, table…)
│   ├── lib/
│   │   ├── tms/                       # The application's "backend" — all client-side
│   │   │   ├── services.ts            # All business logic/mutations (Result<T> pattern)
│   │   │   ├── store.tsx              # React Context + localStorage persistence
│   │   │   ├── permissions.ts         # RBAC predicate functions
│   │   │   ├── seed.ts                # Hardcoded demo dataset (users, plants, 1 template, units)
│   │   │   ├── services.test.ts
│   │   │   └── permissions.test.ts
│   │   ├── utils.ts                   # cn() classname helper
│   │   ├── error-capture.ts
│   │   ├── error-page.ts
│   │   └── lovable-error-reporting.ts # Lovable.dev error webhook integration
│   ├── types/
│   │   ├── domain.ts                  # Canonical types, ExecutionStatus, EXECUTION_TRANSITIONS
│   │   └── domain.test.ts
│   ├── hooks/
│   │   └── use-mobile.tsx
│   ├── test/
│   │   └── setup.ts                   # Vitest setup
│   ├── router.tsx                     # QueryClient + TanStack Router factory
│   ├── routeTree.gen.ts               # Auto-generated route tree (build artifact, but architectural)
│   ├── server.ts                      # Nitro/Cloudflare-style SSR fetch() handler
│   ├── start.ts                       # createStart() + CSRF/error middleware registration
│   └── styles.css
├── e2e/                                # Playwright end-to-end specs
│   ├── critical-workflow.spec.ts      # THE golden-path spec (tester→checker→retest→approve)
│   ├── supervisor-association.spec.ts
│   ├── supervisor-dashboard-viewport.spec.ts
│   ├── tester-status-tabs.spec.ts
│   ├── tester-status-tabs-screenshots.spec.ts
│   ├── worksheet-viewport.spec.ts
│   └── fixtures/
├── scripts/
│   ├── build-gh-pages.mjs             # GitHub Pages static build helper
│   └── gh-pages-sim-server.mjs        # Local static-hosting simulator
├── public/                            # favicon, logo, robots.txt
├── package.json / bun.lock            # Dependency manifest
├── vite.config.ts / vitest.config.ts / playwright.config.ts / tsconfig.json
├── BUSINESS_WORKFLOW.md
├── PRODUCT_CONTEXT.md
└── CURRENT_PRODUCT_AUDIT.md           # Self-identifies as a superseded historical snapshot
```

_(Excluded as generated/dependency: `node_modules`, `.output`, `.tanstack/tmp`, `test-results`.)_

---

## 2. Per-Folder Purpose and Connections

|Folder|Purpose|Connects to|
|---|---|---|
|`src/routes/`|One screen per file; reads state via `useTms()`, calls mutations via `run()`|`src/lib/tms/*`, `src/components/tms/*`, `src/components/ui/*`|
|`src/lib/tms/`|The entire domain layer — logic (`services.ts`), state+persistence (`store.tsx`), authorization (`permissions.ts`), demo data (`seed.ts`)|Consumed by every route; `services.ts` itself calls into `permissions.ts` to reject unauthorized mutations at the source (not just hiding UI)|
|`src/types/domain.ts`|Canonical types and the `ExecutionStatus` state machine (`EXECUTION_TRANSITIONS`, `canTransition()`)|Imported by `services.ts`, `store.tsx`, every route|
|`src/components/tms/`|Shared domain UI — `AppShell.tsx` is the universal wrapper enforcing the login/location/station access gate for every protected route|Imported by `__root.tsx` and route files|
|`src/components/ui/`|Generic shadcn/Radix primitives, no domain logic|Imported throughout `routes/` and `components/tms/`|
|`src/lib/` (root)|Cross-cutting infra: classname helper, SSR error capture/rendering, Lovable.dev error reporting webhook|Used by `__root.tsx`, `start.ts`, `server.ts`|
|`src/server.ts` / `src/start.ts` / `src/router.tsx`|Runtime boot: SSR fetch handler, middleware registration (CSRF/error), router+QueryClient factory|Framework glue only — no business logic|
|`e2e/`|Playwright specs describing the real, expected end-to-end workflows|Drives against the running app; strongest evidence of "critical" business flows|
|`scripts/`|Static-hosting build/deploy helpers for GitHub Pages|Independent of app runtime|

---

## 3. Important-Files Table (Priority)

|Priority|File|Why|
|---|---|---|
|P0|`src/types/domain.ts`|Canonical types + execution state machine; everything else derives from this|
|P0|`src/lib/tms/services.ts`|All business logic — the closest thing to a "backend"|
|P0|`src/lib/tms/store.tsx`|State container + localStorage persistence — the closest thing to a "database"|
|P0|`src/lib/tms/permissions.ts`|RBAC gates used by both UI and services|
|P0|`src/components/tms/AppShell.tsx`|Universal layout + actual enforcement point for access control|
|P0|`src/routes/index.tsx`, `otp.tsx`, `verify-location.tsx`, `verify-station.tsx`|The full login/access gate, in sequence|
|P0|`src/routes/executions.$executionId.tsx`|The Digital Quality Worksheet — largest, most central screen|
|P0|`src/routes/__root.tsx`|App shell, providers, error/404 boundaries|
|P1|`src/routes/templates.$templateId.tsx`|Full checklist authoring/versioning UI|
|P1|`src/routes/admin.tsx`|Largest CRUD surface (users/plants/stations/devices/units/assignments)|
|P1|`src/routes/reports.tsx`|Analytics/KPI computation and CSV export|
|P1|`src/routes/reviews.index.tsx`, `reviews.$executionId.tsx`|Quality Checker review/approve/reject/retest flow|
|P1|`src/routes/dashboard.tsx`, `my-tests.tsx`, `units.$unitId.tsx`|Core tester navigation screens|
|P1|`src/lib/tms/seed.ts`|Demo data — what a new dev actually sees when running the app|
|P1|`src/start.ts`, `src/server.ts`, `src/router.tsx`|Runtime boot entry points|
|P2|`src/routes/templates.index.tsx`, `templates.import.tsx`, `templates.categories.tsx`, `templates.test-cases.tsx`|Secondary template-management screens|
|P2|`src/components/tms/CommandPalette.tsx`, `Timeline.tsx`, `badges.tsx`|Shared UI (usage confirmed via imports)|
|P2|`src/components/ui/*`|Generic primitives, standard boilerplate|
|P2|`e2e/critical-workflow.spec.ts`|Best single artifact for the "golden path" end to end|

---

## 4. Backend / Database / API — Explicit Gap Analysis

**Finding: this is a frontend-only application. No real backend, database, or API layer exists.**

Evidence:

- `package.json`: no database client (no Prisma/Drizzle/Supabase/`pg`/Mongo driver), no HTTP client (`axios`), no auth SDK.
- Repo-wide search for `createServerFn|axios|fetch\(` across `src/`: **one match** — `src/server.ts` line 48, `async fetch(request, env, ctx)`, the inbound Nitro/Cloudflare SSR handler signature, not an outbound call. TanStack Start's `createServerFn()` is never used anywhere despite the framework supporting it.
- `src/start.ts` and `src/server.ts` (both fully read): only SSR error-normalization and CSRF middleware for server functions that don't exist in this codebase.
- Every route mutates state exclusively via `run((s) => someServiceFn(s, ...))` against an in-memory `AppState`.
- `src/lib/tms/store.tsx`: persistence = `localStorage.setItem("pibythree-quality-hub-v1", ...)`. That is the entire "database."
- Evidence uploads: `FileReader.readAsDataURL()` → base64 string stored inside the same `AppState`/localStorage blob (5 MB/file cap). No object storage.
- CSV import/export: both fully client-side (`File.text()` parsing; `Blob` + `<a download>` for export). No server endpoint.
- The one genuine external browser API: `navigator.geolocation.getCurrentPosition()`, used only as non-authoritative corroboration on the session object — never gates access.
- Confirmed by the project's own docs: `PRODUCT_CONTEXT.md` §4 states the prototype "operates entirely client-side without an external backend, database, or network API layer"; `CURRENT_PRODUCT_AUDIT.md` §2 independently reaches the same conclusion via the same grep method.

**Explicitly missing pieces:**

- No cloud/relational database or ORM — state lives in one browser's `localStorage` only.
- No backend API service — `services.ts` runs entirely client-side.
- No real object/file storage for evidence images.
- No real authentication provider — hardcoded shared demo password (`"pibythree@2026"`) and hardcoded OTP (`"123456"`) checked against a hardcoded seed user list.
- No real GPS/hardware validation — location/station are manual list selections; geolocation is advisory only.
- No cross-device/cross-user sync — state is single-browser.
- No real AI/LLM — the "AI-assisted insight" feature is deterministic keyword-overlap/frequency counting, disclosed in the UI copy itself as "AI-assisted recommendation — Quality validation required."

`PRODUCT_CONTEXT.md` §30 describes a _hypothetical future_ production architecture (microservices, PostgreSQL/TimescaleDB, S3, Kafka, OAuth2/SAML) — none of it exists in the current code; it is stated as a future direction, not a current claim.

---

## 5. User-Facing Business Domain and Workflow

**Domain**: Manufacturing quality inspection (branded "Pibythree Quality Hub," modeled on a Tata Electronics–style Hosur, India plant), digitizing paper/Excel QA checklists (FATP/EQT-style final assembly testing).

**Roles** (from `src/types/domain.ts` `Role` union, cross-checked against `permissions.ts` and `AppShell.tsx` nav visibility): `tester` (labeled "Quality Technician"), `quality_checker`, `manager` (labeled "Supervisor"), `template_manager`, `admin` — five roles only. **Discrepancy noted**: `PRODUCT_CONTEXT.md` and `CURRENT_PRODUCT_AUDIT.md` describe an additional sixth "Senior Manager" role (`TE-4001`, `canViewSeniorDashboard`) that **does not exist in the current code**. `CURRENT_PRODUCT_AUDIT.md` self-identifies as a superseded historical snapshot, which explains the drift; the newer `BUSINESS_WORKFLOW.md` lists five roles and matches the code.

**Actual routes** (verified by direct file reads): `/`, `/otp`, `/verify-location`, `/verify-station`, `/dashboard`, `/my-tests`, `/units/$unitId`, `/executions/$executionId`, `/reviews/`, `/reviews/$executionId`, `/reports`, `/templates/`, `/templates/$templateId`, `/templates/categories`, `/templates/test-cases`, `/templates/import`, `/admin`.

**Critical workflow** (per `e2e/critical-workflow.spec.ts`, describe block literally titled _"Pibythree Quality Hub — critical workflow"_): tester `TE-1001` logs in, executes 17 seeded checks on unit `USN-OJAS-000451` (execution `exec-1`), deliberately fails measurement check `ACO-002` (95 dB against a 60–90 dB range), submits → quality checker `TE-2001` reviews the AI-assisted insight and requests a scoped retest on `ACO-002` → tester retests only that check (others locked) and resubmits → checker approves → Supervisor `TE-3001` sees it as "Completed" in the review queue. Additional specs in the same file cover save/unsaved-changes guards, a Template Manager's (`TE-5001`) CSV-import → publish → revision → diff workflow, and a checker viewing never-attempted checks as neutral.

**Docs vs. code**: `BUSINESS_WORKFLOW.md`'s stated execution flow (assign → verify → execute → submit → review → retest/approve) and template flow (draft → CSV import → validate → publish → immutable revision) match the code exactly. `PRODUCT_CONTEXT.md` matches on every specific mechanism spot-checked (demo password/OTP values, autosave debounce, evidence size cap, execution state diagram, audit action list), with the one drift being the extra "Senior Manager" role noted above.

**Files inspected**: `src/types/domain.ts`, `src/lib/tms/services.ts`, `store.tsx`, `permissions.ts`, `seed.ts` (partial, through line ~1966/2876 — remainder is more static demo data, doesn't change conclusions), all ~24 `src/routes/*.tsx` files, `src/components/tms/AppShell.tsx`, `src/lib/utils.ts`, `src/start.ts`, `src/server.ts`, `src/router.tsx`, `e2e/critical-workflow.spec.ts`, `.lovable/project.json`, `BUSINESS_WORKFLOW.md`, `PRODUCT_CONTEXT.md`, `CURRENT_PRODUCT_AUDIT.md`. **Not inspected**: remainder of `seed.ts`, individual `src/components/ui/*.tsx` primitives, `src/hooks/use-mobile.tsx`, `src/lib/error-capture.ts`/`error-page.ts`/`lovable-error-reporting.ts`, test files (`*.test.ts`, `src/test/setup.ts`), five of six e2e specs, `public/*`, `scripts/*.mjs`, `AGENTS.md`, `README.md`, and all remaining config files.

**Investigating agent output handling****Planning E2E fixture discovery and testing**

Update Todos

- [ ] 
    
    Inspect remaining routes, support code, and E2E coverage without modifying files
    
- [ ] 
    
    Synthesize verified architecture documentation
    

**Planning detailed E2E inspection****Ensuring verified source inspection**

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\e2e\supervisor-association.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\e2e\supervisor-dashboard-viewport.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\e2e\tester-status-tabs.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\e2e\tester-status-tabs-screenshots.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\e2e\worksheet-viewport.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\components\tms\ReassignSheet.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\components\tms\AssignCheckerDialog.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\hooks\use-mobile.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-1001)

**Planning deeper inspection of e2e and styles****Planning targeted reads of components and styles****Scheduling offset file read and component listing**

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\lib\tms\seed.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 1911-2910)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\components\tms\AppShell.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\components\tms\badges.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-1001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\components\tms\Timeline.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-1001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\components\tms\AccessSteps.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-1001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\components\tms\EmptyState.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-1001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\styles.css](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-2001)

Read [c:\Users\Dipak.Mandlik\OneDrive - PibyThree Consulting Services Private Limited\testflow-enterprise-main\src\routeTree.gen.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/index.html?id=a90af91a-28ed-48a7-a378-4b4ec29b4417&parentId=1&origin=43a310ad-71bf-4fb3-bac4-b0f2f64f4340&swVersion=6&extensionId=Anthropic.claude-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView&session=78fd8b41-c34b-4bfc-ba04-119fcca0babb#) (lines 2-1001)

**Reviewing configuration files for verification****Verifying tech stack versions and output order****Confirming component output sequence and package data****Completing synthesis and updating status**

Update Todos

- [ ] 
    
    Inspect remaining routes, support code, and E2E coverage without modifying files
    
- [ ] 
    
    Synthesize verified architecture documentation
    

**Preparing final structured documentation****Compiling exact manifest versions and architecture overview****Noting local demo auth limitations****Analyzing approval-to-completion transition****Reviewing deployment setup and documentation scope****Planning verified codebase documentation structure****Outlining application architecture****Documenting authentication and permissions flow****Confirming documentation approach and scope**

# Pibythree Quality Hub — Verified Codebase Documentation

**Inspection scope:** source, configuration, build/deployment scripts, unit-test specifications, and Playwright specifications were read.  
**Not performed:** no code was modified, no packages installed, and no lint, typecheck, build, unit, or E2E command was executed. Test coverage below describes what the test source specifies—not observed passing results.

---

## 1. Verified technology stack

Versions below are the version declarations in [package.json](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/package.json); `^` means the manifest permits compatible newer versions.

|Area|Technology|
|---|---|
|Language|TypeScript `^5.8.3`|
|UI runtime|React `^19.2.0`, React DOM `^19.2.0`|
|Application framework|TanStack Start `1.168.32`|
|Router|TanStack Router `1.170.18`; router plugin `1.168.23`|
|Query provider|TanStack React Query `^5.101.1`|
|Build tool|Vite `^8.2.0`|
|Server/build runtime|Nitro `3.0.260603-beta`, configured as `node-server`|
|Styling|Tailwind CSS `^4.2.1`, `@tailwindcss/vite ^4.2.1`|
|UI primitives|Local shadcn/ui-style components built on Radix UI packages|
|Icons|`lucide-react`|
|Forms|React Hook Form `^7.71.2`, `@hookform/resolvers ^5.2.2`, Zod `^3.24.2`|
|Toast notifications|Sonner `^2.0.7`|
|Charts|Recharts `^2.15.4`|
|Date utilities|date-fns|
|Command palette|cmdk|
|Testing|Vitest `^4.1.10`, JSDOM `^30.0.1`, Testing Library|
|Browser E2E testing|Playwright `^1.62.1`|
|Linting|ESLint `^9.32.0`|
|Formatting|Prettier|
|Package manager used by CI|Bun|
|Hosting target|GitHub Pages|

### Code and UI conventions

- TypeScript is strict: `strict`, `noImplicitReturns`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, and related checks are enabled in [tsconfig.json](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/tsconfig.json).
- JSX uses the automatic React transform: `jsx: "react-jsx"`.
- The `@/*` alias resolves to [src/](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/).
- [components.json](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/components.json) defines shadcn configuration using the `new-york` style, CSS variables, Lucide icons, and [src/styles.css](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/styles.css).
- The UI loads IBM Plex Sans and IBM Plex Mono from Google Fonts in [src/routes/__root.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/__root.tsx).
- The application uses a fixed-height workspace: the document is hidden from scrolling, while the authenticated content pane scrolls internally. This is verified by [e2e/worksheet-viewport.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/worksheet-viewport.spec.ts).

### Package-management evidence

Both [bun.lock](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/bun.lock) and [package-lock.json](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/package-lock.json) exist. GitHub Actions installs with Bun and `--frozen-lockfile`, so **Bun is CI-authoritative**.

[bunfig.toml](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/bunfig.toml) sets a 24-hour minimum package release age, with explicit exclusions for selected Lovable packages.

---

## 2. Actual end-to-end architecture

```text
Browser
  │
  ├─ TanStack Router / Root Route
  │    └─ QueryClientProvider + TmsProvider + Sonner Toaster
  │
  ├─ Authentication UI
  │    Login → OTP → local Session
  │
  ├─ Tester access gate
  │    Plant + location selection → station selection
  │    Optional browser geolocation recorded as corroborating evidence
  │
  ├─ Routes and shared UI
  │    Dashboard / My Tests / Worksheet / Review / Reports /
  │    Templates / Administration
  │
  ├─ Domain layer
  │    services.ts + permissions.ts + domain.ts
  │
  ├─ State layer
  │    One in-memory AppState in React Context
  │
  ├─ Persistence
  │    browser localStorage: "pibythree-quality-hub-v1"
  │
  └─ Outputs
       React UI, local audit trail, notifications, evidence data URLs,
       Recharts dashboards, and browser-generated CSV downloads

Build/deployment path
  Vite + TanStack Start/Nitro
    → .output node-server build
    → GitHub Pages conversion script
    → dist/
    → GitHub Pages
```

### Runtime entry points

|Layer|Actual responsibility|
|---|---|
|[src/router.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/router.tsx)|Creates a TanStack Router and a `QueryClient`; enables scroll restoration.|
|[src/routes/__root.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/__root.tsx)|Provides the document shell, global CSS, Query Client, `TmsProvider`, toast renderer, root 404 page, and root error boundary.|
|[src/start.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/start.ts)|Registers TanStack Start request middleware, including CSRF middleware limited to Start server-function requests and error fallback behavior.|
|[src/server.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/server.ts)|Wraps TanStack Start server handling and substitutes local error HTML for catastrophic SSR/H3 error responses.|
|[src/components/tms/AppShell.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/AppShell.tsx)|Provides authenticated navigation, mobile navigation, notifications, command palette, user menu, demo reset, and the tester location/station gate.|
|[src/lib/tms/store.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/store.tsx)|Owns the browser state repository, localStorage hydration/persistence, state reset, and service-result handling.|
|[src/lib/tms/services.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/services.ts)|Contains the business operations and selectors for authentication, execution, review, reporting, templates, administration, and audit records.|
|[src/lib/tms/permissions.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/permissions.ts)|Centralizes role checks and execution/review eligibility.|
|[src/types/domain.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/types/domain.ts)|Defines the canonical entities, status labels, state machine, and resolved-status rules.|

### Important architectural clarification

TanStack Start, Nitro, SSR handling, and CSRF middleware are present. However, the inspected application workflow does **not** use server functions, loaders, a remote API, database queries, or server persistence.

**Not present in the current codebase:**

- A remote business API.
- A database or ORM.
- A server-side repository for users, templates, assignments, execution results, evidence, or reports.
- External object storage for uploaded evidence.
- A client integration for Supabase, Firebase, Auth0, Clerk, Stripe, analytics, or a third-party AI provider.
- WebSocket or GraphQL application integration.

The operational product is a **local-first browser application**. State is isolated to the browser storage where it was created.

---

## 3. Data model and data movement

### Canonical state

[src/types/domain.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/types/domain.ts) defines one `AppState` holding:

```text
users, plants, locations, stations, devices,
templates, templateCategories, templateChecks,
units, assignments, executions, checkResults,
evidence, reviews, notifications, audit,
failureCategories, session, pendingLoginUserId
```

The state flow is:

```text
Route/component action
  → run(service operation)
  → permission and business-rule validation
  → immutable next AppState or error result
  → React Context update
  → localStorage persistence
  → Sonner success/error toast
  → UI rerender from updated state
```

### Persistence behavior

[src/lib/tms/store.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/store.tsx) uses:

```text
pibythree-quality-hub-v1
```

as its browser localStorage key.

- On first use, state comes from [src/lib/tms/seed.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/seed.ts).
- On subsequent use, the provider hydrates from localStorage.
- Evidence is stored as a data URL inside the same application state.
- localStorage write failures are caught; the current in-memory session continues, but the updated data may not survive a reload.
- “Reset Demo Data” restores the deterministic seed state through [src/components/tms/AppShell.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/AppShell.tsx).

### Seeded demo data

The seed supplies:

- 9 users across all five roles.
- One plant: Hosur.
- Two locations.
- Four stations, including one maintenance station.
- Four device records.
- One published OJAS-EQT functional-test template, revision 3.
- Nine units, assignments, and executions covering all major execution statuses.
- Historical check-result attempts, evidence, reviews, notifications, and audit events.

The primary seeded checklist covers categories including Check In, Acoustics, Battery and Charging, Camera, WiFi, Front/Rear Optical Sensing, Touch, Display, SWDL, and Check Out.

---

## 4. Authentication, session, and access gate

### Local demo authentication

Authentication is deterministic and implemented in [src/lib/tms/services.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/services.ts):

```text
Password: pibythree@2026
OTP:      123456
```

Flow:

```text
Employee ID + fixed password
  → active seeded user lookup
  → pendingLoginUserId
  → fixed six-digit OTP validation
  → session creation
  → tester location/station gate or dashboard
```

**Not present in the current codebase:**

- Real password hashing or credential verification.
- OTP delivery by email, SMS, authenticator app, or provider.
- OAuth, SSO, SAML, identity-provider integration, JWT refresh, or cookie-backed sessions.
- Server-side session validation.

### Tester location and station verification

For testers, [src/routes/verify-location.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/verify-location.tsx) and [src/routes/verify-station.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/verify-station.tsx) enforce:

```text
Login
  → choose plant
  → choose location within that plant
  → choose active station within that location
  → access worksheet/dashboard
```

- The user-selected plant and location are the authoritative access record.
- Browser geolocation is requested with `navigator.geolocation.getCurrentPosition`.
- Geolocation coordinates, accuracy, and capture time become supporting audit metadata only.
- Denied or unavailable browser geolocation does not block manual verification.
- Station verification requires an already verified location.
- Only active stations are selectable; maintenance/inactive stations are unavailable.
- A station’s device is displayed but device online/offline state does not replace the station gate.
- A new login clears prior tester location/station verification state.

The shared route shell redirects a tester who attempts a direct protected-route navigation back to the correct verification step. See [src/components/tms/AppShell.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/AppShell.tsx).

---

## 5. Roles and authorization

The five declared roles are in [src/types/domain.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/types/domain.ts):

|Role value|Display label|Main responsibilities|
|---|---|---|
|`tester`|Quality Technician|Perform assigned work, submit for review, perform scoped retests, view personal reports.|
|`quality_checker`|Quality Checker|Review eligible other-user submissions, approve, reject, or request retests; operational reports.|
|`manager`|Supervisor|Operational dashboard/reporting, assignments, reassignment, technician/checker associations, review authority.|
|`template_manager`|Template Manager|Create, edit, import, validate, publish, archive, clone, and compare checklists.|
|`admin`|Administrator|All manager/template-manager abilities plus user, plant, station, and device administration.|

### Verified policy rules

- Only active assigned testers can execute their own assigned or retest-in-progress executions.
- A tester cannot review an execution.
- A reviewer cannot review their own execution.
- Quality checkers, managers, and admins may review eligible pending-review work.
- Managers and admins manage assignments and reassignments.
- Template managers and admins manage templates and failure categories.
- Admins manage users, plants, stations, and devices.
- Testers have personal reporting; report-authorized non-testers have operational reporting.
- CSV export is restricted to report-authorized users.
- UI visibility is role-conditioned, but service operations also enforce permission checks.

### Quality-checker routing

Each technician can be associated with an active quality checker. Assignment creation can use:

1. An explicit selected checker.
2. The assigned tester’s default association.
3. No checker, representing a checker pool.

The effective checker is snapshotted into the `Execution`. Later association changes do not rewrite historical routing. The UI and E2E coverage for that behavior are in [src/components/tms/AssignCheckerDialog.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/AssignCheckerDialog.tsx), [src/components/tms/ReassignSheet.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/ReassignSheet.tsx), and [e2e/supervisor-association.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/supervisor-association.spec.ts).

---

## 6. Quality-execution workflow and business rules

### Execution state machine

[src/types/domain.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/types/domain.ts) defines:

```text
ASSIGNED
  → IN_PROGRESS
  → PENDING_REVIEW
  → APPROVED
  → COMPLETED

PENDING_REVIEW
  → REJECTED

PENDING_REVIEW
  → RETEST_REQUIRED
  → RETEST_IN_PROGRESS
  → PENDING_REVIEW
```

`REJECTED` and `COMPLETED` are terminal states.

### Actual tester workflow

```text
Assigned work
  → verify location/station
  → start execution
  → record each check outcome
  → attach supporting evidence where required
  → submit when mandatory checks are resolved
  → wait for Quality review
  → if requested, retest flagged checks only
  → resubmit
```

The main worksheet is [src/routes/executions.$executionId.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/executions.$executionId.tsx).

### Check-result behavior

A check supports:

```text
binary      Pass / Fail
ternary     Pass / Fail / N/A
measurement Numeric value evaluated against configured limits
text        Text observation
visual      Visual inspection
```

Rules verified in services, UI behavior, and test specifications:

- A mandatory check must reach a resolved state before submission.
- Resolved states are `passed`, `failed`, `na`, `retest_passed`, and `retest_failed`.
- A non-measurement result generally requires an actual-result value unless marked N/A.
- N/A is accepted only where the template check permits it.
- A failed result requires a failure category and failure description.
- Failure descriptions are validated.
- Evidence-required checks require evidence.
- A failed check can require evidence depending on the check configuration.
- The evidence size ceiling is 5 MiB.
- Allowed evidence MIME families include PNG, JPEG, WebP, SVG, PDF, and plain text.
- Measurements are automatically compared against configured minimum/maximum values.
- A manually selected pass cannot override an out-of-range measurement; it is recorded as failed.
- Save is explicit: the worksheet tracks unsaved changes and does not persist a result until **Save result** is used.
- Page navigation and browser unload guards protect unsaved worksheet changes.

### Retest history preservation

Each result is keyed by:

```text
executionId + templateCheckId + attempt
```

A retest appends a new `CheckResult`; it does not mutate the original failed result. The current check result is the row with the highest attempt number.

During a retest:

- Only reviewer-flagged checks are editable.
- Earlier resolved checks are locked.
- Resubmission increments the execution round.
- The review history, evidence, prior attempts, notifications, and audit events remain available.

### Review workflow

The review queue is in [src/routes/reviews.index.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/reviews.index.tsx), and the decision page is [src/routes/reviews.$executionId.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/reviews.$executionId.tsx).

A reviewer can:

|Decision|Result|
|---|---|
|Approve|Moves the execution through approval/completion processing.|
|Reject|Moves the execution to terminal `REJECTED`.|
|Request retest|Requires a comment of at least 15 characters and one or more selected checks; creates a scoped retest request.|

The review page supports all, failure, skipped, retest, and evidence filters.

“Skipped” is a presentation label for a missing/unresolved recorded result. It is **not** an extra persisted `CheckStatus`.

### Notifications and audit trail

Meaningful operations generate local notifications and audit records, including:

- Authentication, location, and station verification.
- Assignment creation/reassignment.
- Execution start, submission, resubmission, and completion.
- Check-result changes.
- Evidence upload/removal.
- Review approval, rejection, and retest requests.
- User, plant, station, device, unit, failure-category, and template management events.

[src/components/tms/Timeline.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/Timeline.tsx) maps these audit events into readable timeline entries.

---

## 7. Templates and checklist lifecycle

### Template status values

```text
draft
under_review
approved
published
archived
```

Template work is primarily implemented in:

- [src/routes/templates.index.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/templates.index.tsx)
- [src/routes/templates.$templateId.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/templates.$templateId.tsx)
- [src/routes/templates.categories.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/templates.categories.tsx)
- [src/routes/templates.test-cases.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/templates.test-cases.tsx)
- [src/routes/templates.import.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/templates.import.tsx)

### Lifecycle rules

- Draft, under-review, and approved templates are editable/importable.
- Published templates are immutable.
- Changing a published checklist requires creating a cloned revision.
- Checklists are grouped by `familyCode` and revision.
- Template validation runs before publishing.
- Publishing is unavailable while validation problems remain.
- The application supports comparison with the preceding revision.
- Template operations add audit events.

### Checklist authoring

The editor supports:

- Categories: add, rename, remove.
- Checks: add, edit, duplicate, remove, reorder.
- Check types: binary, ternary, measurement, text, visual.
- Mandatory, N/A eligibility, evidence-required configuration.
- Measurement unit/minimum/maximum.
- Default failure category.
- Technician-oriented preview.
- Revision-diff display.
- Archive, clone, publish, and submit-for-review actions.

### CSV imports

[src/routes/templates.import.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/templates.import.tsx) reads a user-selected CSV in the browser using `File.text()`.

Required columns:

```text
category
checkCode
title
```

The UI previews imported check/category counts before committing the import. Import parsing, duplicate protection, category creation, validation, and audit events are covered in [src/lib/tms/services.test.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/services.test.ts).

---

## 8. Dashboards, reporting, search, and outputs

### Role-aware dashboards

[src/routes/dashboard.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/dashboard.tsx) selects a dashboard based on the authenticated role:

|Perspective|Verified content|
|---|---|
|Tester|Personal assignment queue, retest/in-progress work, associated quality checker, personal activity.|
|Quality Checker|Pending review work, retest activity, completed reviews, technician/checker association and workload.|
|Manager/Admin|Execution/quality metrics, workload analysis, assignment/reassignment access, category quality, audit activity.|
|Template Manager|Draft/review/published checklist status and template metrics.|

### Reports

[src/routes/reports.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/reports.tsx) provides two modes:

|Audience|Report scope|
|---|---|
|Tester|Personal first-pass yield, failure rate, retest rate, resolved checks, pass/fail trend, personal failure categories.|
|Non-tester report users|Operational execution distribution, plant-wide trends, failure hotspots, failed-result drill-down, and per-station quality metrics.|

Reports derive from the current browser state, especially current check-result attempts.

CSV output is generated entirely in the browser using `Blob`, `URL.createObjectURL`, and an anchor click.

**Not present in the current codebase:** a reporting warehouse, scheduled exports, report API, email delivery, or external BI integration.

### Local “AI-assisted” insight

The source labels this feature “Quality Intelligence” / “AI-assisted insight,” but the implementation is deterministic:

- `failureHotspots` groups failed outcomes by failure category.
- `similarFailures` performs keyword-overlap matching against historical failure descriptions.
- The UI explicitly requires Quality validation of the recommendation.

**Not present in the current codebase:**

- An LLM.
- A hosted AI model.
- An external AI API.
- Autonomous approval, rejection, or quality-result override.

### Command palette

[src/components/tms/CommandPalette.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/CommandPalette.tsx) supports `Ctrl/Cmd + K` search across:

- Units.
- Executions visible to the current role.
- Template checks.
- Static navigation destinations.

It searches current in-memory application state; it does not query a remote search service.

---

## 9. Folder map

```text
project-root/
├── .github/
│   └── workflows/
│       └── deploy.yml                 CI and GitHub Pages deployment
├── e2e/
│   ├── fixtures/                      E2E test input data
│   ├── screenshots/                   Screenshot-test outputs/baselines
│   └── *.spec.ts                      Playwright workflow and viewport tests
├── public/                            Static favicon, brand image, robots.txt
├── scripts/
│   ├── build-gh-pages.mjs             Static GitHub Pages conversion
│   └── gh-pages-sim-server.mjs        Local Pages-behavior simulator
├── src/
│   ├── components/
│   │   ├── tms/                       Domain-specific UI and application shell
│   │   └── ui/                        Reusable shadcn/Radix primitives
│   ├── hooks/                         Small UI hooks
│   ├── lib/
│   │   ├── tms/                       State, permissions, services, seed data, tests
│   │   ├── error-capture.ts           SSR/runtime error preservation
│   │   ├── error-page.ts              Server-rendered fallback HTML
│   │   └── lovable-error-reporting.ts Editor/runtime telemetry bridge
│   ├── routes/                        File-based TanStack Router screens
│   ├── test/                          Vitest setup
│   ├── types/                         Domain contract and state machine
│   ├── routeTree.gen.ts               Generated router tree; do not edit
│   ├── router.tsx                     Router factory
│   ├── server.ts                      Start/Nitro server entry
│   ├── start.ts                       Start middleware setup
│   └── styles.css                     Tailwind theme and application styling
├── package.json                       Scripts and dependency manifest
├── vite.config.ts                     Vite, Start, Nitro, base-path configuration
├── vitest.config.ts                   Unit/component-test configuration
├── playwright.config.ts               Browser E2E configuration
├── tsconfig.json                      TypeScript rules
├── bunfig.toml                        Bun installation policy
└── components.json                    shadcn configuration
```

Generated/dependency directories such as `node_modules`, `.output`, and `.tanstack` are not application source.

---

## 10. Important files

|Priority|File|Purpose|
|---|---|---|
|Critical|[src/types/domain.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/types/domain.ts)|Canonical entities, labels, execution transitions, and result semantics.|
|Critical|[src/lib/tms/services.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/services.ts)|Primary business-rule and state-transition implementation.|
|Critical|[src/lib/tms/store.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/store.tsx)|React state provider, localStorage persistence, reset, and result runner.|
|Critical|[src/lib/tms/permissions.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/permissions.ts)|Central RBAC policy.|
|Critical|[src/routes/executions.$executionId.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/executions.$executionId.tsx)|Core tester worksheet, evidence, explicit save, submit, retest lock, and dirty-state behavior.|
|Critical|[src/routes/reviews.$executionId.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/reviews.$executionId.tsx)|Reviewer decisions, evidence/history inspection, and scoped retest selection.|
|Critical|[src/components/tms/AppShell.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/AppShell.tsx)|Authenticated layout, navigation, notifications, reset, and tester access gate.|
|Critical|[src/routes/__root.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/__root.tsx)|Global providers, HTML shell, root 404/error UI, and toast mounting.|
|Critical|[scripts/build-gh-pages.mjs](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/scripts/build-gh-pages.mjs)|Produces a static GitHub Pages deployment and deep-link fallback.|
|Critical|[.github/workflows/deploy.yml](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/.github/workflows/deploy.yml)|CI validation and GitHub Pages deployment pipeline.|
|High|[src/lib/tms/seed.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/seed.ts)|Entire deterministic local demo dataset.|
|High|[src/routes/admin.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/admin.tsx)|User, facility, device, unit, assignment, category, and audit administration.|
|High|[src/routes/templates.$templateId.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/templates.$templateId.tsx)|Checklist authoring, lifecycle, validation, revision, and diff workflows.|
|High|[src/routes/reports.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/reports.tsx)|Personal/operational metrics, charts, drill-down, and browser CSV export.|
|High|[src/routes/index.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/index.tsx) and [src/routes/otp.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/otp.tsx)|Local demo sign-in and OTP flow.|
|High|[src/routes/verify-location.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/verify-location.tsx) and [src/routes/verify-station.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/routes/verify-station.tsx)|Tester shop-floor verification gates.|
|High|[src/start.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/start.ts) and [src/server.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/server.ts)|TanStack Start middleware and SSR error protection.|
|High|[e2e/critical-workflow.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/critical-workflow.spec.ts)|Browser-level specification of the primary tester → review → retest → approval workflow.|
|Medium|[src/components/tms/CommandPalette.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/CommandPalette.tsx)|In-memory global search and keyboard navigation.|
|Medium|[src/components/tms/Timeline.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/Timeline.tsx)|Human-readable audit timeline renderer.|
|Medium|[src/components/tms/badges.tsx](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/components/tms/badges.tsx)|Status and priority display mappings.|
|Medium|[scripts/gh-pages-sim-server.mjs](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/scripts/gh-pages-sim-server.mjs)|Local simulation of project-scoped GitHub Pages fallback behavior.|
|Medium|[src/styles.css](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/styles.css)|Tailwind source setup, design tokens, and fixed-scroll workspace styling.|

---

## 11. Test coverage found

### Unit and service specifications

|File|Verified intent|
|---|---|
|[src/types/domain.test.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/types/domain.test.ts)|State transitions, retest loops, terminal states, and role-specific status labels.|
|[src/lib/tms/permissions.test.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/permissions.test.ts)|Execution/review eligibility, self-review denial, role restrictions, checker pool/routing, reports, assignments, and admin authority.|
|[src/lib/tms/services.test.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/lib/tms/services.test.ts)|Local auth, location/station verification, results, measurements, evidence, submission gates, retest history, notifications, audits, template lifecycle/import/revisions, administration, assignments, and associations.|

### Playwright specifications

|File|Verified intent|
|---|---|
|[e2e/critical-workflow.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/critical-workflow.spec.ts)|Full tester execution, failure evidence, scoped retest, approval, template import/publish/revision/diff, explicit save, unsaved-change protection, and skipped-review presentation.|
|[e2e/supervisor-association.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/supervisor-association.spec.ts)|Technician/checker associations and execution-level checker routing.|
|[e2e/tester-status-tabs.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/tester-status-tabs.spec.ts)|Worksheet status filters, explicit save, counts, failed measurements, search/filter composition, and non-tester visibility.|
|[e2e/tester-status-tabs-screenshots.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/tester-status-tabs-screenshots.spec.ts)|Visual states for workspace tabs and reviewer filters.|
|[e2e/worksheet-viewport.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/worksheet-viewport.spec.ts)|Internal workspace scrolling rather than document scrolling.|
|[e2e/supervisor-dashboard-viewport.spec.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/e2e/supervisor-dashboard-viewport.spec.ts)|Dashboard viewport behavior at desktop and tablet sizes.|

Playwright is configured for Chromium, one worker, serial execution, geolocation permission/mock coordinates, and retained traces on failure in [playwright.config.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/playwright.config.ts).

---

## 12. Build, CI/CD, GitHub Pages, and deep links

### Local scripts

[package.json](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/package.json) defines:

```text
dev             vite dev
build           vite build
build:dev       vite build --mode development
build:gh-pages  vite build && node scripts/build-gh-pages.mjs
preview         vite preview
lint            eslint .
typecheck       tsc
test            vitest run
e2e             playwright test
format          prettier --write .
```

### Vite/Start configuration

[vite.config.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/vite.config.ts) configures:

```text
base = GH_PAGES_BASE || "/"
TanStack Start server entry = "server"
Nitro preset = "node-server"
```

The configured Start entry resolves to [src/server.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/src/server.ts).

### GitHub Actions workflow

[.github/workflows/deploy.yml](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/.github/workflows/deploy.yml) triggers for:

- Pushes to `main`.
- Pull requests targeting `main`.
- Manual dispatch.

The build-and-test job performs:

```text
checkout
→ setup Bun
→ bun install --frozen-lockfile
→ bun run lint
→ bun run typecheck
→ bun run test
→ install Playwright Chromium
→ bun run e2e
→ build GitHub Pages artifact
```

For a main-branch push, it then:

```text
GH_PAGES_BASE=/<repository-name>/
→ bun run build:gh-pages
→ upload dist/ as Pages artifact
→ deploy through actions/deploy-pages
```

The Playwright report is uploaded even when the test job fails, with seven-day retention.

### Static GitHub Pages conversion

[scripts/build-gh-pages.mjs](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/scripts/build-gh-pages.mjs):

1. Requires the Nitro node-server entry at `.output/server/index.mjs`.
2. Recreates `dist/`.
3. Starts the generated server temporarily on port `8790`.
4. Fetches the configured root and saves rendered HTML as `dist/index.html`.
5. Injects a route-restoration script before application boot.
6. Writes `dist/404.html`.
7. Copies `.output/public` to `dist/`.
8. Writes `dist/.nojekyll`.
9. Stops the temporary server.

### Deep-link behavior

GitHub Pages cannot natively serve arbitrary SPA routes. The deployment uses this pattern:

```text
Direct visit to /<repo>/reviews/exec-4
  → GitHub Pages serves dist/404.html
  → 404 script redirects to the configured project base
     with the requested route encoded in the query string
  → dist/index.html restoration script calls history.replaceState
  → TanStack Router starts on the original route
```

[scripts/gh-pages-sim-server.mjs](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/scripts/gh-pages-sim-server.mjs) locally simulates that project-site behavior at `/testflow-enterprise`.

### Environment requirements

Verified application/environment variables:

|Variable|Used by|Purpose|
|---|---|---|
|`GH_PAGES_BASE`|[vite.config.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/vite.config.ts), [scripts/build-gh-pages.mjs](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/scripts/build-gh-pages.mjs), GitHub Actions|Sets the project-relative GitHub Pages base path.|
|`CI`|[playwright.config.ts](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/playwright.config.ts)|Controls Playwright dev-server reuse.|
|`PORT`, `NODE_ENV`|[scripts/build-gh-pages.mjs](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/scripts/build-gh-pages.mjs)|Set for its temporary generated-server process.|

**Not present in the current codebase:**

- `.env`, `.env.example`, or another environment-file template.
- A configured database URL.
- Auth-provider configuration.
- API base URL configuration.
- External service secrets.
- A custom-domain `CNAME` file.
- A `homepage` field in [package.json](vscode-webview://0flfb88bsv8v1bes5fd9al8tne8elt8dinn1fvssadnhdjh05ovh/package.json).

---

## 13. Final implementation position

Pibythree Quality Hub is a **browser-local, seeded demonstration application** for a digital manufacturing quality-inspection workflow. It has a detailed domain model, role policy, state machine, template versioning, evidence capture, audit events, review/retest history, browser-generated reports, and GitHub Pages deployment support.

It does **not** currently have a production backend, remotely shared persistence, real identity verification, actual OTP delivery, external evidence storage, or AI-model integration.