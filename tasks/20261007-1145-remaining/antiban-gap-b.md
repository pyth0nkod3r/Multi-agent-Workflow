# SPEC: anti-ban Gap B — consistency layer + periodic self-check (wa-listener)

Run: 20261007-1145-remaining · Unit: antiban-gap-b · Builder-owned files: wa-listener/** only
Source: docs/RESEARCH-academic-bias-detection.md `## Result` item #2 (HIGH impact / LOW-MED effort adopt)
  — FP-Inconsistent (arXiv:2406.07647) + JA4 (arXiv:2602.09606): inconsistent cross-layer fingerprints
  are the top evasion failure; behavioral entropy (arXiv:2410.18233) is the discriminator.

## Context (verify on disk before writing)
- Repo /workspace/platform; wa-listener/ = source mirror of what is deployed on HidenCloud
  (/home/container/wa-listener). Listeners are NOT yet live-linked (user hasn't filled tg.env /
  paired WA) — repo changes are picked up at go-time (go-live checklist includes npm install/build).
  Do NOT attempt SSH/SFTP deploy.
- Known duplication to kill: src/index.ts lines ~68-73 set `markOnlineOnConnect:false` +
  `syncFullHistory:false` at TWO call sites (initial + reconnect) — exactly the cross-layer
  self-contradiction risk this unit eliminates.
- Existing anti-ban posture (commit be08e45 — DO NOT REGRESS any of it): markOnlineOnConnect:false,
  syncFullHistory:false, random read-delay 500–2000ms, strictly read-only (no sendMessage / no
  presence updates), group allowlist, 24h age window, stable session, batched POSTs.
- src/ = config.ts, filter.ts, heartbeat.ts, index.ts, queue.ts, types.ts; tests/filter.test.ts
  (12/12 via tsx --test; .js→.ts imports need tsx).

## Done criteria
1. SINGLE SOURCE OF TRUTH: extract the Baileys socket config into one builder
   (e.g. `buildSocketConfig()` in src/config.ts) used by BOTH call sites in src/index.ts —
   zero duplicated markOnlineOnConnect/syncFullHistory literals outside it (gate 5).
2. Periodic self-check: src/selfcheck.ts — `runSelfCheck(cfg)` validating the runtime posture:
   markOnlineOnConnect === false, syncFullHistory === false, read-delay bounds within 500–2000ms,
   allowlist semantics sane, and a config-shape check proving index.ts passes the builder's output.
   Called at startup AND on an interval (every 6h, `.unref()` so it never holds the event loop).
   Contradiction → structured one-liner `console.warn('SELFCHECK-FAIL: <field> ...')` — NEVER a
   crash, NEVER a silent pass. Read-only over config; never mutates.
3. Tests (new tests/selfcheck.test.ts, ≤300 lines): builder parity (both call sites identical
   output), each predicate true/false path, fail-output shape. Gates: `npx tsc --noEmit -p tsconfig.json`
   → 0; `npx tsx --test tests/` all green (12/12 + new); prettier clean on touched files.
4. ZERO behavioural change beyond centralization + the check: read-delay, allowlist, batching,
   queue, heartbeat logic untouched. Importer surface: zero changes outside wa-listener/.

## Quality gates
v4 gates 1-7 + gate 8 zero-day audit: audit every lint error as a potential zero-day BEFORE any
suppression; no blind noqa — fix or verify-safe-with-comment. Git hygiene: builders NEVER git
commit/push — orchestrator commits after disk verification. Checkpoint to disk after every step
(never >1 step unwritten); <2min foreground commands; 60%-budget early-stop with a status paragraph.

## Result
BUILDER RESULT (20261007-1145-remaining / antiban-gap-b) — DONE, disk-verified 7 Oct 2026:
1. SINGLE SOURCE OF TRUTH: src/config.ts now exports `SocketConfig` + `buildSocketConfig(auth)`
   (markOnlineOnConnect:false, syncFullHistory:false live ONLY there; grep-verified zero
   duplicated literals elsewhere in src/). src/index.ts builds `socketConfig` once and passes it
   to BOTH ternary call sites of makeWASocket.
2. NEW src/selfcheck.ts (104 lines): pure `runSelfCheck(cfg, socket, onFail)` (arity 3) with
   per-predicate checks (markOnlineOnConnect, syncFullHistory, builder-parity, read-delay
   500–2000ms bounds incl. min<=max + finiteness, allowlist group-JID sanity); never mutates,
   never crashes (thrown predicates → SELFCHECK-FAIL: selfcheck entry). `startSelfCheck(cfg,
   socket)` runs at startup + 6h interval with `.unref()`; re-entrant-safe (clears prior timer —
   reconnect recursion cannot stack intervals). Failure line: `SELFCHECK-FAIL: <field> - <detail>`
   via console.warn (exported warnFailure; injectable onFail for tests).
3. NEW tests/selfcheck.test.ts (~155 lines, ≤300): builder parity, every predicate true/false
   path, fail-line shape (prefix + single-line).
QA (all triggered personally, post-final-format): `npx tsc --noEmit -p tsconfig.json` exit 0;
`npx tsx --test tests/` 28/28 pass (12 existing filter + 16 new); prettier (--single-quote
--print-width 100 --trailing-comma none, wa-listener has no own config) clean on selfcheck.ts +
selfcheck.test.ts; config.ts fails ONLY on the pre-existing 118-char groupAllowlist line
(predates this unit; untouched per patch discipline — same baseline as all 6 pre-existing
wa-listener files which fail bare prettier too).
Zero behavioural change: read-delay/allowlist/batching/queue/heartbeat logic untouched; zero
changes outside wa-listener/. Files: src/config.ts, src/index.ts, src/selfcheck.ts,
tests/selfcheck.test.ts. NEEDS COMMIT: wa-listener/{src/config.ts,src/index.ts,src/selfcheck.ts,tests/selfcheck.test.ts} [run 20261007-1145-remaining].

ORCHESTRATOR CORROBORATION (7 Oct 2026, ~12:56): independent disk verification CONFIRMS every claim —
`npx tsc --noEmit -p tsconfig.json` → 0; `npx tsx --test tests/` → 28/28 (12 filter + 16 selfcheck);
all 4 files on disk; selfcheck.ts content matches the described predicates. Committed by orchestrator.
