# GOAL — EaseApply UI redesign: research, plan & direction (run 20260911-2150-ui-redesign-research)

## User request (verbatim intent, 11 Sept 2026)
> "I like the UI design of useapplydeck.com. Can we redesign EaseApply to follow similar pattern or even a better one? However, our UI menu should match the functionalities on our backend. Research, plan and come up with a better UI for both desktop and mobile - which will be used for the redesigning. Don't start the redesign activity yet."

## Deliverables (all under /workspace/platform/docs/design/)
1. **01-functionality-inventory.md** — Complete inventory of EaseApply's real backend functionality (routers → endpoints → purpose → role gating), every frontend page classified real/partial/mock, and the authoritative feature→menu mapping. ANCHOR CONSTRAINT: every nav/menu item must map to a real backend capability; anything without one is explicitly listed as remove-or-defer.
2. **02-applydeck-patterns.md** — UI/UX pattern teardown of useapplydeck.com: IA, navigation model, page layout patterns, component patterns, copy voice, responsive behavior. Explicit "adopt / adapt / skip" verdicts. Features applydeck has that we DON'T (Chrome extension, resume builder, mock-interview tooling) must NOT leak into our nav.
3. **03-wireflows/** — pen-generated flow wireframes (grayscale, no color decisions) for desktop AND mobile covering key flows: landing→signup/login, dashboard, jobs+rank, application tracker, auto-apply workflow (with human checkpoint before submit), interview prep, profile/settings, admin console.
4. **04-direction-briefs.md** — 2–3 design direction briefs per direction-brief.template.md (one primary + one accent + two neutrals max, one display font + one body font, explicit vibrancy rules), with a recommended direction + rationale. EaseApply live token fallbacks available if needed: primary #3399ff, accent #17cfa1, bg #0b0d13, card #12151c, text #e7ebef, destructive #df3a3a, warning #f6a823, success #22c373; fonts Outfit/JetBrains Mono.
5. **05-ia-menu-spec.md** — Final information architecture: sitemap, desktop nav + mobile nav (bottom tab / drawer) mapped 1:1 to backend functionality, route names, role visibility (user / pro / admin), empty/deferred features explicitly excluded.
6. **User-facing summary** — wireflow PNGs shown via show_image, user picks a direction and approves flow. RUN ENDS at this approval gate.

## Hard constraints
- **STOP before implementation**: no edits to frontend/src or backend, no theme/token changes, no Stitch design-system asset creation. pen wireflows (flow-level, grayscale) and written briefs ARE in scope — they are proposals, not implementation. Stitch generation happens AFTER the user picks a direction (Phase 2 proper).
- **Menu ↔ backend 1:1** — no nav item without a real endpoint behind it.
- **Desktop AND mobile** — both must be fully specified.
- Current EaseApply UI is explicitly REJECTED as a design reference (CLAUDE.md design rule) — do not copy or extend it.
- Design artifacts live in the project repo (docs/design/). The ai-design-workflow skill + /workspace/staging-ai-design stay project-agnostic — no EaseApply/FabricFocus names may be added there.
- Known backend reality (memory, 11 Sept): real LLM generation = templates today; scraping partial (arbeitnow real, YAML registry queued); PDF compile pending; jobs free-text search queued; OTP auth + billing pending; roles user/pro/admin with seeded demo users; admin has AdminDashboard w/ users tab.

## Recon already done (orchestrator, turn 1)
- **applydeck landing copy** (useapplydeck.com): "Your whole job search, in one deck." Positioning: find roles worth your time → tailor resume → score before apply → track every step to offer. Sections: "Roles that actually fit you, not keyword noise" (100k+ live listings from career sites, ranked by fit), "Score your resume before you hit apply" (tailor + graded on keyword coverage/gaps/results), "Applications that fill themselves" (Chrome extension drops details + tailored resume + cover letter into any form, 28 fields, confirms when done), "Every application, on one board" (wishlist→applied→interview→offer board with resume+notes per card), "Find out which topic you're worst at" (10-min scored mock interview built from the posting). Stats: 100k+ roles, 150+ countries, 28 fields, 10-min interview. FAQ: sources = career sites/ATS ("real pipeline not reposts"), free to start (paid = higher limits + heavier tailoring), no fabrication (rewrites/reorders, you see every line), data export/delete + no-training-without-opt-in pledge.
- **applydeck nav / app routes** (link map): public nav = Job search, Mock interviews, Extension, Blog; app = /jobs (matching), /documents?tab=resumes (resume builder), /trackers (application tracker board), /tools/mock-interview, /signup; CTAs "Start your job search" / "See how it works"; footer = About/Careers/Blog/Contact/Privacy/Terms/Security.
- **Platform layout confirmed**: backend/app/{routers/, db.py, db_pg.py, deps.py, main.py, pg_core.py, pg_mixins/, schemas.py}; frontend/src/pages/{AdminDashboard, Applications, Billing, Dashboard, Expand, Index, Integrations, InterviewPrep, Jobs, Landing, Login, Markets, NotFound, Profile, Rank, Settings, Setup, Signup, Tools, Upskill}; docs/{openapi.yaml, RESEARCH-*.md incl RESEARCH-applydeck.md (business intel, done), ROLE-MODEL-PLAN.md, POSTGRES-ADOPTION-PLAN.md}.
- **Business intel already on file** (do not redo): docs/RESEARCH-applydeck.md + memory — applydeck = freemium global resume co-pilot, Chrome ext, 100k+ career-site roles, no public pricing; adoption shortlist T1/T2/T3 in docs/RESEARCH-autoapplyai.md.

## Unit sketch (planner refines into specs; disjoint file ownership)
- **U1 (researcher)** → 01-functionality-inventory.md. Read backend/app/routers/*, docs/openapi.yaml, docs/ROLE-MODEL-PLAN.md, backend/BACKEND-NOTES.md if present, frontend/src/pages/* + App.tsx routes, CLAUDE.md, memory-known gaps. Output = feature truth + menu mapping table.
- **U2 (researcher)** → 02-applydeck-patterns.md. Fetch useapplydeck.com interior pages (/jobs, /trackers, /documents, /mock-interviews, /extension, /signup) + landing. UI patterns only (business intel is done). Adopt/adapt/skip verdicts per pattern. Note responsive/mobile patterns explicitly.
- **U3 (designer)** → 03-wireflows/*.png + *.pen. GATED on U1+U2 outputs on disk. pen CLI: `pen --out X.pen --agent codex --model gpt-5.5 --prompt "..." --export X.png` (workspace_shell; PEN_AGENT_API_KEY must be UNSET; grayscale/flow focus, no colors).
- **U4 (designer)** → 04-direction-briefs.md + 05-ia-menu-spec.md. GATED on U1+U2 (briefs draw on patterns; IA on inventory). 2–3 directions per template + recommended pick + full menu spec desktop+mobile with role visibility.
- **U5 (critic)** → verdict on all 5 artifacts: menu-1:1-vs-backend check (grep endpoints vs menu items), template compliance (direction-brief template rules, WCAG contrast math on proposed palettes), desktop+mobile coverage, no-implementation-scope breach, applydeck-feature leakage check (no extension/resume-builder in our nav). Verdict ≥ 8/10 to pass.
- Merger: only if artifacts conflict (planner judges; likely unnecessary — U4 integrates).

## Ops rules (roster v2/v3)
- Researchers/designers/critic: checkpoint to disk after EVERY step; RESULT-LAST — fill spec ## Result only after verifying artifact on disk; never claim what isn't written.
- Timeouts: designers/builders 900s, critic 600s. Resume-by-default: re-dispatch same profile with resume spec up to 2 attempts before inline fallback (orchestrator fills ## Result itself after disk verification when a unit dies pre-writeback).
- Verify disk after every report (SUCCEEDED ≠ done).
- No parallel writes to shared files; specs own disjoint paths.
