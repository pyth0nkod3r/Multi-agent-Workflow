status-planner-05.md · 2026-09-14 05:00 UTC

01-builder-tokens.md · verified · 2026-09-14T05:00:00Z
  Complete token-migration spec: Mission+Context (verified ground truth from DESIGN.md, index.css, tailwind.config.ts, App.css, chart.tsx), Steps (6 steps with checkpoints covering :root rewrite, tailwind borderRadius update, App.css legacy hex removal, component-level hex sweep, vitest pass, vite build pass), Done criteria (grep-able old-value hits + build green), Output (3 files), Depends (none).

02-builder-nav.md · verified · 2026-09-14T05:00:00Z
  Complete nav-restructure spec: Mission+Context (verified current App.tsx routes, AppSidebar.tsx flat nav, AuthContext role system, 05-ia-menu-spec §2/§3/§4 verbatim tables, no BottomNav/MoreSheet on disk), Steps (7 steps with checkpoints covering ProChip creation, AppSidebar group restructure + active-state rail, BottomNav+MoreSheet creation, AppLayout update, App.tsx router diff + AdminShell, Rank→Shortlist rename, vitest pass), Done criteria (grep-able route strings + group labels + PRO chip counts + guard changes), Output (7 files), Depends (none).

Summary: Both spec files are self-contained, disk-verified, and parallel-safe. 01 covers the CSS token layer migration (hsl(var(--*)) pattern preserved, zero component churn) with the complete new :root block verbatim. 02 covers the nav/IA restructure per 05 contract with exact route diffs, sidebar group structure, mobile 5-tab + More sheet, PRO chips, and admin island. Old routes /markets, /billing, /rank are removed/renamed; /jobs/:id and /shortlist added. Dispatch order: 01 and 02 can run concurrently with 03; 01 and 02 have no inter-dependency.
