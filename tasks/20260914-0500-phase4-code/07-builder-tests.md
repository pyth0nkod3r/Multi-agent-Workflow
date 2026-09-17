# Task Spec — Final integration tests (run 20260914-0500-phase4-code)

## Mission
Write and run the final integration test suite that proves Phase 4 is complete: token migration is clean (Clear Deck values present, old palette absent), the nav contract matches 05-ia-menu-spec exactly, every wired page family renders with mocked client, and the build + full vitest suite are green. This is the capstone unit — it runs LAST after all builders. Do NOT modify any production code; only add test files under `src/test/`.

## Context (paths + exact current state)

- `/workspace/platform/frontend/src/index.css` — token layer (migrated by 01).
- `/workspace/platform/frontend/src/App.css` — legacy Vite scaffold (cleaned by 01).
- `/workspace/platform/frontend/src/App.tsx` — router (restructured by 02).
- `/workspace/platform/frontend/src/components/AppSidebar.tsx` — nav shell (restructured by 02).
- `/workspace/platform/frontend/src/pages/` — all page files (rewritten by 04, 05, 06).
- `/workspace/platform/frontend/src/lib/api/client.ts` — client methods (completed by 03).
- `/workspace/platform/frontend/src/test/setup.ts` — existing test setup (jsdom + localStorage shim).
- `/workspace/platform/frontend/src/test/dashboard-empty-stats.test.tsx` — existing test pattern (3.5KB, uses `vi.mock` on client, `MemoryRouter`, `AuthProvider`, `screen` queries).
- `/workspace/platform/frontend/vitest.config.ts` — vitest config (environment jsdom, setupFiles `./src/test/setup.ts`, include `src/**/*.{test,spec}.{ts,tsx}`).
- `/workspace/platform/docs/design/05-ia-menu-spec.md` — authoritative route map §4.
- `/workspace/platform/DESIGN.md` — Clear Deck token contract.

**Current test inventory** (verified 14 Sept):
- `src/test/dashboard-empty-stats.test.tsx` — 1 test file.
- `src/test/example.test.ts` — trivial placeholder.
- `src/test/setup.ts` — setup shim.

**Route map from 05 §4 (what the nav-contract test must verify):**
- /dashboard, /jobs, /jobs/:id, /shortlist, /applications, /interview, /upskill, /expand, /tools, /integrations, /profile, /settings, /setup, /admin
- DELETED: /markets, /billing, /rank
- ADD: /jobs/:id

**Token values to audit (from 01 spec + DESIGN.md):**
- Present: `1d4ed8`, `217 78 30`, `14 89 37`, `222 47 17`, `215 16 40`, `0 0% 100%`, `210 33% 91%`, `217 20% 94%`, `160 92% 21%`, `0 78 37`
- Absent: `225 25% 6%`, `165 80% 45%`, `210 100% 60%`, `0 72% 55%`, `38 92% 55%`, `150 70% 45%`, `0.75rem`

## Steps

### Step 1 — Token audit test (core — run first)
Write `src/test/tokens.test.ts`:
1. **index.css checks**: read `src/index.css` as text. Assert it contains `1d4ed8` (or `217 78 30`) and does NOT contain `225 25% 6%`, `165 80% 45%`, `210 100% 60%`, `0 72% 55%`, `38 92% 55%`, `150 70% 45%`, `0.75rem`.
2. **App.css legacy sweep**: read `src/App.css`. Assert it does NOT contain `#646cff`, `#61dafb`, `#888`.
3. **Component hex sweep**: recursively scan `src/pages/*.tsx` and `src/components/*.tsx` for `#[0-9a-fA-F]{3,6}`. Assert zero matches outside `src/components/ui/` (shadcn is allowed to have its own token classes). Use `fs.readdirSync` + `fs.readFileSync` in the test; do NOT use grep shell commands.
4. **Radius vars**: read `tailwind.config.ts`. Assert it contains `radius-lg`, `radius-md`, `radius-sm` and does NOT contain `0.75rem`.
5. **Run**: `npx vitest run src/test/tokens.test.ts` exits 0.

**Checkpoint 1:** `test -f src/test/tokens.test.ts` and `npx vitest run src/test/tokens.test.ts` exits 0.

### Step 2 — Nav contract test (core — run second)
Write `src/test/nav-contract.test.tsx`:
1. **Route existence**: read `src/App.tsx` as text. Assert it contains every route from 05 §4: `/dashboard`, `/jobs`, `/jobs/:id`, `/shortlist`, `/applications`, `/interview`, `/upskill`, `/expand`, `/tools`, `/integrations`, `/profile`, `/settings`, `/setup`, `/admin`.
2. **Deleted routes absent**: assert `src/App.tsx` does NOT contain `/markets` or `/billing` outside comments. Assert `/rank` is absent (replaced by `/shortlist`).
3. **Tools guard**: assert `/tools` route in App.tsx uses `AuthGuard` (not `AdminGuard`).
4. **Admin isolation**: assert `/admin` route uses `AdminGuard`.
5. **Sidebar group labels**: read `src/components/AppSidebar.tsx`. Assert it contains `MAIN`, `GROW`, `ACCOUNT` group labels.
6. **PRO chips**: assert AppSidebar.tsx contains `ProChip` component usage for Upskill, Expand, Tools, Integrations.
7. **Render test — free user sees GROW**: render `<MemoryRouter initialEntries={['/interview']}><AuthProvider><InterviewPrepPage /></AuthProvider></MemoryRouter>` with a mocked free-user localStorage session. Assert the page renders without redirecting away. Use `screen.getByText(/interview prep/i)`.
8. **Render test — admin sees only admin shell**: render `<MemoryRouter initialEntries={['/admin']}><AuthProvider><AdminDashboard /></AdminProvider></AuthGuard></MemoryRouter>` with mocked admin session. Assert admin dashboard renders.
9. **Run**: `npx vitest run src/test/nav-contract.test.tsx` exits 0.

**Checkpoint 2:** `test -f src/test/nav-contract.test.tsx` and `npx vitest run src/test/nav-contract.test.tsx` exits 0.

### Step 3 — Per-page-family smoke tests (batched — run third)
Write smoke tests for every page family. Each test mocks `@/lib/api/client` with `vi.mock`, sets a localStorage authed user, renders the page inside `MemoryRouter` + `AuthProvider`, and asserts the page heading renders without crashing.

**Batch 3a — Core pages** (Dashboard, Jobs, Shortlist, Applications):
- Write `src/test/pages-core.test.tsx`.
- Mock `api.getStats`, `api.getApplications`, `api.getJobs`, `api.runRank`.
- Render each page at its route. Assert a heading or unique text is present.
- Use `screen.findByText` with `async` to wait for loading to resolve.

**Batch 3b — GROW + Account pages** (InterviewPrep, Upskill, Expand, Tools, Integrations, Profile, Settings, Setup):
- Write `src/test/pages-grow-account.test.tsx`.
- Mock the client methods each page needs (from 03 spec).
- Render each page. Assert heading or unique text.

**Batch 3c — Public pages** (Landing, Login, Signup):
- Write `src/test/pages-public.test.tsx`.
- No auth needed for Landing. For Login/Signup, render without localStorage user.
- Assert headings render.

**Pattern for all smoke tests**:
```ts
vi.mock('@/lib/api/client', async (importOriginal) => {
  const actual = await importOriginal<typeof import('@/lib/api/client')>();
  return { ...actual, api: { ...actual.api, <method>: vi.fn().mockResolvedValue(<mock>) } };
});
```
Wrap each render in `cleanup()` afterEach. Use the existing `src/test/setup.ts` shims.

**Checkpoint 3:** `npx vitest run src/test/pages-*.test.tsx` exits 0 for all three batches.

### Step 4 — Build + full suite (capstone — run last)
1. Run `cd /workspace/platform/frontend && npx vitest run`. ALL tests (existing + new) must pass.
2. Run `npm run build`. Must exit 0.
3. Execute the GOAL.md done-criteria checklist verbatim (see below).

**GOAL.md done-criteria checklist** (run as a script or manually verify each):
```bash
# 1. Frontend builds + all tests green
npx vitest run && npm run build

# 2. Nav matches 05 §2/§3 exactly
grep -c "/shortlist" src/App.tsx && grep -c "/markets" src/App.tsx | grep -v ":0" || true && grep -c "/billing" src/App.tsx | grep -v ":0" || true

# 3. Token audit passes
grep -c "1d4ed8\|217 78 30" src/index.css && grep -c "225 25% 6%" src/index.css | grep ":0" && grep -c "165 80% 45%" src/index.css | grep ":0"

# 4. Every route in 05 §4 is wired (manual spot-check against grep output)
grep -oE 'path="/[^"]*"' src/App.tsx | sort

# 5. Zero hard-coded hex in components
grep -rn "#[0-9a-fA-F]\{6\}" src/pages/ src/components/ | grep -v "src/components/ui/" || true
```

**Checkpoint 4:** All commands exit 0 or produce expected output.

### Step 5 — Cut lines (resumability)
If the full suite exceeds 30 minutes of worker time, split into sequential sessions:
- **Session A**: Steps 1-2 (token audit + nav contract). Both files written and verified.
- **Session B**: Step 3 batch 3a (core pages smoke tests).
- **Session C**: Step 3 batch 3b (GROW + account smoke tests).
- **Session D**: Step 3 batch 3c (public pages smoke tests).
- **Session E**: Step 4 (full suite + build + checklist).

After each session, the next session reads the previous session's disk output and continues.

## Done criteria (grep-able)
- `test -f src/test/tokens.test.ts`
- `test -f src/test/nav-contract.test.tsx`
- `test -f src/test/pages-core.test.tsx`
- `test -f src/test/pages-grow-account.test.tsx`
- `test -f src/test/pages-public.test.tsx`
- `npx vitest run` exits 0 in `frontend/`
- `npm run build` exits 0 in `frontend/`
- `grep -c "1d4ed8\|217 78 30" src/index.css` ≥ 1
- `grep -c "225 25% 6%" src/index.css` = 0
- `grep -c "165 80% 45%" src/index.css` = 0
- `grep -c "0.75rem" src/index.css src/tailwind.config.ts` = 0
- `grep -c "radius-lg\|radius-md\|radius-sm" src/tailwind.config.ts` ≥ 3
- `grep -c "/shortlist" src/App.tsx` ≥ 2
- `grep -c "/markets" src/App.tsx | grep -v ":0" || true` produces 0 matches
- `grep -c "/billing" src/App.tsx | grep -v ":0" || true` produces 0 matches
- `grep -c "/rank" src/App.tsx | grep -v ":0" || true` produces 0 matches
- `grep -c "/jobs/:id" src/App.tsx` ≥ 1
- `grep -c "MAIN\|GROW\|ACCOUNT" src/components/AppSidebar.tsx` ≥ 3
- `grep -c "ProChip" src/components/AppSidebar.tsx` ≥ 4

## Output (files touched)
- `src/test/tokens.test.ts` — NEW
- `src/test/nav-contract.test.tsx` — NEW
- `src/test/pages-core.test.tsx` — NEW
- `src/test/pages-grow-account.test.tsx` — NEW
- `src/test/pages-public.test.tsx` — NEW

## Depends
- `01-builder-tokens` (token migration must be on disk)
- `02-builder-nav` (nav restructure must be on disk)
- `03-builder-client-methods` (client methods must exist)
- `04-builder-pages-grow` (GROW pages must be on disk)
- `05-builder-pages-core` (core pages must be on disk)
- `06-builder-pages-account` (account pages must be on disk)

## Result
(pending)
