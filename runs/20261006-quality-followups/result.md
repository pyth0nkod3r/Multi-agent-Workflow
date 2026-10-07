# Result — quality-ledger triage follow-ups (frontend)

Run: 20261006-quality-followups · Builder run, inline-dispatched (no spec file → this file is the result block).
Date: 6 Oct 2026. Status: DONE. Rating: 10/10 against the dispatch's done criteria (`npx tsc -b` 0 / zero output).

## What was produced

### `frontend/src/components/AdminShell.tsx`
- **UsersTab error guard**: restored the `error` getter (triage had elided it to setter-only) and added
  `if (error) return <LoadError message={error} onRetry={load} />;` immediately before the return — the exact
  pattern the five sibling tabs use (verified against PortalsTab etc., same component + placement).
- **MarketsTab unmapped section PAINTED**: restored the `unmapped`/`unmappedError` getters using the EXISTING
  `GET /markets/unmapped` fetch (unit 08's wiring untouched). `unmapped` now initializes to `null` (unloaded
  boundary) so the empty state cannot lie before the fetch resolves (unit-04 lesson). Return wrapped in the
  siblings' `space-y-4` div; new "Unmapped locations" Card with boundary predicates:
  section-local error (text-destructive `<p>`, AntiBanTab precedent) → "Loading unmapped locations…" →
  "All job locations map to a market" → rows mirroring the Markets Registry row markup (location | count badge).
  No null-paint on any path.
- Nothing else in the file touched.

### `frontend/src/test/admin-shell-tabs.test.tsx` (extended, file kept ≤300 lines)
New describe "markets unmapped section + users error guard (triage follow-ups)", 4 tests:
1. Unmapped rows render (fixture Lagos, Nigeria/7 + Accra, Ghana/2 → location + count in panel).
2. Empty state "All job locations map to a market" (default `[]` stub).
3. Section-local error: `getUnmappedLocations` rejects, `getMarkets` resolves → panel shows the error
   message, registry still paints, NO tab-wide LoadError.
4. UsersTab guard: `getUsers` rejects → "Couldn't load data" + message + Retry visible, "User Management" hidden.

## QA (personally triggered on final disk state)
- `bash -c 'npx tsc -b ...'` → exit 0, zero output (real typecheck, -b build per 4 Oct lesson).
- `npx vitest run src/test/admin-shell-tabs.test.tsx src/test/admin-shell-responsive.test.tsx` → 2 files, 12/12 passed.
- `npx prettier --check` both touched files → clean.
- `npx eslint` both touched files → 0 errors; zero NEW warnings vs HEAD (verified by eslint run over
  `git show HEAD:…AdminShell.tsx`: 15 size/complexity warnings before, 15 after — all the ledger-justified
  multi-tab-shell signals). One transient `max-lines` (301>300) warning on the test file was FIXED (fixture
  folded into its test), not ledgered.

## Verify-before-fix
HEAD of AdminShell.tsx = 12ea0ea (the triage commit that intentionally elided both getters and recorded the
two follow-ups). Neither the guard nor the render existed on disk → applied fresh; nothing re-applied from a
reverted fix.

## Needs commit (orchestrator)
- `frontend/src/components/AdminShell.tsx`
- `frontend/src/test/admin-shell-tabs.test.tsx`
- message tag: `[run 20261006-quality-followups]`
- After commit: strike/annotate the two follow-up lines in `docs/quality-ledger.md` (triage run 20261005
  section — UsersTab guard + MarketsTab unmapped render). Ledger was NOT touched by this builder (not owned).

## Gate compliance (v4, 8 gates)
Pass clean: 1 (SRP — only new function is a 3-arg test helper), 2 (guard-clause order preserved), 3 (arity
≤3), 4 (explicit error paths: section-local vs tab-wide, no silent catch), 5 (no hardcoded config; reuses
LoadError; row markup is sibling-parity mandated by spec, not copy-paste logic), 6 (DI/layering untouched),
7 (style = repo prettier/eslint verified clean, naming follows file), 8 (zero-day audit N/A — no lint errors
suppressed, nothing skipped).

## Deviations
- Inline dispatch, no spec file → no `## Result` placeholder to fill; this file is the durable result block.
- quality-ledger.md left open for the orchestrator (file not in this unit's ownership).
