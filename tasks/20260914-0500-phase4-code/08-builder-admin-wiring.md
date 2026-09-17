# Task Spec — Admin shell content wiring (run 20260914-0500-phase4-code)

*Supplemental unit written by the ORCHESTRATOR (ground-truth-verified amendment, 14 Sept ~12:50 —
builder-02's AdminShell shipped placeholders; the real wired content was left in the unrouted
AdminDashboard.tsx). The 9 tsc errors below live in AdminDashboard.tsx and are this unit's to clear.*

## Mission
Wire the REAL admin content (users, submissions moderation, market toggles, portal health) from the
now-unrouted `src/pages/AdminDashboard.tsx` into the routed `src/components/AdminShell.tsx` tab
structure, per 05-ia-menu-spec §4 admin rows. Delete nothing that has real data calls; every backend
admin endpoint keeps its UI (standing user directive).

## Context (verified on disk 14 Sept)
- `src/App.tsx` routes `/admin` → `AdminGuard` → `AdminShell` (new minimal shell, 5 tabs: Users /
  Portals / Markets / Submissions / Templates — placeholder TabsContent divs only).
- `src/pages/AdminDashboard.tsx` (unrouted, 700+ lines) has the real content: Users tab (api.getUsers,
  search, edit dialog, role limits), Submissions tab (fetchSubmissions/approveSubmission/rejectSubmission
  from lib/api/admin.ts), Markets tab (api.getMarkets/activateMarket/deactivateMarket toggles),
  portal health (api.getPortalHealth), audit log tab (getAuditLog), plus 9 tsc errors to fix:
  6× `title:` in object literals (TS2353 — toast/description shape misuse), `canExportData` (TS2304),
  `RefreshCw` + `Server` (TS2304 — missing lucide imports).
- AdminShell tabs per spec §4: Users (GET /users) · Portals (GET /jobs/portal-health, PATCH /portals/{id},
  POST /tools/robots-check, POST /tools/portals, POST /tools/portals/{id}/test sentinel) · Markets
  (GET /markets, activate/deactivate) · Submissions (GET /submissions, approve/reject) · Templates
  (GET+POST /tools/templates, activate, verify-compile).
- Client methods available (verify in client.ts): getUsers, getPortalHealth, getMarkets,
  activateMarket, deactivateMarket, verifyTemplateCompile, testPortal + admin.ts fetchSubmissions/
  approveSubmission/rejectSubmission. If getPortals/getTemplates exist, prefer them for Portals/Templates tabs.
- Tools.tsx already has admin portal extras (sentinel-test); the ADMIN shell is the operator's primary
  surface for the same endpoints (§4 "admin extras" on /tools are for admin-role seekers visiting /tools —
  do not strip them).

## Steps
1. Refactor AdminDashboard content INTO AdminShell: move each real tab's JSX + handlers (users,
   submissions, markets) into AdminShell's TabsContent; keep AdminShell's own header/tabs. One tab per
   step; checkpoint after each (write file, run `npx tsc --build tsconfig.app.json 2>&1 | grep AdminShell`).
2. Wire Portals tab: portal-health list + per-portal enable/disable + sentinel test (api.testPortal).
3. Wire Templates tab: template list + activate + verify-compile result display (api.verifyTemplateCompile).
4. DELETE src/pages/AdminDashboard.tsx once all its real content is moved (grep imports first).
5. Fix the 9 tsc errors as you port: correct the `title:` object-literal misuses (toast API takes
   strings), import RefreshCw/Server from lucide-react or drop them, resolve canExportData (delete or
   implement via hasPermission('admin:reports:read')).
6. Clear Deck styling: tokens only (AdminShell already token-clean); ported JSX must drop any
   old-palette classes if present.

## Done criteria (grep-able)
- `npx tsc --build tsconfig.app.json 2>&1 | grep -c "error TS"` → 0 for AdminShell.tsx + AdminDashboard gone.
- `ls src/pages/AdminDashboard.tsx` → No such file (content ported, file deleted).
- AdminShell.tsx contains: `api.getUsers`, `fetchSubmissions`, `activateMarket`, `getPortalHealth`,
  `verifyTemplateCompile|getTemplates`, `testPortal` (grep each ≥1).
- `npx vitest run` exit 0.
- Zero hard-coded hex in AdminShell.tsx (`grep -cE "#[0-9a-fA-F]{6}" src/components/AdminShell.tsx` → 0).

## Output (files touched)
- src/components/AdminShell.tsx (content wiring)
- src/pages/AdminDashboard.tsx (DELETED after port)

## Depends
02-builder-nav (AdminShell exists) — satisfied. Runs in Wave D alongside 07-builder-tests.

## Result
(pending)
