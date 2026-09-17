# Planner-07 status (20260914-0500-phase4-code)
04-builder-pages-grow.md · verified · 2026-09-14T05:16:31Z
05-builder-pages-core.md · verified · 2026-09-14T05:16:31Z

Planner-07 decomposed the GROW and CORE page rebuilds into two self-contained specs. 04 covers 5 pages (InterviewPrep, Upskill, Expand, Tools, Integrations) with exact client method names from 03, token-only styling, PRO gating, honest microcopy, and per-page grep-able done criteria. 05 covers 5 surfaces (Dashboard, Jobs list, JobDetail, Shortlist rename from Rank, Applications) with Stitch screen IDs, closingSoon surfacing, 5-col kanban, follow-ups strip, and a Rank→Shortlist rename. Both specs include Step-by-step instructions ending in "write + verify on disk" so each page is a checkpointable unit. Dependency order: dispatch 03 first (client methods), then 04 and 05 can run in parallel; within each spec, pages are ordered by data dependency (list pages before detail pages).
