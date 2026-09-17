# SPEC-05: BE-D — relevance-cut engine (+ render.yaml tectonic vendoring)

> Run wave2 session 2 · gated on SPEC-03/04 disk-verified · builder ≤30 min

## Context
- `backend/app/routers/fit.py` compile flow — page-count gate (hard 422) +
  layout findings staged (verification.layoutGates). THIS unit adds the
  auto-fix loop so layout findings can be promoted to hard gates.
- Fit scoring: `backend/app/routers/fit.py` evaluate-fit + scoreBreakdown
  conventions; upstream smart-relevance-cut semantics (RESEARCH-aimode-thread
  §2.3): score every CV bullet vs the target JD, cut lowest-scoring lines to
  hit the page budget — not oldest-first.
- `backend/app/services/verify.py` — check_page_count (MAXIMUM semantics),
  check_layout, run_document_gates.
- `render.yaml` (repo root, rootDir backend) + docs/DEPLOYMENT-CHECKLIST.md.

## Deliverables
1. `backend/app/services/relevance.py` (new): `score_bullet(bullet: str,
   jd_text: str) -> float` — keyword-overlap + seniority/skill weighting
   (simple, deterministic, no LLM in this unit — P7 wires real gen later);
   `cut_cv(profile: dict, job: dict, *, page_limit: int, pdf_bytes: bytes)
   -> tuple[dict, list[str]]` — when check_page_count fails, iteratively
   drop lowest-scoring bullets from build_cv_tex output (rebuild source,
   recompile, re-gate; cap iterations at 3, document residual).
2. Wire into fit.py compile: page-count failure → relevance-cut loop →
   recompile → re-gate; if still over budget after 3 passes → the existing
   422 (with residual note in metrics). Layout findings promotion: with the
   loop in place, layout failures ALSO become hard 422s (remove the staged
   exception ONLY for docs the loop can fix — update the staged-policy
   comment + ledger in the same commit).
3. render.yaml build step: curl the Tectonic static binary + pre-warm/freeze
   the bundle cache (see tasks/research-latex-strategy.md — primary
   recommendation); document in DEPLOYMENT-CHECKLIST.md (var table unchanged).
4. Tests `backend/tests/test_relevance_cut.py` (TDD):
   `test_score_bullet_ranks_relevant_higher`,
   `test_cut_cv_reduces_pages_under_budget`,
   `test_cut_cv_cuts_lowest_scoring_first`,
   `test_residual_after_3_passes_raises_with_note`,
   `test_layout_gate_promoted_to_hard`.

## QUALITY (gates v4)
SRP (scoring pure, loop in service, endpoint unchanged shape) · ≤3 nesting ·
arity ≤3 (kwargs) · I/O-boundary error handling · no hardcoded config
(iteration cap + weights as named constants) · no ≥5-line duplication ·
fn 50 / file 300 / complexity 10 / depth 3 · QA env vars per RUN.md.

## Checkpoints
relevance.py → tests → fit.py wiring → render.yaml → full QA → "## Result".
Never >1 step unwritten.

## Result
(built by — orchestrator fills after disk verification)
