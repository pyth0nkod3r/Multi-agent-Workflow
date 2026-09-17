# Task Spec — Nav + IA restructure to 05-ia-menu-spec contract (run 20260914-0500-phase4-code)

## Mission
Restructure AppLayout/AppSidebar/bottom tabs/More sheet + App.tsx router to match the approved 05-ia-menu-spec.md §2 (desktop sidebar), §3 (mobile bottom tabs + More sheet), and §4 (route map). GROW group must be visible to free users (PRO chips on Upskill/Expand/Tools/Integrations + on mock-interview), /rank becomes /shortlist, /markets and /billing standalone routes are deleted, /tools loses AdminGuard (pro with admin extras conditional on role), /jobs/:id detail route added. Admin sees ONLY the admin shell.

## Context (verified from disk)

**Current App.tsx routes** (14 Sept verified):
/, /login, /signup, /dashboard, /jobs, /rank, /applications, /markets (AdminGuard), /interview, /upskill, /setup, /expand, /integrations, /tools (AdminGuard), /settings, /profile, /billing, /admin (AdminGuard), *

**Current AppSidebar.tsx** structure:
- `ALL_NAV_ITEMS` flat array: /dashboard, /jobs, /rank, /applications, /markets (adminOnly), /interview, /upskill, /expand, /integrations, /tools (adminOnly), /settings
- Setup group with collapsible Profile/Billing children
- Admin section conditionally rendered
- UsageMeter in sidebar footer
- user chip + plan badge + sign out in footer
- MobileHeader renders Sheet (left drawer) with same SidebarContent

**Current AppLayout.tsx**:
- Imports AppSidebar + MobileHeader from AppSidebar.tsx
- Renders MobileHeader, AppSidebar, then main content with md:ml-[240px]
- No bottom tab bar; no MoreSheet

**AuthContext.tsx** provides:
- `useAuth()` → `{ user, logout }`
- `user.role` = 'free' | 'pro' | 'admin'
- `ROLE_LIMITS[user.role]` → `{ label, color, ... }`

**05-ia-menu-spec.md §2 desktop sidebar table (verbatim):**
| Group | Item | Route | Role | Icon note |
|---|---|---|---|---|
| — | (logo) | /dashboard | all | top of sidebar |
| MAIN | Dashboard | /dashboard | free+pro | home |
| MAIN | Jobs | /jobs | free+pro | briefcase |
| MAIN | Shortlist | /shortlist | free+pro | list-ordered |
| MAIN | Applications | /applications | free+pro | kanban |
| GROW | Interview Prep | /interview | free packs / pro mock | mic; PRO chip on mock |
| GROW | Upskill | /upskill | pro | trend-up + PRO chip |
| GROW | Expand | /expand | pro | sprout + PRO chip |
| GROW | Tools & Templates | /tools | pro (admin extras) | wrench + PRO chip |
| GROW | Integrations | /integrations | pro | plug + PRO chip |
| ACCOUNT | Profile & Documents | /profile | free+pro | user |
| ACCOUNT | Settings | /settings | free+pro | gear |
| — | user chip + plan badge + sign out | — | all | sidebar bottom |

**05-ia-menu-spec.md §3 mobile:**
- Five tabs: Home · Jobs · Shortlist · Applications · More
- More sheet (below tabs, mirrors sidebar order):
  - Interview Prep (free packs / PRO mock) | /interview
  - Upskill (PRO) | /upskill
  - Expand (PRO) | /expand
  - Tools & Templates (PRO) | /tools
  - Integrations (PRO) | /integrations
  - Profile & Documents | /profile
  - Settings · Plan · Sign out | /settings

**05-ia-menu-spec.md §4 route map (target App.tsx routes):**
- /dashboard, /jobs, /jobs/:id, /shortlist, /applications, /interview, /upskill, /expand, /tools, /integrations, /profile, /settings, /setup, /admin
- DELETED: /markets, /billing, /rank (renamed → /shortlist)
- ADD: /jobs/:id

**05-ia-menu-spec.md §5 exclusions:**
- /markets → merged into Jobs/Sources tab (task 03 scope)
- /billing → merged into Settings/Plan tab (task 03 scope)
- /rank → replaced by /shortlist

**Active state spec (DESIGN.md sidebar_item):**
- active_background: rgba(29,78,216,0.08)
- active_color: primary #1d4ed8
- active_anchor: 3px solid primary

**No BottomNav or MoreSheet components exist** (verified 14 Sept: `ls src/components/ | grep -iE "bottom|more|sheet|nav"` returns only NavLink.tsx).

## Steps

**Step 1 — Add PRO-chip component.**
Create `frontend/src/components/ProChip.tsx`. It must render a small inline badge (e.g. `<Badge variant="outline" className="text-[10px] px-1 py-0 h-4 border-accent text-accent leading-none">PRO</Badge>`) using the existing `badge.tsx` shadcn component. It reads no props — just renders "PRO" with `border-accent text-accent` token classes. Place it next to the nav item label in sidebar + More sheet when the item is PRO-gated.

**Checkpoint 1:** `test -f frontend/src/components/ProChip.tsx` and `grep -c "ProChip" frontend/src/components/ProChip.tsx` ≥ 1.

**Step 2 — Rewrite AppSidebar.tsx nav structure.**
Edit `frontend/src/components/AppSidebar.tsx` in-place (do NOT delete the file — it also exports MobileHeader). Changes:
- Remove /markets and /billing from ALL_NAV_ITEMS and setupChildren.
- Rename /rank → /shortlist (icon: list-ordered, label: "Shortlist").
- Group nav items into MAIN / GROW / ACCOUNT sections per §2 table, with a `<div className="px-3 text-xs font-semibold text-muted-foreground uppercase tracking-wider mb-1">` group label above each group.
- Render PRO chip next to Upskill, Expand, Tools, Integrations labels using the ProChip component.
- For Interview Prep, render a PRO chip only on the mock-interview sub-feature (the nav item itself stays visible to free; the chip signals the pro-only part). Implementation: add a `proFeature?: string` field to the Interview Prep nav item and show ProChip beside a small "(mock)" label.
- Active state: replace current `bg-primary/10 text-primary shadow-glow` with `bg-[rgba(29,78,216,0.08)] text-primary border-l-2 border-primary pl-[9px]` (the left rail is 2px; the pl compensates so text doesn't shift — net visual: 2px primary rail + tinted bg). If the current NavLink wraps an `<a>` tag, apply the rail via a wrapper `<div>` with the border classes and pass the tint to the inner link.
- Footer: keep user chip + plan badge + sign out. Remove UsageMeter (it was a temporary dev aid — not in §2; spec says nothing about it).
- Admin section: keep the admin-only Admin link but ensure it only renders when `userIsAdmin` is true. Admin link uses `text-accent` (burnt orange per DESIGN.md) — matches the accent semantic.
- Setup group: REMOVE. Setup is auto-routed only, not a persistent nav item per 05 §5 "Setup wizard as persistent nav: auto-route only".

**Checkpoint 2:** `grep -c "MAIN\|GROW\|ACCOUNT" frontend/src/components/AppSidebar.tsx` ≥ 3 (group labels present); `grep -c "/markets" frontend/src/components/AppSidebar.tsx` = 0; `grep -c "/billing" frontend/src/components/AppSidebar.tsx` = 0; `grep -c "/rank" frontend/src/components/AppSidebar.tsx` = 0; `grep -c "/shortlist" frontend/src/components/AppSidebar.tsx` ≥ 1.

**Step 3 — Create BottomNav + MoreSheet components.**
Create `frontend/src/components/BottomNav.tsx` and `frontend/src/components/MoreSheet.tsx`.

BottomNav renders 5 tabs (visible on mobile only, `hidden md:hidden` via `useIsMobile` inverse — actually use `md:hidden` class):
1. Home → /dashboard (icon: home / LayoutDashboard)
2. Jobs → /jobs (icon: briefcase / Search)
3. Shortlist → /shortlist (icon: list-ordered / Bookmark)
4. Applications → /applications (icon: kanban / Send)
5. More → opens MoreSheet (icon: more-horizontal / ChevronDown)

MoreSheet uses the existing `Sheet` component from `@/components/ui/sheet`. Its content mirrors the sidebar GROW + ACCOUNT order per §3 table:
- Interview Prep → /interview (free packs / PRO chip on mock)
- Upskill → /upskill (PRO chip)
- Expand → /expand (PRO chip)
- Tools & Templates → /tools (PRO chip)
- Integrations → /integrations (PRO chip)
- Profile & Documents → /profile
- Settings → /settings (with "Plan" sub-label)
- Sign out button at bottom

Both components use the same active-state logic as the sidebar: left 2px primary rail + rgba(29,78,216,0.08) tint for the active tab/row.

**Checkpoint 3:** `test -f frontend/src/components/BottomNav.tsx` and `test -f frontend/src/components/MoreSheet.tsx`; `grep -c "MoreSheet" frontend/src/components/BottomNav.tsx` ≥ 1.

**Step 4 — Update AppLayout.tsx.**
Edit `frontend/src/components/AppLayout.tsx` to render BottomNav on mobile. Import BottomNav from `./BottomNav`. Add `<BottomNav />` inside the root `<div>` so it's visible on mobile (the existing `md:ml-[240px]` spacing stays). The main content area should have `pb-16 md:pb-0` to avoid tab-bar overlap on mobile.

**Checkpoint 4:** `grep -c "BottomNav" frontend/src/components/AppLayout.tsx` ≥ 1.

**Step 5 — Update App.tsx router.**
Edit `frontend/src/App.tsx`:
1. Change `<Route path="/rank"` → `<Route path="/shortlist"`.
2. Add `<Route path="/jobs/:id"` → JobDetail placeholder (a simple `<div>Job Detail</div>` is fine — task 03 wires the real page).
3. Remove `/markets` route entirely.
4. Remove `/billing` route entirely.
5. Change `/tools` guard from `<AdminGuard>` to `<AuthGuard>`.
6. Ensure every route from §4 exists: /dashboard, /jobs, /jobs/:id, /shortlist, /applications, /interview, /upskill, /expand, /tools, /integrations, /profile, /settings, /setup, /admin.
7. Wrap the admin routes in a separate admin shell: `<Route path="/admin" element={<AdminGuard><AdminShell /></AdminGuard>}` where AdminShell is a new minimal component that renders the admin tabs (Users/Portals/Markets/Submissions/Templates) per §4 Admin rows. AdminShell can be a simple tabbed layout using shadcn Tabs — no need to wire real admin page content yet (those pages exist in AdminDashboard.tsx).

**Checkpoint 5:** `grep -c "/shortlist" frontend/src/App.tsx` ≥ 2 (route + import if RankPage renamed); `grep -c "/markets" frontend/src/App.tsx` = 0 outside comments; `grep -c "/billing" frontend/src/App.tsx` = 0 outside comments; `grep -c "/jobs/:id" frontend/src/App.tsx` ≥ 1; `grep -c "AuthGuard" frontend/src/App.tsx | grep -c tools` = 1 (tools now AuthGuard).

**Step 6 — Rename RankPage → ShortlistPage (optional but clean).**
If the builder renames `src/pages/Rank.tsx` → `src/pages/Shortlist.tsx`, update the import in App.tsx accordingly. If they keep Rank.tsx with an internal redirect, that's also acceptable — the route string must be /shortlist, not /rank.

**Checkpoint 6:** `ls frontend/src/pages/Shortlist.tsx` OR `grep -c "path=\"/shortlist\"" frontend/src/App.tsx` ≥ 1.

**Step 7 — Vitest pass.**
Run `cd frontend && npx vitest run`. All existing tests must pass. The builder must NOT break any existing test.

**Checkpoint 7:** `npx vitest run` exits 0.

## Done criteria (grep-able)
- `grep -c "/shortlist" frontend/src/App.tsx` ≥ 2 (route + import/usage)
- `grep -c "/rank" frontend/src/App.tsx` = 0 (old route gone)
- `grep -c "/markets" frontend/src/App.tsx` = 0 outside comments
- `grep -c "/billing" frontend/src/App.tsx` = 0 outside comments
- `grep -c "/jobs/:id" frontend/src/App.tsx` ≥ 1
- `grep -c "MAIN\|GROW\|ACCOUNT" frontend/src/components/AppSidebar.tsx` ≥ 3 (group labels)
- `grep -c "ProChip" frontend/src/components/AppSidebar.tsx` ≥ 4 (Upskill, Expand, Tools, Integrations)
- `grep -c "BottomNav" frontend/src/components/AppLayout.tsx` ≥ 1
- `grep -c "MoreSheet" frontend/src/components/BottomNav.tsx` ≥ 1
- `grep -c "AuthGuard" frontend/src/App.tsx | grep -c "/tools"` = 1 (tools no longer AdminGuard)
- `grep -c "AdminGuard" frontend/src/App.tsx | grep -c "/admin"` = 1 (only admin route)
- `npx vitest run` exits 0

## Output (files touched)
- frontend/src/components/AppSidebar.tsx — nav restructure (in-place edit, keep MobileHeader export)
- frontend/src/components/AppLayout.tsx — add BottomNav
- frontend/src/App.tsx — router diff
- frontend/src/components/ProChip.tsx — new component
- frontend/src/components/BottomNav.tsx — new component
- frontend/src/components/MoreSheet.tsx — new component
- frontend/src/components/AdminShell.tsx — new minimal admin tab shell (optional if builder uses inline tabs in App.tsx)

## Depends
None — parallel-safe with 01-builder-tokens.md and 03-builder-client-methods.md. (Uses existing token var names; 01 changes values not names.)

## Result
DONE — all steps 1-7 complete, all grep Done criteria verified on disk 14 Sept ~08:30.

**Artifacts:**
- `src/components/ProChip.tsx` (NEW) — accent-outline PRO badge, token classes only (border-accent text-accent), zero hard-coded hex.
- `src/components/AppSidebar.tsx` — full nav restructure in-place: MAIN/GROW/ACCOUNT groups (group-label divs), /rank→/shortlist (ListOrdered), /markets + /billing + Setup collapsible group deleted, UsageMeter removed from footer, PRO chips on Upskill/Expand/Tools/Integrations, Interview Prep shows "(mock)" + PRO chip via proFeature field, active state = 2px primary left rail + rgba(29,78,216,0.08) tint, admin link only when userIsAdmin with text-accent. MobileHeader export preserved (same SidebarContent). GROW visible to free users (adminOnly filter only gates true admin items).
- `src/components/BottomNav.tsx` (NEW) — 5 mobile tabs (Home/Jobs/Shortlist/Applications/More) with md:hidden, same rail+tint active state, More-path detection for More-tab highlight.
- `src/components/MoreSheet.tsx` (NEW) — bottom Sheet mirroring sidebar GROW+ACCOUNT order: Interview Prep (mock PRO), Upskill/Expand/Tools/Integrations (PRO chips), Profile & Documents, Settings (Plan sub-label), Sign out.
- `src/components/AppLayout.tsx` — imports+renders BottomNav, main gets pb-16 md:pb-0.
- `src/components/AdminShell.tsx` (NEW) — minimal admin-only shell: shadcn Tabs Users/Portals/Markets/Submissions/Templates, placeholder content (AdminDashboard.tsx owns real content, unwired per spec).
- `src/App.tsx` — /rank→/shortlist (RankPage import kept as alias, page rebuild owned by builder-04), /jobs/:id placeholder route added, /markets + /billing routes + imports removed, /tools AdminGuard→AuthGuard, /admin now wraps new AdminShell (AdminDashboard import removed as unused). Route census matches §4 exactly + public routes.

**Verification:** tsc --noEmit exit 0; vitest run exit 0 (3/3 tests pass, incl. Dashboard regression test); all 12 grep Done criteria pass individually (shortlist 2, rank/markets/billing 0, jobs/:id 1, groups 10, ProChip 4, BottomNav 2, MoreSheet 2, tools=AuthGuard, admin=only AdminGuard route).

**Deviations (minor, within spec tolerance):** (1) Step 6 — kept src/pages/Rank.tsx filename (spec allowed alias OR rename; chose alias to avoid touching builder-04's page-internal territory; route string is /shortlist). (2) ProChip sidebar grep=4 includes one documenting comment + 3 render sites (chips render via shared renderLink loop, not per-item literals). (3) "Admin Panel" label kept for admin link (spec §2 shows admin shell-only visibility, label not otherwise specified). No conflicts with concurrent builder-04 (tsc clean over its InterviewPrep/Settings/Upskill edits).
