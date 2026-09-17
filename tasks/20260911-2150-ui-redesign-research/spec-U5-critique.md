# SPEC U5 — Critic gate on design research artifacts (run 20260911-2150-ui-redesign-research)

**Executor profile**: critic. **Budget**: ≤10 commands, 600s. **Output**: this file's `## Result` + the verdict file.

## GATE
All five artifacts must exist on disk first:
- /workspace/platform/docs/design/01-functionality-inventory.md
- /workspace/platform/docs/design/02-applydeck-patterns.md
- /workspace/platform/docs/design/03-wireflows/ (≥5 PNGs + INDEX.md)
- /workspace/platform/docs/design/04-direction-briefs.md
- /workspace/platform/docs/design/05-ia-menu-spec.md

## Your job
Bounded verification pass on the run's artifacts. You have no vision — do NOT try to view PNGs; verify them by existence + size + INDEX.md notes only. All other checks are textual (grep/read/math).

## Checks (do each; record pass/fail + evidence)
1. **Menu 1:1 vs backend** — extract every menu/route row from 05-ia-menu-spec.md and grep each backing endpoint against /workspace/platform/docs/openapi.yaml and the router files under /workspace/platform/backend/app/routers/. Any nav item without a real endpoint = FAIL with item name. Spot-check ≥5 randomly chosen menu items deeply (method+path exist in code).
2. **Applydeck leakage** — grep 05 + 03-wireflows/INDEX.md + 04 for: extension/autofill, resume builder/documents, mock-interview TOOLING (vs our interview-prep endpoints), "100,000", "100k+" claims. Our nav must not contain them (02's SKIP list is the reference).
3. **Brief template compliance** — for each direction in 04: exactly 1 primary + 1 accent + ≤2 neutrals; 1 display + 1 body font; vibrancy mechanism present; shape/motion signature present; Stitch-ready prompt block present; ≥2 directions offered; Decision section NOT pre-filled (user picks).
4. **Contrast math** — recompute WCAG ratios for each direction's palette (relative luminance formula) from the hexes; confirm body-text pairs ≥4.5:1 and large/UI pairs ≥3:1. Show YOUR numbers, flag any that differ from the artifact's claim.
5. **Desktop AND mobile coverage** — 05 has both navs; 03 has the mobile sheet (06) + desktop sheets (01-05); wireflows cover all key flows listed in GOAL.md (landing→signup, dashboard, jobs+rank, apply w/ checkpoint, tracker, interview prep, profile/settings, admin).
6. **Scope guard** — `cd /workspace/platform && git status --porcelain` must show ONLY docs/design/* additions (plus nothing in frontend/src or backend/). Implementation before user approval = automatic FAIL.
7. **Coherence** — 04's recommended direction rationale cites 02's verdicts; 05's nav labels match the wireflow screen labels in INDEX.md; 01's DEFER/REMOVE list matches 05's excluded list.

## Verdict format
`/workspace/multiagent/tasks/20260911-2150-ui-redesign-research/critique-result.md` — per check: PASS/FAIL + 1-2 line evidence; overall **Verdict N/10** (≥8 passes the gate); Gaps (numbered, actionable, each with owner-unit); Recommendation to orchestrator. Use the structure of /workspace/staging-ai-design/templates/qa-critique.template.md where applicable.

## Result
**Verdict 8.5/10 — PASSES the gate (≥8).** Written to /workspace/multiagent/tasks/20260911-2150-ui-redesign-research/critique-result.md (disk-verified). Checks: 1-6 PASS (menu 1:1 verified in router code incl. 5 decorator-level spot-checks; zero applydeck leakage in nav; template compliance; 19/19 contrast ratios recomputed exact, all ≥4.5:1; desktop+mobile coverage with all 6 PNGs; git scope clean — only `?? docs/design/`), check 7 coherence FAIL-partial. Top gaps: (1) no Shortlist/rank wireframe screen; (2) desktop wireframe sidebar labels drift from 05 §2 (missing Integrations/Profile); (3) INDEX.md lacks 06 DONE line + screen labels. Recommendation: proceed to user approval gate; batch gaps 1-3 as one designer polish unit; surface interview-prep deferral (GOAL contradiction resolved correctly) in the user summary.
