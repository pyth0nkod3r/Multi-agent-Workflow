# RUN.md — run 20260914-0500-phase4-code (EaseApply Phase 4: Clear Deck code migration)

Ledger per ORCHESTRATION §A3. Statuses: dispatched / done(disk-verified) / FAILED.

## Protocol amendment (user directive 14 Sept 2026)
- Try CONCURRENT fan-out dispatches FIRST; solo only as fallback after concurrent fails.
- Concurrent dispatches capped at 2 per session (wave); sequential sessions gated on
  success — next session runs only after the previous session's units are disk-verified.

## Session plan (7 specs over 2 sessions, max 2 concurrent each)
- SESSION 1: planner-05 (specs 01-builder-tokens.md + 02-builder-nav.md)
             planner-06 (spec 03-builder-client-methods.md)
- SESSION 2 (gated on S1 success): planner-07 (specs 04-builder-pages-grow.md + 05-builder-pages-core.md)
                                   planner-08 (specs 06-builder-pages-account.md + 07-builder-tests.md)

## Dispatch log
2026-09-14T04:5xZ · planner-01 · 32f35d66-181e-4d37-8f23-0a153d1c98b3 · FAILED — solo dispatch, fast-fail (SUCCEEDED status, 0 trips/0 tokens/~21s; nothing on disk)
2026-09-14T05:0xZ · planner-02/03/04 · b6542c77/bb2610a5/1dda15b5 · FAILED — concurrent wave of 3, all fast-fail (0 trips/0 tokens/~15s each; nothing on disk; verified via ls before this entry)
2026-09-14T05:2xZ · planner-05 · (pending) · DISPATCHED — SESSION 1 — owns 01-tokens + 02-nav
2026-09-14T05:2xZ · planner-06 · (pending) · DISPATCHED — SESSION 1 — owns 03-client-methods

## Session 1 — CLOSED 2026-09-14 ~04:12 (disk-verified)
2026-09-14T03:5xZ · planner-05 · c410f02f-1810-4785-98ad-33a9e33629a8 · done(disk-verified) — 01-builder-tokens.md (229 ln) + 02-builder-nav.md (173 ln) + status-planner-05.md
2026-09-14T04:0xZ · planner-06 · 5109f3dd-7ff6-409e-aed6-9389975dc80e · done(disk-verified) — 03-builder-client-methods.md (723 ln; 18 new methods + 2 extensions + 10 types + 7 mocks + 3 page migrations) + status-planner-06.md
GATE S1→S2: PASSED (all 3 specs present, sections complete, content spot-checked: method names, token values, grep criteria all real)

## Session 2 — DISPATCHED 2026-09-14 ~04:3xZ (gated on S1 success)
planner-07 · (pending) · DISPATCHED — owns 04-builder-pages-grow.md + 05-builder-pages-core.md
planner-08 · (pending) · DISPATCHED — owns 06-builder-pages-account.md + 07-builder-tests.md

## PROTOCOL AMENDMENT (user directive 14 Sept, post-S2): TRUE CONCURRENCY
Evidence: S1/S2 same-block foreground dispatches were serialized by the engine
(planner-06 started 38ms after planner-05 finished; planner-08 37ms after planner-07).
Fix: waves dispatch with run_in_background=true (both spin up immediately) + disk-polling;
cap stays 2 concurrent per wave; waves stay gated on disk verification.

## Session 2 — CLOSED 2026-09-14 ~04:26 (disk-verified)
planner-07 · 2d6ab118 · done(disk-verified) — 04-builder-pages-grow.md (147 ln) + 05-builder-pages-core.md (150 ln) + status-planner-07.md
planner-08 · 3311fbaf · done(disk-verified) — 06-builder-pages-account.md (159 ln) + 07-builder-tests.md (162 ln) + status-planner-08.md
GATE S2→BUILD: PASSED (all 7 specs present/complete; 04/05/06 reference exact 03 method names; dispatch order per planners: 03→01/02→04/05/06→07)

## BUILD WAVES (2 concurrent max, true-parallel via background mode, gated)
WAVE A: builder-01 (tokens) + builder-03 (client methods) — DISPATCHED ~05:0xZ
WAVE B (after A verified): builder-02 (nav) + builder-04 (pages-grow)
WAVE C (after B verified): builder-05 (pages-core) + builder-06 (pages-account)
WAVE D (after C verified): builder-07 (tests, solo capstone) → critic pass → commit series

## WAVE A — CLOSED 2026-09-14 ~06:2x (disk-verified)
2026-09-14T05:10Z · builder-01 · 43874e5a · done(disk-verified) — tokens COMPLETE 8/8 (self-Result + orchestrator grep re-verify: primary present, old bg/mint/0.75rem=0, radius vars 3, vitest+build green); TIMED_OUT at cap AFTER writeback
2026-09-14T05:10Z · builder-03 · cdd1498c · done(disk-verified) — client methods COMPLETE: 18 methods + 10 types + 9 mocks + 3 page migrations; TIMED_OUT mid-verification; retry1 (fe040d51, 600s) landed Upskill.tsx migration, died pre-Result; orchestrator verified (vitest 3/3 green) + filled spec ## Result per standing rule
WAVE A GATE: PASSED → Wave B cleared

## ORCHESTRATOR FIXES — 14 Sept ~12:5x (pre-Wave C)
- security.ts Permission union + 'integrations:manage' (pro+admin both listed it; union lacked it) → AppSidebar+security tsc errors cleared
- Deleted dead pages Markets.tsx + Billing.tsx (unrouted by builder-02, zero importers)
- Wrote 08-builder-admin-wiring.md inline (supplemental unit: AdminDashboard real content → AdminShell tabs; its 9 tsc errors owned there; orchestrator-provenance amendment, ground-truth verified)
- tsc remaining: 14 (9 AdminDashboard → unit 08; Jobs 1 / Profile 1 / Setup 3 → Wave C builders rebuild those files)

## WAVE C — DISPATCHED 14 Sept ~12:5xZ (true-concurrent background)
builder-05 · (pending) · spec 05-builder-pages-core.md
builder-06 · (pending) · spec 06-builder-pages-account.md
WAVE D queue (after C verified): builder-07 (07-builder-tests) + builder-08 (08-builder-admin-wiring) → critic → commit series

## WAVE C attempt 1 — FAILED 14 Sept ~13:0x (engine-level)
builder-05 · 67358285 · FAILED — fast-fail (7s runtime, 0 trips/tokens, SUCCEEDED-status lie, zero disk writes)
builder-06 · 4b308b7d · FAILED — silent death (11m runtime, zero disk writes, Result pending)
tsc unchanged at 14; none of the 8 target pages touched. Re-dispatching both (attempt 1 of 2).
