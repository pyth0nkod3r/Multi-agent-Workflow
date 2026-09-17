# SPEC U1 — EaseApply functionality inventory (run 20260911-2150-ui-redesign-research)

**Executor profile**: researcher. **Output owner**: this file's `## Result` (fill LAST, after disk verify).

## Context (you cannot see the parent conversation)
EaseApply (repo `/workspace/platform`) is a job-application platform: FastAPI backend (`backend/app/`, routers in `backend/app/routers/`, OpenAPI at `docs/openapi.yaml`) + React/TS frontend (`frontend/src/`, routes in `App.tsx`, 19 pages). Roles: free / pro / admin; admin is an operator with ZERO job-seeking surface (enforced server-side). We are preparing a full UI redesign; the redesign's navigation must map 1:1 to REAL backend functionality. Your unit produces the authoritative inventory that everything else builds on. Research/planning ONLY — do NOT edit any code.

## Read (absolute paths)
- /workspace/multiagent/tasks/20260911-2150-ui-redesign-research/GOAL.md (goal + recon)
- /workspace/platform/docs/openapi.yaml (38 paths, 53 ops, 15 tags)
- /workspace/platform/backend/app/routers/*.py — all 16: applications, auth, automation, dashboard, expand, fit, integrations, jobs, markets, portals, profile, rank, setup, submissions, tools, users. For each: list every route (method + path), its purpose, auth/role gating, and whether the underlying work is REAL or TEMPLATE/MOCK today.
- /workspace/platform/ROLE-MODEL-PLAN.md — §2 capability matrix (free/pro/admin per capability) is the role-gating authority.
- /workspace/platform/BACKEND-NOTES.md (§12 mock debt) — mock/real classification source.
- /workspace/platform/frontend/src/App.tsx + frontend/src/pages/*.tsx — for each of the 19 pages: which endpoints it actually calls, and classify page = real / partial / mock.
- /workspace/platform/CLAUDE.md (product conventions).

Known backend reality (verify, don't assume): LLM generation is TEMPLATE-based today (no real LLM); PDF compile pending (no TeX on Render); jobs free-text search pending; OTP auth + billing PAYMENT pending (billing page/plans exist); arbeitnow portal scraping real, other portals mockish; dashboard stats real; demo auth seeded (user@demo.com/user123, pro@demo.com/pro123, admin@demo.com/admin123).

## Output file (only this path; nothing else)
`/workspace/platform/docs/design/01-functionality-inventory.md` — structure:
1. **Endpoint inventory** — table grouped by router/tag: Method | Path | Purpose (1 line) | Role gating (free/pro/admin/any) | Status (REAL / PARTIAL / TEMPLATE / MOCK) | Notes.
2. **Frontend page audit** — table: Page (route) | Endpoints called | Classification | Gaps vs backend.
3. **Feature → menu mapping proposal** — table: Menu item | Route | Backing endpoint(s) | Role visibility | Verdict (KEEP / MERGE / DEFER / REMOVE). Every KEEP must name ≥1 real endpoint. Items with no real backing go to DEFER/REMOVE with reason.
4. **Deferred list** — features the backend cannot support yet (payments, OTP, PDF export, real LLM scoring, …), each with what would be needed to enable it later.
5. **Admin surface** — separate section: admin-only menu, restating the operator-not-seeker rule.

## Constraints
- Menu↔backend 1:1 is the ANCHOR constraint — the redesign nav may only contain what you confirm exists.
- Do not invent features. Do not propose UI; this is a functionality inventory only.
- Checkpoint to disk after each numbered section (write the file incrementally, never hold it all in memory).

## Result
(filled by executor LAST, only after `ls` confirms the output file exists and is complete; 3-6 lines: file path, line count, counts of endpoints/pages/menu items, anything you could NOT verify)
