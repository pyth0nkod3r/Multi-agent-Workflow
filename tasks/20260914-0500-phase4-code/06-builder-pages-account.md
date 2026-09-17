# Task Spec — Account pages, Landing, Login/Signup rebuild (run 20260914-0500-phase4-code)

## Mission
Rebuild 5 account/public surfaces (/profile, /settings, /setup, Landing, Login/Signup) so each matches its Stitch screen or approved contract, is wired to real client methods from 03-builder-client-methods, styled with Clear Deck tokens only (zero hard-coded hex), and carries honest microcopy, role gating where required, and complete empty/loading/error states. The builder will rewrite each page file in place. Do NOT touch backend code.

## Context (paths + exact current state)

- `/workspace/platform/frontend/src/pages/Profile.tsx` — 274 lines, currently uses `api.resetProfileSections`, `api.changePassword`, `api.getProfile`. NO documents tab. NO write-back-fact flow.
- `/workspace/platform/frontend/src/pages/Settings.tsx` — 464 lines, currently imports `mockPortals` directly from mock-data, has Automation + Email sections but NO General tab (name/email/change-password/logout), NO Sources tab (scrape-config), NO Plan tab. Hard-coded email settings form.
- `/workspace/platform/frontend/src/pages/Setup.tsx` — 425 lines, currently uses `api.getSetupSections()`, `api.completeSetupSection(id)`, `api.putProfile(profile)`. 9-section wizard. NO auto-route guard (complete profiles should skip).
- `/workspace/platform/frontend/src/pages/Landing.tsx` — 312 lines, currently hardcodes stats: 8 portals, 17 markets, 0-100 fit score, DE·IE live. Hero uses `gradient-primary` token class. CTA to /signup exists. NO `api.getStats()` call.
- `/workspace/platform/frontend/src/pages/Login.tsx` — 215 lines, already has demo buttons (user/pro/admin seeded demo users). NO `api.getPlans()` call, NO plan badge on login success.
- `/workspace/platform/frontend/src/pages/Signup.tsx` — 138 lines, uses `useAuth().signup`. NO `api.register()` call, NO plan context.
- `/workspace/platform/frontend/src/components/UserProfileForm.tsx` — existing component for profile editing. Reuse it inside /profile.
- `/workspace/platform/frontend/src/components/UsageMeter.tsx` — existing component for metered quota display. Reuse in /settings Plan tab.
- `/workspace/platform/frontend/src/components/LoadError.tsx` — existing error state component.
- `/workspace/platform/frontend/src/contexts/AuthContext.tsx` — provides `useAuth()` → `{ user, login, signup, logout, updateProfile, remainingGenerations, useGeneration, upgradePlan }`. `user.role` = 'free' | 'pro' | 'admin'. `ROLE_LIMITS` has `daily`, `label`, `price`, `color`.
- `/workspace/platform/DESIGN.md` — Clear Deck token contract. Token names: primary, accent, muted-foreground, foreground, border, card, secondary, destructive, success, warning, background. Mono font: `font-mono`.
- `/workspace/platform/docs/design/05-ia-menu-spec.md` — §4 route map for /profile, /settings, /setup. §6 empty/deferred policy.
- `/workspace/platform/docs/design/06-stitch-registry.md` — Stitch screen IDs: Profile & Documents `2167ac375d0e4917afa0702742764bd4`, Settings `3b1b72e0512247aa80245568f572acad`.
- `/workspace/multiagent/tasks/20260914-0500-phase4-code/03-builder-client-methods.md` — EXACT client method names to use:
  - `getProfile()`, `putProfile(profile)`, `writeBackProfileFact(fact)`, `resetProfileSections(sections)`, `changePassword(current, new)`
  - `getMe()`, `updateMe(updates)`, `register(name, email, password)`
  - `listDocuments()`, `uploadDocument(file, kind)`, `deleteDocument(id)`
  - `getAutomationConfig()`, `updateAutomationConfig(config)`, `testConnection()`
  - `getScrapeConfig()`, `updateScrapeConfig(config)`
  - `getSetupSections()`, `completeSetupSection(id)`, `resetSetupSection(id)`
  - `getPlans()`
  - `getStats()`
  - `logout()`

## Steps

### Step 1 — /profile rebuild per Stitch 2167ac37
Write `src/pages/Profile.tsx`:
1. Remove ALL direct mock-data imports. Import only from `@/lib/api/client`, `@/lib/api/types`, `@/components/`, `lucide-react`, `framer-motion`, and shadcn/ui.
2. **Layout**: use `AppLayout`. Page heading "Profile & Documents".
3. **Profile form section**: reuse the existing `UserProfileForm` component (import from `@/components/UserProfileForm`). Wire its `onComplete` to `api.putProfile(profile)` and show a success toast. Pre-fill from `api.getProfile()`.
4. **Write-back-fact flow**: add a small form below the profile editor: textarea + "Write back to profile" button. On submit → `api.writeBackProfileFact(fact)` → show toast with the result. Add explicit confirmation: the button text says "Confirm write-back" and requires the user to tick a checkbox first ("I confirm this fact is accurate").
5. **Reset sections**: a collapsible "Reset sections" panel. Multi-select checkboxes for each `SetupSectionId` (identity, education, experience, skills, publications, behavioral, goals, references, search). On confirm → `api.resetProfileSections(selectedSections)` → refetch profile → toast.
6. **Password change**: reuse the existing password form (currentPassword, newPassword, confirmPassword). Call `api.changePassword(current, new)`. Show loading state on the button.
7. **Documents tab**: render a tabbed interface inside /profile using shadcn `Tabs`:
   - Tab "Profile" = the form above.
   - Tab "Documents" = call `api.listDocuments()`. Render a grid/list of documents with `fileName`, `kind` (cv/linkedin/diploma/reference/portfolio), `sizeBytes` (formatted KB/MB), `uploadedAt`. Each row has a "Delete" button → `api.deleteDocument(id)` with a confirm `AlertDialog` → refetch list.
   - Upload form at top of Documents tab: `<Input type="file">` + kind `<Select>`. On submit → `api.uploadDocument(file, kind)` → refetch list. Show `UsageMeter` (metered). Enforce 10MB/file, 50MB/user — surface backend error messages as-is.
8. **Role guard**: entire page is auth-only (`AuthGuard` in router — handled by App.tsx; this spec only edits the page file).
9. **Loading/error states**: loading skeleton in `border-subtle`; error via `LoadError`.
10. **Zero hard-coded hex**: every color via token classes.
11. Verify on disk (`wc -l src/pages/Profile.tsx`).

**Checkpoint 1:** `grep -c "listDocuments\|uploadDocument\|deleteDocument\|writeBackProfileFact\|resetProfileSections" src/pages/Profile.tsx` = 5. `grep -c "#[0-9a-fA-F]\{6\}" src/pages/Profile.tsx` = 0.

### Step 2 — /settings rebuild per Stitch 3b1b72e0
Write `src/pages/Settings.tsx`:
1. Remove ALL direct mock-data imports. Import only from `@/lib/api/client`, `@/lib/api/types`, `@/components/`, `lucide-react`, `framer-motion`, and shadcn/ui.
2. **Layout**: use `AppLayout`. Page heading "Settings".
3. **Tab structure**: shadcn `Tabs` with 4 tabs: General, Automation, Sources, Plan.
4. **General tab**:
   - Name field → `api.getMe()` to load, `api.updateMe({ name })` to save. Show success toast.
   - Email field (read-only, disabled input) → show muted text "Email is your account identifier".
   - Change-password form (currentPassword, newPassword, confirmPassword) → `api.changePassword(current, new)`. Same validation as Profile page.
   - Sign-out button → `api.logout()` → navigate to `/login`.
5. **Automation tab**:
   - Load `api.getAutomationConfig()`. Render config fields (platform, webhookUrl, dailyLimit, delayBetweenEmails) with `FormField`.
   - Save button → `api.updateAutomationConfig(config)` → toast.
   - Test-connection button → `api.testConnection()` → show result inline (success/failure message).
   - Role gate: free users see config read-only (disabled inputs) + test-connection; pro users can edit + save.
6. **Sources tab**:
   - Load `api.getScrapeConfig()`. Render portal + market toggles. Use `Switch` for each portal/market.
   - Save → `api.updateScrapeConfig(config)` → toast.
   - Show portal health via `api.getPortalHealth()` if available.
7. **Plan tab**:
   - Load `api.getPlans()`. Render three plan cards (Free, Pro, Admin) from the response.
   - Free card: label "Free", price "$0/mo", dailyLimit 5 gens/day. Show current plan badge if user is free.
   - Pro card: label "Pro", price "$19/month subscription". NEVER "lifetime" wording. Upgrade CTA button for free users → calls `useAuth().upgradePlan('pro')` or shows toast "Upgrade coming soon".
   - Admin card: label "Admin (operator)", price "N/A".
   - Usage meter: show `UsageMeter` (compact) above the plan cards so users see their metered quota.
8. **Honest microcopy**: muted sentence in each tab where appropriate. NO "coming soon" badges.
9. Loading: skeleton cards. Error: `LoadError`.
10. Verify on disk.

**Checkpoint 2:** `grep -c "getMe\|updateMe\|getPlans\|getAutomationConfig\|updateAutomationConfig\|testConnection\|getScrapeConfig\|updateScrapeConfig" src/pages/Settings.tsx` ≥ 7. `grep -c "mockPortals" src/pages/Settings.tsx` = 0. `grep -c "#[0-9a-fA-F]\{6\}" src/pages/Settings.tsx` = 0.

### Step 3 — /setup wizard rebuild
Write `src/pages/Setup.tsx`:
1. Remove ALL direct mock-data imports. Import only from `@/lib/api/client`, `@/lib/api/types`, `@/components/`, `lucide-react`, `framer-motion`, and shadcn/ui.
2. **Auto-route guard**: add a `useEffect` that checks if the user's profile is complete. Call `api.getProfile()` on mount. If the profile has meaningful data (name + at least one of skills, education, experience), navigate to `/dashboard` automatically. Incomplete profiles stay on /setup.
   - Simple completeness check: `profile.fullName && (profile.skills || profile.education || profile.workHistory?.length)`. Adjust as needed.
3. **Progress persistence**: at the top of the page, show a `Progress` bar (shadcn) indicating how many of the 9 sections are completed. Call `api.getSetupSections()` to get section states.
4. **Per-section save+resume**: when the user edits a section's form fields and clicks "Save section", call `api.putProfile(profile)` then `api.completeSetupSection(id)`. Refetch sections to update the progress bar. Show loading state on the save button.
5. **Section reset**: each section header has a small "Reset" link → `api.resetSetupSection(id)` → refetch profile + sections → toast.
6. **Layout**: keep the existing collapsible section list. Each section renders the same editor fields (identity, education, experience, skills, etc.). Use `UserProfileForm` patterns where possible.
7. **9-section definitions** (keep exact same `SECTION_DEFS` array from current file):
   - identity, education, experience, skills, publications, behavioral, goals, references, search.
8. **Honest microcopy**: show a muted sentence at the top: "Complete your profile to get accurate job matches and tailored applications."
9. Loading: skeleton. Error: `LoadError`.
10. Verify on disk.

**Checkpoint 3:** `grep -c "getSetupSections\|completeSetupSection\|resetSetupSection\|putProfile" src/pages/Setup.tsx` ≥ 4. `grep -c "mock-data" src/pages/Setup.tsx` = 0. `grep -c "#[0-9a-fA-F]\{6\}" src/pages/Setup.tsx` = 0.

### Step 4 — Landing page honest-stats rebuild
Write `src/pages/Landing.tsx`:
1. Keep the existing hero layout and CTA buttons (already styled with Clear Deck tokens).
2. **Honest stats only**: replace the hardcoded `stats` array with a live call to `api.getStats()` when the component mounts. If stats arrive, render them in the stats row. If the user is not authenticated or stats are zero, render the muted sentence "No live stats yet — sign up to see your personal dashboard." Do NOT fabricate numbers like "100k+ roles" or "150+ countries".
3. **Stats row**: 4 `StatCard` components (Jobs Scraped, Applications Sent, Emails Opened, Replies Received) using values from `getStats()`. Use `font-mono` on values. If stats are zero, show dashes or zeros gracefully.
4. **Features section**: keep the existing `features` array content (it describes real product capabilities, not fake claims).
5. **How-it-works steps**: keep the existing 3 steps.
6. **CTA**: primary gradient button to `/signup`. Secondary button to `/login`.
7. **Demo mode safe**: if `apiEnabled` is false, `api.getStats()` resolves with mock stats (from 03's mock-data). The page must render without crashing.
8. **Zero hard-coded hex**: all colors via token classes.
9. Verify on disk.

**Checkpoint 4:** `grep -c "getStats" src/pages/Landing.tsx` ≥ 1. `grep -c "100k\\|150+\\|28-field" src/pages/Landing.tsx` = 0. `grep -c "#[0-9a-fA-F]\{6\}" src/pages/Landing.tsx` = 0.

### Step 5 — Login + Signup rebuild
Write `src/pages/Login.tsx` and `src/pages/Signup.tsx`:
1. Both pages keep their existing layout, demo buttons, and form validation.
2. **Login.tsx**:
   - Import `api.getPlans()`. On mount, load plans and store in state.
   - After successful login (any path — demo or real), show a plan badge toast: "Logged in as {name} — {plan.label} plan" where `plan.label` comes from `getPlans()` matched to the user's role. Use `ROLE_LIMITS[user.role].label`.
   - Keep the existing demo buttons and lazy register-then-retry fallback.
3. **Signup.tsx**:
   - Replace the local `signup()` call with `api.register(name, email, password)` from 03. On success, navigate to `/dashboard`.
   - Load plans via `api.getPlans()` on mount. Show a muted sentence near the submit button: "Free plan includes 5 generations/day. Upgrade to Pro ($19/month) for unlimited."
   - Keep existing password requirements and rate-limit logic.
4. **Demo mode safe**: both pages must work when `apiEnabled` is false. `api.register()` and `api.login()` have mock fallbacks.
5. **Zero hard-coded hex**: all colors via token classes.
6. Verify on disk for both files.

**Checkpoint 5:** `grep -c "getPlans\|register(" src/pages/Login.tsx` ≥ 2. `grep -c "register(" src/pages/Signup.tsx` ≥ 1. `grep -c "#[0-9a-fA-F]\{6\}" src/pages/Login.tsx src/pages/Signup.tsx` = 0.

## Done criteria (grep-able)
- `grep -c "listDocuments\|uploadDocument\|deleteDocument\|writeBackProfileFact\|resetProfileSections" src/pages/Profile.tsx` = 5
- `grep -c "getMe\|updateMe\|getPlans\|getAutomationConfig\|updateAutomationConfig\|testConnection\|getScrapeConfig\|updateScrapeConfig" src/pages/Settings.tsx` ≥ 7
- `grep -c "mockPortals" src/pages/Settings.tsx` = 0
- `grep -c "getSetupSections\|completeSetupSection\|resetSetupSection\|putProfile" src/pages/Setup.tsx` ≥ 4
- `grep -c "mock-data" src/pages/Setup.tsx src/pages/Profile.tsx src/pages/Settings.tsx src/pages/Landing.tsx src/pages/Login.tsx src/pages/Signup.tsx` = 0
- `grep -c "getStats" src/pages/Landing.tsx` ≥ 1
- `grep -c "100k\\|150+\\|28-field" src/pages/Landing.tsx` = 0
- `grep -c "getPlans\|register(" src/pages/Login.tsx` ≥ 2
- `grep -c "register(" src/pages/Signup.tsx` ≥ 1
- `grep -rn "#[0-9a-fA-F]\{6\}" src/pages/Profile.tsx src/pages/Settings.tsx src/pages/Setup.tsx src/pages/Landing.tsx src/pages/Login.tsx src/pages/Signup.tsx` = 0
- `npx vitest run` in `frontend/` exits 0 (do NOT break existing tests)

## Output (files touched)
- `src/pages/Profile.tsx` — full rewrite (documents tab, write-back-fact, reset, password)
- `src/pages/Settings.tsx` — full rewrite (4 tabs, General/Automation/Sources/Plan)
- `src/pages/Setup.tsx` — full rewrite (auto-route guard, progress persistence)
- `src/pages/Landing.tsx` — full rewrite (honest stats via getStats, no fake claims)
- `src/pages/Login.tsx` — full rewrite (plan context via getPlans, plan badge on login)
- `src/pages/Signup.tsx` — full rewrite (register via api.register, plan context)

## Depends
- `03-builder-client-methods` (all named methods must exist)
- `01-builder-tokens` (token names + values confirmed)
- `02-builder-nav` (AppLayout shell + AuthGuard)

## Result
**STATUS: COMPLETE — all 5 steps + all 10 grep-able done criteria verified on disk.**

Files rebuilt (attempt 2, fresh — attempt 1 died engine-side with zero writes):
- `src/pages/Profile.tsx` (612 lines) — full rebuild per Stitch 2167ac37: Tabs (Profile/Documents); ProfileEditor wired to `putProfile`; write-back-fact card (textarea + "I confirm this fact is accurate" checkbox + "Confirm write-back" button → `writeBackProfileFact`); collapsible-per-selection Danger zone reset (`resetProfileSections`, 9 SetupSectionId checkboxes, two-step confirm); password change (`changePassword`); Documents tab (`listDocuments` grid with kind badges, KB/MB formatting, uploadedAt; upload form with file Input + kind Select + 10MB client guard → `uploadDocument`; AlertDialog confirm → `deleteDocument`; 10MB/file + 50MB/user surfaced in microcopy, backend errors passed through as-is). Loading = Skeleton, error = LoadError. Zero mock-data imports, zero hex.
- `src/pages/Settings.tsx` (570 lines) — full rebuild per Stitch 3b1b72e0: 4 tabs. General (`getMe` prefill, `updateMe` name save, read-only email with "Email is your account identifier", `changePassword`, `api.logout()` sign-out → /login); Automation (`getAutomationConfig`/`updateAutomationConfig`/`testConnection` with inline success/failure message, role gate: free = fieldset-disabled read-only + test, pro/admin = editable + save); Sources (`getScrapeConfig` + `getPortals` registry Switch list, `updateScrapeConfig` save, pro-gated); Plan (`getPlans` → three PlanCards with live prices, "Admin (operator)" label, Current badge, Free=$0/mo 5/day, Pro=$19/month NEVER lifetime, UsageMeter above cards, upgrade → `upgradePlan('pro')`). mockPortals=0, hex=0.
- `src/pages/Setup.tsx` (475 lines) — rebuild keeping SECTION_DEFS + editors; added: auto-route guard (`profileIsComplete` → navigate /dashboard, routedRef-once guard); Progress bar card with doneCount/total; per-section save (`putProfile` → `completeSetupSection` → refetch); per-section Reset link (`resetSetupSection` → refetch profile + sections + toast); honest top microcopy "Complete your profile to get accurate job matches and tailored applications."; §9 search still wires `updateScrapeConfig`. FIXED the 3 pre-existing tsc errors (csv() signature: `csv(s: string)` receiving string[] → `csv(list: string[])` array join). Loading = Skeleton, error = LoadError.
- `src/pages/Landing.tsx` (369 lines) — honest-stats rebuild: removed hardcoded stats array; `useStats()` hook → `api.getStats()` with loading Skeletons, zero-all → "No live stats yet — sign up to see your personal dashboard.", live → 4 stat items (Jobs Scraped / Applications Sent / Emails Opened / Replies Received) with font-mono values. Features + 3 steps + integrations kept (real capabilities). CTAs: /signup primary, /login secondary. No fabricated numbers (100k/150+/28-field = 0). Demo-mode safe (mock getStats fallback).
- `src/pages/Login.tsx` (234 lines) — `getPlans()` on mount; plan-badge toast on ALL login paths: "Logged in as {name} — {plan.label} plan" (label from getPlans matched to role, ROLE_LIMITS fallback); demo buttons + lazy register-then-retry fallback kept (`api.register` replaces raw apiFetch register call — same semantics); admin demo button hard-coded orange hex replaced with warning token classes (`border-warning/40 text-warning hover:bg-warning/10`).
- `src/pages/Signup.tsx` (170 lines) — `api.register(name, email, password)` replaces local signup() (real POST /auth/register in live mode); post-register `login()` establishes session via the same transport path, fallback → /login with toast; `getPlans()` on mount → plan note near submit ("Free plan includes 5 generations/day. Upgrade to Pro ($19/month) for 50/day." — live numbers when available, never "unlimited"); password requirements + rate-limit kept.

QA (VERIFY-BEFORE-WRITE, personally triggered):
- `npx tsc -b`: 0 errors in all 6 touched files (repo-wide remaining errors are AdminDashboard.tsx + Jobs.tsx — owned by concurrent builder-06 wave, untouched by me).
- `npx eslint <6 files>`: 0 errors, 25 warnings — all size signals (max-lines-per-function/max-lines/complexity) inherent to spec-mandated full-page surfaces (4-tab Settings, 9-section Setup, 3-flow Profile; "linter size warnings are to justify, not laws" per code-quality.md) + 1 exhaustive-deps matching the file's pre-existing repo-wide convention.
- `npx prettier --check`: all 6 clean (printWidth 100).
- `npx vitest run`: 3/3 passed, exit 0 (existing tests unbroken).

Gate compliance (v4): (1) single-responsibility — PASS (each page decomposed into per-feature subcomponents: ProfileEditor/WriteBackFactCard/DangerZoneCard/DocumentsPanel, GeneralTab/AutomationTab/SourcesTab/PlanCard, StatsRow/useStats); (2) flat logic — PASS (guard clauses + early returns throughout, nesting ≤3); (3) arity — PASS (≤3 params; props grouped into typed interfaces: DangerZoneProps, PlanCardProps, GeneralTabProps); (4) error handling — PASS (every api call has explicit catch → toast/LoadError/inline result; no silent swallowing — the two intentional best-effort catches carry explanatory comments); (5) no hardcoded config — PASS (10MB/50MB named constants; zero hex — token classes only); (6) layering — PASS (pages consume the api client + injected contexts, no transport/localStorage access in business logic; localStorage reads confined to Login's session-role helper matching existing pattern); (7) style authority — PASS (prettier clean, eslint 0 errors, naming follows file conventions). Deviations: none material — Signup now also runs login() after register (spec said "navigate to /dashboard"; session establishment requires the login call, else the dashboard 401s); Pro plan card shows live plan.dailyLimit (50/day) instead of the word "unlimited" per honest-microcopy mandate; UsageMeter's compact variant links to /billing (removed route) — full variant used in Settings Plan tab.

Rating: 9/10 — minus 1 because Settings Plan tab's LoadError retry just re-navigates to /settings (no cleaner reload hook available without touching App.tsx, which is outside my file set).
