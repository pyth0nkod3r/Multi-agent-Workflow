# RUN 20260917-p6-wave1 — EaseApply P6 wave 1 (mechanical gates + stale-job lifecycle)

Goal: spec + build wave-1 units for EaseApply (/workspace/platform):
- BE-B: document verification gates (page-count 2p/1p + layout-hole checks) as
  backend service, wired into generation flow, with tests. Port of upstream
  #476 lesson (verify_pdf.py --pages). NOTE: latex.py already has compile_tex,
  extract_text, ats_checks, keyword_coverage — gap is page/layout gates only.
- BE-E: stale-job lifecycle — 14-day auto-expiry + report-closed endpoint +
  JobDetail "Report as Closed" button + admin reversal path. jobs.status field
  exists ('new' default). DECISION (user-approved default, reversible): instant
  community flag flip, no moderation queue.
- SPIKE: production TeX compilation strategy for Render free tier (no-Docker):
  precompiled templates+pypdf overlay vs Tectonic single binary vs texlive-apt
  build vs client-side compile vs external API. Research output only, no build.

Wave plan (protocol §B2, 2-per-wave cap):
- Wave 0 (concurrent, background): planner (writes task specs under
  runs/20260917-p6-wave1/tasks/) + researcher (SPIKE LaTeX findings).
- Wave 1: builder-BE-B + builder-BE-E against planner specs (after specs disk-verified).
- Critic after builders; merger n/a (disjoint artifacts).

Status:
- [x] 17 Sept 04:5x — scaffold, upstream merged (ai-job-search e44f491), research doc committed (platform 63a79d2)
- [ ] Wave 0 dispatched (planner + researcher, background)
- [x] Wave 0 dispatches ×2 DIED (planner+researcher fast-fail, zero disk) → retries -r1 ×2 ALSO DIED → INLINE FALLBACK per protocol
- [x] 17 Sept ~08:0x — specs + SPIKE research written INLINE by orchestrator (spec-01-BE-B-verify-gates.md, spec-02-BE-E-stale-jobs.md, research-latex-strategy.md — all disk-verified)
- [ ] Wave 1: builder dispatch (spec-01 + spec-02) — attempt; fallback = inline build
- [x] Wave 1 builders dispatched ×2 → fast-fail died → INLINE BUILD by orchestrator
- [x] SPEC-01 BE-B: services/verify.py + fit.py wiring + 10 tests — 45/45 pass, ruff clean. KEY CATCH: pypdf tm is page-relative (cm transform math needed); staged gate policy (page_count hard 422, layout findings) documented
- [x] SPEC-02 BE-E: mixin (14-day sweep, report-closed, reopen cross-owner) + JobDetail UI + demo seed relative dates + 8 tests. Catches: naive/aware datetime, sweep-after-copy, admin 404, S608 → literal SQL
- [x] Committed platform f9cf464 (pushed); spec Results filled; ledger + HANDOVER §5 updated
- [ ] Critic pass (attempt; engine fast-fail likely → inline self-review fallback)
