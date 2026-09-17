# SPEC-01: BE-B — document verification gates (page-count + layout)

> Run 20260917-p6-wave1 · EaseApply (/workspace/platform) · builder unit ≤30 min
> Provenance: upstream ai-job-search #476 — "measure first, then look" must be
> runnable code. See /workspace/platform/docs/RESEARCH-aimode-thread.md §2.3.

## Context (read only what's named)
- `backend/app/services/latex.py` — has `compile_tex`, `extract_text`,
  `ats_checks`, `keyword_coverage`, `CompileResult` (`.pages` exists).
  Do NOT duplicate these. This unit fills the missing gate: hard page-count +
  layout-hole enforcement on GENERATED documents.
- Reference semantics: `/workspace/ai-job-search/tools/verify_pdf.py` (`--pages`)
  and `/workspace/ai-job-search/tools/verify_layout.py` (hole > 100pt, final
  page > 35% empty, non-final page ending > 25% early, heading stranded at
  break). Port the checks, not the CLI.
- Router wiring point: `backend/app/routers/tools.py` (see
  `verify_template_compile` for the response-shape convention).

## Deliverables (files this unit owns — disjoint from SPEC-02)
1. `backend/app/services/verify.py` (new):
   - `check_page_count(pdf_bytes: bytes, expected: int) -> GateResult` — pypdf
     page count, hard fail unless exactly `expected`.
   - `check_layout(pdf_bytes: bytes, *, page_h: float = 841.89) -> GateResult`
     — per page: bottom whitespace share, largest vertical gap between
     consecutive text lines (from `extract_text` + y-coords via pypdf
     `visitor_text`), final-page emptiness. Fail thresholds: hole > 100pt,
     final page > 35% empty, non-final page ending > 25% early.
   - `GateResult` dataclass: `ok: bool`, `gate: str`, `detail: str`,
     `metrics: dict`. Frozen, no I/O inside checks (bytes in → verdict out).
2. Wire into `latex.py` compile flow + `routers/tools.py`: generated CV →
   `check_page_count(.., 2)` + `check_layout(..)`; cover letter → 1 + layout.
   Over-budget → structured 422-style error payload (`gate`, `detail`,
   `metrics`), never a silent warning. Constants (2, 1, thresholds) as module
   constants, not magic numbers inline.
3. Tests `backend/tests/test_verify_pages.py` (TDD-lite, name these):
   - `test_cv_over_two_pages_fails` (3-page fixture bytes → not ok)
   - `test_cv_two_pages_passes`
   - `test_cover_letter_two_pages_fails` (limit 1)
   - `test_final_page_over_35pct_empty_fails`
   - `test_hole_over_100pt_fails`
   - `test_router_returns_structured_gate_error` (API-level, tools.py path)
   Build minimal PDF fixtures with pypdf/reportlab-free approach (hand-rolled
   minimal PDF or reuse compiled canary from latex.py CANARY_TEX).

## QUALITY (gates v4)
Single responsibility (checks pure, wiring separate) · ≤3 nesting (guard
clauses) · arity ≤3 params (use kwargs object/dataclass if more needed) ·
error handling at I/O boundary only (subprocess/pypdf calls) · no hardcoded
config (module constants; page limits referenced from one place) · no ≥5-line
duplication (layout loop shared) · thresholds: fn 50 / file 300 / complexity
10 / depth 3 · ruff + pytest green (`UV_CACHE_DIR=/workspace/platform/backend/.uv-cache
UV_LINK_MODE=copy uv run ruff check . && uv run pytest -q` in backend/).

## Checkpoints
Write verify.py → run pytest (new file) → write router wiring → run full
backend QA → fill "## Result". Never >1 step unwritten.

## Result
(built by — orchestrator fills after disk verification)
