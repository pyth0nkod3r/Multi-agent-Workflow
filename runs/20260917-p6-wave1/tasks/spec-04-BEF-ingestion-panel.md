# SPEC-04: BE-F — ingestion panel UI (admin-only)

> Run wave2 · EaseApply (/workspace/platform) · builder unit ≤30 min
> User directives: users don't see the scrape menu — admin does.

## Context (read only what's named)
- `frontend/src/pages/Integrations.tsx` + admin shell wiring (AdminShell
  tabs per HANDOVER §5 FR-1) — the panel lives in the ADMIN surface.
- `frontend/src/lib/api/client.ts` — dual-mode pattern (apiEnabled →
  apiFetch, else mock mutation); `frontend/src/lib/api/types.ts` — type
  conventions. No hard-coded hex (DESIGN.md tokens; shadcn components).

## Deliverables (disjoint from SPEC-03/05)
1. `client.ts`: `ingestPortal(portalId)` → POST /tools/portals/{id}/ingest
   (returns {ingested, deduped, errors}); `getIngressLog()` → GET
   /admin/ingress (mock fallback: in-memory list, initially empty).
2. `frontend/src/pages/IngestionPanel.tsx` (new): admin panel — per-portal
   "Ingest now" action + recent raw-ingress log (source, fetched_at,
   ingested/deduped counts, processed flag). shadcn Table/Badge/Button,
   existing tokens only. Loading/disabled states.
3. Wire into the admin shell (AdminShell tabs — read the existing wiring in
   Integrations.tsx/AppLayout or wherever AdminShell lives) as a new tab.
4. Frontend test: follow existing src/test conventions — a render test
   asserting the panel renders and the ingest action exists.
5. QA: cd /workspace/platform/frontend && npm run qa (eslint+prettier+tsc+vitest).

## QUALITY (gates v4)
SRP · ≤3 nesting · arity ≤3 · I/O error handling · no hardcoded config ·
no ≥5-line duplication · fn 50 / file 300 / complexity 10 / depth 3.
NOTE: JobDetail.tsx is at 519 lines (ledger) — do NOT add to it; the panel is
a separate file.

## Checkpoints
client methods → panel component → admin wiring → test → QA → "## Result".

## Result
(built by — orchestrator fills after disk verification)
