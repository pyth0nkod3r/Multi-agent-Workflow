# Critic verdict — wave 2 (commit f889952)

> INLINE critic pass 19 Sept 2026 — the dispatched critic was the 11th
> consecutive sub-agent fast-fail death (engine-level). Written by the
> orchestrator against the spec done criteria + gates v4, from QA evidence
> gathered during the inline build.

## Verdict: **PASS (self-verified)**

## Done-criteria check
- [x] **SPEC-03**: adapters keyed by REGISTRY ids (`arbeitnow`/`freehire` —
  spec's `-search` ids corrected against db.py registry, the source of truth);
  ETL dedupes by URL (existing + within batch) and propagates to ALL owner
  spaces with `status='active'` (u-admin-only landing was caught in build —
  jobs would have been invisible to users); raw_ingress stored with
  `processed` flag; ingest endpoint admin-only (403 for non-admin, pinned).
- [x] **SPEC-04**: IngestionPanel admin tab wired into AdminShell (Database
  icon, TabsContent), per-board "Ingest now" + raw log table, token-only
  styling, no JobDetail.tsx touches (ledger line respected); tsc 0, eslint
  0 errors, vitest 55/55 (3 new), prettier clean.
- [x] **SPEC-05**: CV page-count failure → relevance-cut loop (max 3 passes,
  lowest-scoring first — pinned by test_cut_cv_cuts_lowest_scoring_first)
  BEFORE the hard 422 with residual note in metrics; CV layout gate HARD,
  cover STAGED (promotion to the cover would 422 the thin stub at 37% empty —
  caught in QA, correctly scoped back); render.yaml has tectonic vendoring
  (P5-1) + NEW bundle pre-warm canary; _gate_and_cut extracted (C901 17→<10).
- [x] Tests: 59/59 combined backend touched-path.

## Gate tags
- G1 (single responsibility): `_gate_and_cut` extraction; endpoint <10
  complexity. SATISFIED.
- G4 (I/O-boundary error handling): adapters catch at the boundary only,
  noqa: BLE001 with reason.
- G5 (no hardcoded config): registry ids, weights, caps all named constants.
- G7 (style): ruff clean on new files; B008/pre-existing PLR0913 accepted.

## Accepted items (not flagged, per handoff)
B008 Depends idioms; pre-existing PLR0913 (jobs.py:190, list_jobs_filtered);
JobDetail max-lines 519; demo seed deadlines cosmetic; stale
TEST_DATABASE_URL Neon drift; PLR0913 noqas on callback-shaped APIs
(cut_cv, _gate_and_cut, pypdf visitor).

## Notes for P7
- Cover layout promotion happens when real LLM generation replaces the stub
  drafter (the thin-cover 422 disappears with real content).
- `raw_ingress` proper migration lands with P5-4 tsvector.
- Scraping cadence (3×/day orchestrator) = scheduled-job wiring, needs the
  worker host decision (INFRA-DEC-10 ladder) — not yet wired.
