# Critique Result — run 20260911-2150-ui-redesign-research (U5)

**Critic**: spec U5, blind pass. **Date**: 11 Sept 2026. **Method**: textual/grep/math verification only (no vision — PNGs verified by existence + size + generation prompts, never viewed).

---

## Check 1 — Menu 1:1 vs backend: **PASS**
All 45 endpoint fragments cited in 05 §4 grep-hit real router files under `backend/app/routers/` (fit, jobs, applications, expand, tools, submissions, integrations, profile, setup, auth, automation, users, markets, portals, dashboard, rank). Deep spot-check of 5 randomly chosen items at decorator level, all verified in code: `fit.py:74 @router.post("/jobs/{jobId}/evaluate-fit")` · `applications.py:218 @router.post("/applications/{id}/follow-up-sent")` · `integrations.py:126 @router.post("/integrations/notion/push")` · `users.py:49 @router.get("/users")` · `markets.py:54 @router.post("/markets/{code}/activate")` (+ `markets.py:71 deactivate`). Zero nav items without endpoints — the "code-verified" claim in 05 §4 holds.
Note: openapi.yaml is missing `/applications/{id}/follow-up-sent`, `/auth/change-password` (both disclosed in 01 as yaml-stale) AND `/submissions/{id}/approve` (NOT disclosed in 01's stale list) — yaml regeneration backlog item.

## Check 2 — Applydeck leakage: **PASS**
Grep of 05 + 04 + 03/INDEX.md for extension/autofill, resume builder/documents, mock-interview tooling, voice/transcript, 100,000/100k/150+/28-field: every hit sits in 05 §5 "Explicitly EXCLUDED" (the guard list itself, which is required to name them, each cross-referenced to 02 #15-18) or in 04's citation of the adopted trust-voice pattern (02 #10 — a copy-voice adoption, not a feature). Zero hits in 05 nav sections §1-§4 (`grep -icE` on §2-§4 = 0). Nav is clean per 02 §6's leakage guard list.

## Check 3 — Brief template compliance: **PASS**
All three directions (≥2 required): exactly 1 primary + 1 accent each (A #2c5cd6/#c2410c, B #3399ff/#17cfa1, C #0f4c46/#b45309); type pairing Outfit (display+body) + JetBrains Mono (numerals) — two families, within the 1-display+1-body constraint; vibrancy mechanism present in all three (A gradient hero + duotone stats; B mint numerals + CTA-only glow with an explicit numeral-only rule; C amber underline bar + progress ticks); shape/motion signature present in all three; Stitch-ready prompt block present in all three; Decision section NOT pre-filled ("Chosen: pending user"). Recommendation (B) is present as GOAL requires — not a unilateral pick.
Note: each direction names ink + bg + card = 3 neutral-role colors (template says "2 neutrals MAX"); treated as ink/bg neutrals + card surface, consistent across all three directions. Same treatment for white cards in A/C.

## Check 4 — Contrast math: **PASS**
Recomputed all 19 claimed pairs with the WCAG 2.1 relative-luminance formula (independently, in JS). Every computed ratio matches the artifact's claim to 2 decimals, and every pair clears 4.5:1 (worst: C accent #b45309 on #faf7f2 = 4.70; A accent #c2410c on #f6f8fb = 4.87). Direction A's "accent darkened to pass 4.5:1" claim is honest — the original applydeck-ish #e8590c would have failed. No discrepancies found.

## Check 5 — Desktop AND mobile coverage: **PASS**
05 specifies both navs (§2 desktop sidebar w/ role gating + active-state rule; §3 mobile 5-tab bar + More sheet). 03 has all six sheets on disk: 01-public-onboarding (396,956 B), 02-core-app (354,972 B), 03-tracker-applications (187,702 B), 04-secondary (456,020 B), 05-admin (395,371 B), 06-mobile (323,485 B — exists and is full-size; INDEX.md's last line is "START 06-mobile" with no DONE/png= line, an INDEX-logging gap only). Generation prompts (run-pen-suite.sh) confirm flow coverage: landing→signup/login/setup (01) ✓, dashboard + jobs + job detail + draft-review checkpoint (02) ✓, tracker board/detail/outcome/follow-ups (03) ✓, profile/settings/integrations/tools/expand (04) ✓, admin 5 tabs (05) ✓, mobile login/dashboard/jobs/detail/checkpoint/board/More sheet (06) ✓. Two coverage notes → Gaps 1 & 2.

## Check 6 — Scope guard: **PASS**
`cd /workspace/platform && git status --porcelain` → exactly one line: `?? docs/design/`. Nothing in frontend/src or backend/, no token/theme changes, no Stitch assets. Hard constraint "STOP before implementation" honored.

## Check 7 — Coherence: **FAIL (partial)**
- 01 ↔ 05: MATCH — 01 §3 DEFER (Interview Prep, Upskill), MERGE (Markets→Jobs/Sources, Billing→Settings/Plan), KEEP list maps 1:1 onto 05 §5's exclusion table and §4's route map.
- 05 ↔ wireflow labels: PARTIAL MISMATCH — INDEX.md contains no screen labels at all (build log only), so the check as specified is unverifiable from INDEX; cross-checked via run-pen-suite.sh prompts instead: mobile tab bar (Home/Jobs/Shortlist/Applications/More) and More sheet contents match 05 §3 exactly, and the admin sidebar (Users/Portals/Markets/Submissions/Templates) matches 05 §1 exactly — but the desktop wireframe sidebar (sheets 02 & 04) reads "Dashboard, Jobs, Shortlist, Applications, Expand, Tools, Settings", omitting Integrations and Profile and abbreviating "Tools & Templates" vs 05 §2's 9-item sidebar. Screens for both exist in sheet 04; it is chrome-label drift, not missing coverage.
- 04 ↔ 02: WEAK — the Recommendation (Direction B) does not explicitly cite 02's verdicts; the connection to 02 #10 ("It drafts, you decide") is implicit. The document body does cite 02 #10/#14 where relevant.

---

## Verdict: 8.5/10 — **PASSES the gate** (≥8)

## Gaps
1. **[warn, owner: designer, fix before Phase 2]** No wireframe screen anywhere shows the Shortlist/batch-rank flow (POST /rank — a KEEP menu item and a named GOAL flow "jobs+rank"). It appears only as a sidebar label. Add a shortlist screen (rank run + gated results) to 02-core-app or 04-secondary, or explicitly note its absence in the user summary so the approval gate covers it.
2. **[warn, owner: designer, quick fix]** Desktop wireframe sidebar labels drift from 05 §2 (missing Integrations, Profile; "Tools" vs "Tools & Templates"). Phase-2 code and Stitch prompts will read 05 as the nav contract — regenerate or annotate, so the user doesn't approve wireframes whose chrome contradicts the spec.
3. **[warn, owner: designer, trivial]** INDEX.md has no DONE/png= entry for 06-mobile and carries no per-sheet screen labels; append the DONE line + a one-line screen list per sheet so INDEX is usable as the label reference the coherence check assumes.
4. **[note, owner: orchestrator→backend backlog]** openapi.yaml is missing `/submissions/{id}/approve` in addition to the two stale paths 01 already documents (`/auth/change-password`, `/applications/{id}/follow-up-sent`); add to the yaml-regeneration backlog item in 01 §1 header note (yaml-regen backlog).
5. **[note, owner: orchestrator]** Interview prep appears in GOAL's wireflow flow list but is correctly absent from wireflows and nav (no backend, DEFER in 01/05). This is a GOAL-internal contradiction resolved the right way (anchor constraint wins); record the resolution so the user isn't surprised the flow is missing.
6. **[note, owner: designer]** 04's Recommendation should cite 02's pattern verdicts explicitly (e.g. "B stages 02 #10's adopted human-checkpoint pillar") — currently implicit.

## Recommendation to orchestrator
**Proceed to the user approval gate.** All hard constraints hold: menu↔backend 1:1 verified in code, zero applydeck leakage in nav, contrast math exact, scope untouched, desktop+mobile fully specified. Gaps 1-3 are pre-approval polish for the designer (one short work unit: add shortlist screen, fix sidebar labels, finish INDEX) and can be batched before or alongside showing the PNGs; none block the user picking a direction. Present 04's three directions (recommendation B noted) + wireflow PNGs + 05's nav contract to the user, and surface Gap 5's interview-prep deferral in the summary. No re-dispatch of U1-U4 needed.
