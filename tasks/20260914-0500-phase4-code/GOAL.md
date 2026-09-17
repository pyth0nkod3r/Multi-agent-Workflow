# GOAL — EaseApply Phase 4: Clear Deck code migration (run 20260914-0500-phase4-code)

## User request (14 Sept 2026)
> "Kick off with the next item. Do the replan first."
> Next item = Phase 4 (code) from the approved UI redesign. Phases 1-3 (research, DESIGN.md
> contract, all 11 Stitch screens confirmed) are COMPLETE. The 12-Sept planner dispatch for
> phase4-code died pre-writeback (empty tasks/20260912-1436-phase4-code/ dir) — this run replaces it.

## What Phase 4 IS
Rebuild the frontend so it implements the approved design + nav contract:
- /workspace/platform/DESIGN.md — "Clear Deck" token contract (single source of truth; supersedes
  the REJECTED old dark tokens; sRGB hex; {group.name} references; zero hard-coded hex in components)
- /workspace/platform/docs/design/05-ia-menu-spec.md (R2) — sitemap, sidebar/bottom-tab nav, the
  route↔endpoint map (19 routers / 71 routes, grep-audited 12 Sept), role visibility, GROW-group rules
- 11 confirmed Stitch screens (registry: docs/design/06-stitch-registry.md) — visual reference
  (IDs listed there; HTML exports on Stitch CDN)

## Scope of code change
1. TOKEN MIGRATION: index.css/tailwind tokens → Clear Deck (light theme). Outfit + JetBrains Mono
   (already in use — keep). Radii sm4/md6/lg8. WCAG-verified values from DESIGN.md, NOT eye-balled.
2. NAV + IA RESTRUCTURE: AppLayout/AppSidebar/bottom tabs per §2/§3 of 05-ia-menu-spec —
   GROW group visible to free (PRO chips), /shortlist route, admin island, user chip + plan badge
   + sign-out, active-state left rail.
3. ENDPOINT WIRING: wire frontend pages to the real backend per the §4 route map. client.ts has ~49
   apiEnabled-gated live methods + mock fallback; 4 pages import mock data directly (InterviewPrep,
   Settings, Upskill + Login legacy) — migrate them to the client; add missing client methods for
   the newer endpoints (interviews/upskill/documents/plans/company-research/q-search/closingSoon/
   follow-ups/salary-benchmark/verify-compile/sentinel-test...).
4. NEW/CHANGED SURFACES per 05 §5b: /jobs/:id company-research panel + closingSoon badges,
   Dashboard + Shortlist closingSoon watchlists, /profile?tab=documents (upload, 10MB/50MB caps),
   /settings?tab=plan (GET /plans contract), /interview page, /upskill page on PRO-gate, /shortlist
   (Rank runs; split from /rank page if needed), Landing honest-stats only.
5. TESTS + BUILD: vitest stays green; new tests for token migration (index.css contains Clear Deck
   hex values, no old #3399ff-family HSL), nav contract (routes exist, GROW visibility by role),
   and every wire-up unit page. `npm run build` must pass.

## Hard constraints
- EVERY backend feature has menu + UI (user directive, standing).
- Zero hard-coded hex in components — tokens only.
- No coming-soon badges / dead tabs (hide-entirely policy, 05 §6).
- Differentiate from applydeck: no lifted copy, no cloned components — patterns only (02 register).
- Unit sizing ≤30 min worker time; checkpoint-or-die; RESULT-LAST.
- NO mock-erasure: live mode gated by VITE_API_BASE; demo mode must keep working (mock fallback).
- Do NOT touch backend code except reading it (endpoints already verified). Backend gaps (real LLM
  gen, real scraping, Render TeX) are NOT Phase 4 scope.
- Every task file is SELF-CONTAINED: a sub-agent reads only that file.

## Done criteria (whole run)
- Frontend builds + all tests green.
- Nav matches 05 §2/§3 exactly (grep-able: every route in the map exists in the router).
- Token audit passes: index.css/tailwind carry Clear Deck values; zero old-palette residue.
- Every route in 05 §4 route map is wired to its listed endpoints (client.ts method calls, no
  direct mock-data imports in pages).
- Commit series pushed to pyth0nkod3r/easeapply main.
