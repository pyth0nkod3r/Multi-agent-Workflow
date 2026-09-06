# CRITIC REVIEW — fastapi-backend (run 20260902-0628)

**Critic:** blind code review, judgment-only (tests NOT re-run — 59/59 pytest + 46 OpenAPI paths given as verified).
**Scope read:** docs/openapi.yaml, BACKEND-NOTES.md §1–11, tasks/00-CONTRACT.md, all 15 routers, deps.py, db.py, schemas.py, all 3 test files.

## Verdict: **CONDITIONAL PASS** — 4 of 4 builders did honest, well-scoped work; the RBAC map is 14/15 correct and the per-user scoping is clean. But there is 1 spec-vs-contract admin gap (GET /tools/templates), 1 grounding violation in the follow-up draft template, a days_quiet derivation that ignores dated notes, a global (cross-tenant) template/portal store, and 4 of 15 routers have ZERO tests — which is exactly why the admin gap went unnoticed. Fixes are small; none require a rebuild.

---

## Dimension 1 — Admin enforcement

Admin set per CONTRACT = 6 spec `x-admin-only` ops + /markets activate+deactivate + /automation config PATCH + test-connection + all 5 /integrations/* = **15 endpoints**. Checked each router file:

| Endpoint | Dependency | OK? |
|---|---|---|
| PATCH /portals/{portalId} | require_admin (portals.py:46) | ✅ |
| POST /jobs/scrape | require_admin (jobs.py:210) | ✅ |
| GET /jobs/portal-health | require_admin (jobs.py:246) | ✅ |
| POST /tools/robots-check | require_admin (tools.py:28) | ✅ |
| POST /tools/portals | require_admin (tools.py:51) | ✅ |
| **GET /tools/templates** | **get_current_user (tools.py:71-75)** | ❌ |
| POST /tools/templates | require_admin (tools.py:81) | ✅ |
| POST /tools/templates/{id}/activate | require_admin (tools.py:101) | ✅ |
| POST /markets/{code}/activate | require_admin (markets.py:52) | ✅ |
| POST /markets/{code}/deactivate | require_admin (markets.py:66) | ✅ |
| PATCH /automation/config | require_admin (automation.py:46) | ✅ |
| POST /automation/test-connection | require_admin (automation.py:56) | ✅ |
| POST /integrations/gmail/run | require_admin (integrations.py:62) | ✅ |
| POST /integrations/gmail/classifications/{id}/apply | require_admin (integrations.py:82) | ✅ |
| GET/POST /integrations/notion/{state,connect,push} | require_admin (integrations.py:105,114,128) | ✅ |

**Issue I-1 (medium).** GET /tools/templates is `x-admin-only: true` in docs/openapi.yaml (op starting line 934; marker at ~937) but uses `get_current_user` — any free user can list it. Root cause is a CONTRACT error: 00-CONTRACT.md's admin list names only "…/tools/templates POST, /tools/templates/{id}/activate" and omits GET. The builder followed the contract and the tools.py docstring even asserts "(all users)" — a wrong claim normalized into documentation. The spec is the contract; fix is one dependency swap. (The OpenAPI marker set is exactly 6 ops; the CONTRACT's "ADDITIONALLY" clause is the only source for the other 9 — that part was implemented faithfully.)

Cosmetic: automation.py docstring says "the spec marks it x-admin-only" for PATCH /automation/config — false; that requirement comes from the CONTRACT's additional list, not the spec. Also RankRequest.scope ("new"|"all") is accepted but ignored in rank.py — the pool is the same either way. Mock-acceptable, worth a note.

## Dimension 2 — Domain invariants (BACKEND-NOTES §1–11)

**PASS — deadline never inferred.** `deadline` appears only in seed data (db.py:236,247,258) and is explicitly `None` in create_application (applications.py:117). Scrape mock jobs, rank, status transitions — nothing derives a deadline. ✅

**PASS with deviations — follow-up candidates (§6/§11).** applications.py:181-197 excludes drafted + final (184-185), requires `< 2` follow-ups (190) and `≥ 10` days quiet (192-193). Core gate correct. Deviations:

- **Issue I-2 (medium): days_quiet ignores dated notes.** Spec: "days_quiet (from date or last dated note…)". `_days_quiet` (applications.py:63-76) reads only `lastContactAt`/`appliedAt`, and `lastContactAt` is stamped only on FINAL transitions (applications.py:151-152, record_outcome 171-172). A row that received an `interview_invite` outcome with a dated note yesterday still counts quiet-time from `appliedAt` months ago → false follow-up candidates. The notesLog entries carry `at` timestamps that are never read back.
- **Issue I-3 (medium): follow-up-draft has no eligibility gate.** POST /applications/{id}/follow-up-draft (applications.py:200) drafts for ANY tracked row — drafted rows, final rows, rows with ≥2 follow-ups. The §6 invariants live only in the listing endpoint. One guard clause needed.
- **Issue I-4 (high, for this domain): fabricated claim in the draft template.** applications.py:216-218 hardcodes *"In my current work I own an end-to-end production pipeline, from ingestion through deployment and monitoring…"* — asserted for every user, grounded in nothing (not the profile, not archived materials). §6: drafts are "grounded ONLY in archived materials — no new claims". This is precisely the class of claim the Factual Grounding Audit exists to strip. The word count (60-120) and channel shape are honored (~84 words; test asserts both), but the content invariant is violated. Same-pattern boilerplate exists in fit.py's cover-letter (fit.py:118-121, "I run production systems end to end") — mock posture, but the reviewer notes in review_drafts literally flag "claims experience; ground it" while the generator fabricates.
- Minor: `followUpsSent` is never incremented anywhere in the codebase (seed values only) — the `< 2` cap can never trip through API usage.

**PARTIAL — outcome notes idempotent-append (§11).** record_outcome (applications.py:157-177) is append-only (`setdefault("notesLog", []).append`, comment: "never overwrite prior entries") and re-posting an event does not re-stamp first-transition timestamps — that part is idempotent and tested. But posting the identical event+notes twice appends a duplicate dated note; there is no dedup/idempotency key. Judgment: append-only semantics honored, idempotency not. Minor.

**PARTIAL — rank exclusions (§11).** rank.py:41-50 excludes tracker rows (`jobId` ∈ applications) and `poor_fit`, both tested with pinned ids (test_rank_default_scope_exclusions_and_ordering). BACKEND-NOTES §11 additionally says rank "excludes tracker rows + **gate failures**" — eligibility/language **FAIL** jobs would still be ranked (only `poor_fit` is checked; no seed job has a FAIL gate, so the gap is invisible to tests). Minor-to-medium; notes-vs-implementation drift.
- **Issue I-5 (medium): strengths/gaps are synthesized, not verbatim.** §11: rank "returns verbatim strengths/gaps per job (**never backfilled**)". `_strengths`/`_gaps` (rank.py:18-34, duplicated in fit.py:42-58) emit the same canned string *"End-to-end production ownership (Pipeline + cube-os agent)"* for every job — CLI-era artifact text, identical across all tenants and jobs, referencing nothing in the posting. Violates the verbatim/no-backfill rule in spirit and in letter (it IS backfill).

**Legacy status spellings (§6):** "accept space spellings on read, never write" — writes are safe (ApplicationStatus Literal, schemas.py:140-144, never writes spaces ✅), but reads don't normalize: GET /applications?status="offer declined" matches nothing (applications.py:86-87 exact ==). Minor.

**Correct elsewhere:** market activation is additive and per-user (markets.py:49-73, tested); portal toggle writes the per-user override store, not the global registry (portals.py:58-60 — good, despite its own misleading docstring); setup reset only flips named sections; profile write-back appends with source attribution and defaults to "[chat]" (§5 Standing Rule shape ✅); ACTIVE-TEMPLATE invariant "exactly one active per kind" enforced in tools.py:104-110 ✅; 429 → "inconclusive, never breakage" honored in jobs.py:198-205 and asserted by test ✅.

## Dimension 3 — Multi-tenancy

**PASS on the hot paths.** Every per-user store access in all 15 routers keys off `user["id"]` obtained from `get_current_user`/`require_admin` — jobs, applications, rank pool, fit lookups (404 on foreign job ids), profiles, setup sections, expand runs, gmail runs, notion state, automation config, scrape configs, portal overrides, user_markets, dashboard stats. `_get_row`/`_get_job` all filter by user_id first. No endpoint accepts a user_id parameter. Registration hard-pins role="free" (auth.py:41-44, no self-promotion). PATCH /me re-keys credentials correctly. No cross-user read/write path found on per-user data. ✅

**Issue I-6 (medium): template/portal registries are global, not per-user.** db.py:43-46 declares `templates`, `custom_templates`, `registered_portals` as process-wide lists. Consequences:
1. GET /tools/templates (already missing its admin gate, I-1) returns templates registered by ANY tenant's admin to ANY authenticated user — spec summary says "the user's CV / cover-letter templates".
2. POST /tools/templates appends to the shared list (tools.py:94) — one tenant's template is every tenant's.
3. POST /tools/templates/{id}/activate flips global `active` flags (tools.py:104-110) — one tenant's admin changes the compile-step override for ALL tenants.
4. POST /tools/portals appends to shared `registered_portals` (tools.py:66), visible in every user's GET /portals — BACKEND-NOTES §11 says portal registration "appends to **the user's** registry".

db.py's own comments admit "global registry (admin-managed)" — a deliberate mock simplification, but it contradicts the spec's per-user wording and compounds I-1. Must be re-keyed per-user when MockDB is swapped for Postgres; flag it in BACKEND-NOTES so it doesn't get copied forward.

## Dimension 4 — Test honesty

**Strong where it exists:**
- RBAC 403s asserted for 4 admin endpoints: scrape (test_jobs_portals_markets.py::test_scrape_forbidden_for_non_admin), portal-health, portal toggle, markets activate. 401s and 404s asserted too, not just 200s.
- Domain behavior genuinely tested, not hardcoded echoes: follow-up candidates recomputes the days_quiet formula in the test and asserts drafted-row exclusion (a3) regardless of dates; rank asserts the full exclusion set (tracker 1/2/3/5, poor_fit 7) with pinned reasons and ordering; outcome test asserts first-transition-only `interviewAt` + append-order of notesLog; markets test asserts the additive posture explicitly; portal-health test asserts 429→"inconclusive" with the exact "not evidence of breakage" detail.

**Gaps — this is where the 59/59 flatters the suite:**
- **Issue I-7 (high): 4 of 15 routers have zero tests.** tools.py, integrations.py, automation.py, expand.py — no test file touches them (the three files cover auth/profile/setup, fit/applications/rank, jobs/portals/markets/dashboard only). That means: no 403 assertions for /automation PATCH + test-connection, all 5 /integrations/*, and 4 /tools/* endpoints; no tests for gmail-run's tracker-write behavior (which mutates application rows by company match — the riskiest untested logic in the codebase); no tests for expand apply; and I-1 was guaranteed to survive the suite. 59 passing tests over 11 of 15 routers is not "the backend is verified", it's "the tested part is verified".
- **Issue I-8 (medium): no cross-tenant isolation test anywhere.** Nothing registers a second user and asserts they cannot see user A's jobs/applications/profile (e.g. POST /v1/jobs/{otherUsersJobId}/evaluate-fit → expect 404). The scoping is correct by reading, but the one invariant the spec calls out in bold ("users never see another user's jobs… No cross-user endpoints, ever") is untested.
- **Issue I-9 (low-medium): follow-up candidate exclusions under-asserted.** The candidates test checks a2 (present) and a3 (drafted, absent) but never asserts a4 (followUpsSent=2) or a5 (final/rejected) are excluded — the `< 2` and final-status branches of applications.py:184-190 are untested. Likewise no test that follow-up-draft refuses ineligible rows (it doesn't — I-3), and no grounding assertion on the draft body beyond word count (I-4 thus untested by construction).
- State hygiene note: MockDB is a module-level singleton; suites manage order (read-only first, mutations last, admin tests restore shared state). Works, but it's order-coupled — a future test added in the wrong file will silently depend on mutation order. Worth an autouse fixture that snapshots/restores db state.

---

## Final Issues (actionable)

| # | Severity | Where | What |
|---|---|---|---|
| I-1 | medium | tools.py:71-75 vs openapi.yaml GET /tools/templates (~:937) | Missing require_admin on an x-admin-only op; contract (00-CONTRACT.md) omitted GET from its admin list — fix code + contract |
| I-2 | medium | applications.py:63-76, 151-152, 171-172 | days_quiet ignores dated notesLog entries; lastContactAt stamped on final transitions only → stale quiet-time |
| I-3 | medium | applications.py:200-231 | follow-up-draft drafts for drafted/final/≥2-followup rows — no candidate-gate re-check |
| I-4 | high | applications.py:216-218 (also fit.py:118-121) | Hardcoded capability claim in follow-up draft, grounded in nothing — violates §6 "no new claims / archived materials only" |
| I-5 | medium | rank.py:18-34, fit.py:42-58 | Canned identical "strengths" ("Pipeline + cube-os agent") for every job — violates §11 verbatim/never-backfilled |
| I-6 | medium | db.py:43-46, tools.py:66,94,104-110 | Template + registered-portal registries are process-global; cross-tenant visibility and mutation (spec says per-user) |
| I-7 | high | tests/ | Zero coverage for tools, integrations, automation, expand routers — no RBAC 403s, no gmail tracker-write test; this is why I-1 survived |
| I-8 | medium | tests/ | No cross-tenant isolation test (user B fetching user A's job/application by id) |
| I-9 | low | tests/test_fit_applications_rank.py | Candidates test never asserts a4 (≥2 follow-ups) / a5 (final) exclusions |
| — | low | applications.py:86-87 | Legacy space-spelling statuses not normalized on read (writes correctly never produce them) |
| — | low | applications.py (all) | followUpsSent never incremented by any endpoint — cap unreachable in practice |
| — | low | rank.py:67-70 | RankRequest.scope accepted but ignored; gate-FAIL exclusion (§11) not implemented |
| — | cosmetic | automation.py docstring; tools.py docstring | Docstrings misattribute admin markers to the spec / assert "(all users)" for a spec-admin op |

**Required before merge:** I-1, I-3, I-4 (one-line to few-line fixes). **Strongly recommended:** I-7 (a tools/integrations/automation test file with the 11 missing 403s), I-2, I-5, I-6 flagged into BACKEND-NOTES as known mock debt. Everything else can ride.
