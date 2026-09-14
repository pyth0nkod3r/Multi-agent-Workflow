# Code-Quality Improvement Plan — v2 (supersedes v1)
Date: 2026-09-14 · Status: PROPOSAL — nothing deployed until approved.

## Sources — two passes

**Pass 1 — community corpus** (Google SERP around the original query):
- dev.to "Five Lines of Code Principle" + 29 comments; Reddit r/learnprogramming 30-line thread (95 comments); Reddit "mandatory limit on LOC in a function"; SE/SO function- and file-length threads (133404, 9447, 374262, 621682, 4411413, 436085); r/Frontend + r/webdev file-size threads.
- AI-code-quality research: arXiv 2601.16839 (387 agentic PRs, 364 build smells, 61% merged with minimal review), GitClear churn study (150M LOC), Uplevel 2024 survey (67% spend MORE time debugging with AI), SLR arXiv 2512.05239 (72 studies).

**Pass 2 — the user's actual link** (https://share.google/aimode/IPGvweBxrepcembpL): a 5-turn Google AI Mode conversation:
1. "Is there a required number of lines?" → No; counting lines is a bad metric; soft function-length guidelines (5–20); measure readability/tests/simplicity instead.
2. "I'm asking so I can guide my AI towards writing better maintainable code. Online resources?" → the cited-resource list (below).
3. "Yes" → a full **System Prompt Template** (senior-engineer mode: SOLID, Clean Code, style guides, ≤30–40-line functions, ≤3 args, guard clauses, DI, explicit error handling, no-fluff output).
4. "Draft a detailed guide with cited examples" → "Designing for Maintainability" guide: PEP 8 79-char, Google C++ 40-line signal, Fowler guard clauses, SRP split (calculateAndSaveInvoice → calculateInvoice + saveInvoice), monolithic-vs-maintainable comparison table, process_order anti-pattern vs OrderProcessor example.
5. "Do both" → **Pre-Flight Code Review Checklist** (6 checkboxes) + **Reusable Prompting Macro** with "confirm compliance in 3 bullets" protocol.

AI Mode's cited authorities (now ours too):
- Google Style Guides (C++, Java, JS, Python) — C++ guide: functions >40 lines should be considered for splitting; explicitly "no hard limit".
- PEP 8 (Python) — 79 chars (72 for comments/docstrings), naming, layout.
- Clean Code (Robert C. Martin) + Softensity Clean Code Cheat Sheet — one thing per function, ≤3 arguments, ideal 4 / max 60 lines, narrative reading order.
- SOLID (SRP above all).
- Martin Fowler's Refactoring Catalog — guard clauses / early returns; split by intention vs implementation.
- automata/aicodeguide (GitHub) — Spec-Driven Development, TDD, layer separation for AI agents.
- Machine Intelligence Laboratory Python Style Guide — function ≈30 lines, "not a hard rule".
- WordPress.com "coding standards and the 20-lines rule"; Active Programmer (≤20 lines, ≤2 indent levels); Google Cloud code review style guide (consistency).

## Convergence analysis (Pass 1 vs Pass 2)

The two passes independently agree on the core — which is the strongest possible signal for adoption:
- One-responsibility-per-function via the **'and' test**: Reddit ("if you can't describe it without 'and', split it") ≡ AI Mode/Clean Code (validateAndSave must split) ≡ SOLID SRP.
- Size limits are **signals, not laws**: Reddit consensus ≡ Google C++ guide's own "no hard limit" ≡ MIL's "not a hard rule" ≡ Clean Code cheat sheet's "ideal 4, max 60".
- **Guard clauses / flat nesting**, **explicit error handling**, **inject dependencies for testability**, **style-guide authority over ad-hoc taste** — present in both passes.
- Pass 2 adds what Pass 1 lacked: **argument-count limits (arity ≤3, else config object)**, **pinned line length** (PEP 8 79), **named per-language style authorities**, **TDD/Spec-Driven Development**, and a **builder self-certification protocol** ("confirm compliance in bullets before showing code").

## ADOPTED — final improvement set

### A. Machine gates (configs)

**A1 — Frontend ESLint** (`platform/frontend/eslint.config.js`) — thresholds tuned to Pass-2 citations (Google 40 / MIL 30 / Clean Code 60 → warn at 50; SE threads on file size → 300):
```js
rules: {
  ...reactHooks.configs.recommended.rules,
  "react-refresh/only-export-components": ["warn", { allowConstantExport: true }],
  "@typescript-eslint/no-unused-vars": ["warn", { argsIgnorePattern: "^_" }],
  "max-lines-per-function": ["warn", { max: 50, skipBlankLines: true, skipComments: true }],
  "max-lines": ["warn", { max: 300 }],
  "complexity": ["warn", 10],
  "max-depth": ["warn", 3],          // Fowler: guard clauses, flat logic
  "max-params": ["warn", 3],         // Clean Code: >3 → config object/props
}
```
**A2 — Prettier** for the frontend, `printWidth: 100`, scripts `format` / `format:check`. Line length is pinned in ONE config, formatter-owned, never argued in review. (PEP 8's 79 is the purist reference; 100 is the pragmatic modern choice for TS — flagged as open question Q2b.)

**A3 — Backend ruff via uv** (`uv add --dev ruff`) in `backend/pyproject.toml`:
```toml
[tool.ruff]
line-length = 100
[tool.ruff.lint]
select = ["E", "F", "I", "B", "UP", "SIM", "C90", "S", "PLR0913"]
ignore = ["E501"]  # formatter owns line length
[tool.ruff.lint.mccabe]
max-complexity = 10
[tool.ruff.lint.pylint]
max-args = 4        # Python signatures get one extra slot (Depends/DI params)
[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]  # assert() in tests is correct
```
NEW: `"S"` (flake8-bandit) is the direct machine gate for the research's top *security* smells — hardcoded credentials/paths/URLs (S105/S106, hardcoded tmp paths, etc.).

**A4 — One QA command per repo**: frontend `npm run qa` = `tsc -b && eslint . && prettier --check . && vitest run`; backend `uv run ruff check . && uv run pytest -q`. This is the thing wave-gating and critics actually run.

### B. Builder quality floor — 7 gates (roster/builder.md → v4)

Appended after RESULT-LAST; supersedes v1's six gates:

```
- QUALITY FLOOR (v4, from 2026-09-14 code-quality plan; sources:
  /workspace/multiagent/knowledge/code-quality.md):
  (1) SINGLE RESPONSIBILITY: every function does exactly one thing. Apply the
      'and' test — if you cannot describe it without the word "and", split it
      (calculateAndSaveInvoice → calculateInvoice + saveInvoice). Line counts
      are signals, never quotas: linter size warnings are things to justify,
      not laws (Google C++ guide: "no hard limit" — even at 40 lines).
  (2) FLAT LOGIC: guard clauses + early returns; keep nesting ≤3 levels.
      Exit on error states first; never wrap the happy path in deep indents
      (Fowler, Replace Nested Conditional with Guard Clauses).
  (3) ARITY: ≤3 parameters per function; more → group into a config object /
      data class / props object (Clean Code).
  (4) ERROR HANDLING at every I/O boundary (network, DB, file, subprocess):
      explicit success/failure paths; NO silent catch-all swallowing. It is
      the #2 documented AI-code smell (63 of 364; arXiv 2601.16839).
  (5) NO hard-coded configuration: URLs, paths, ports, keys, secrets, magic
      numbers used 2+ places → config/env/token layer. NO copy-paste
      duplication: second occurrence of a ≥5-line block becomes a helper
      (GitClear: duplication is the dominant AI churn smell).
  (6) TESTABILITY & LAYERING: inject external dependencies (DB client, HTTP
      handler, clock, logger) — never construct them inside business logic;
      keep business logic separate from storage/UI/network layers (SOLID-DIP,
      separation of concerns).
  (7) STYLE AUTHORITY: follow the repo's formatter/linter config exactly;
      where none exists, PEP 8 (Python) / Google style (TS-JS) / official
      Kotlin style (Compose). Naming follows the file's existing convention
      (research: AI code drifts into non-standard naming). Prefer deleting
      and simplifying over adding when the spec allows — negative LOC is
      real productivity.
- VERIFY-BEFORE-WRITE: run the repo's QA command for every touched area
  BEFORE the final artifact+## Result write. Lint clean (warnings addressed
  or justified) and tests green. A green run you did not personally trigger
  counts as FAIL(unverified).
- SELF-CERTIFICATION: your closing paragraph must include one line of gate
  compliance — which gates passed clean, which are justified (with the
  justification). Never claim a gate you did not verify.
```

### C. Critic amendment (roster/critic.md → v4)

```
- QUALITY GATES (v4): beyond the spec's done criteria, run the repo's QA
  command yourself (lint / format:check / targeted tests) — a criterion you
  could not run counts as FAIL(unverified). Flag as 'warn' (blocker only if
  the spec says so): functions failing the 'and' test; linter size/depth/
  arity warnings left unjustified; I/O without error handling; silent
  catch-alls; hard-coded URLs/paths/secrets/magic numbers (grep + ruff S);
  ≥5-line duplicated blocks; hardcoded deps inside business logic; naming
  that breaks the file's convention. Tag each issue with its gate number
  from knowledge/code-quality.md.
```

### D. Planner amendment (roster/planner.md)

```
- Every spec MUST contain a QUALITY section copying the 7-gate checklist
  from /workspace/multiagent/knowledge/code-quality.md as done criteria
  (N/A gates are named explicitly — silence is not exemption).
- TDD-lite (from aicodeguide / Spec-Driven Development): a spec that changes
  behavior must name the test(s) that verify it — extend EXISTING test files;
  done criteria include "tests updated and green". (Our planner→spec→builder
  flow already IS spec-driven development; this closes the test half.)
```

### E. Knowledge file (`/workspace/multiagent/knowledge/code-quality.md`)

The single reference: 7 gates, exact QA commands per repo, all citations (Google/PEP 8/Clean Code/SOLID/Fowler/aicodeguide/MIL/Reddit corpus + arXiv 2601.16839/GitClear/Uplevel). Prompts stay short; evidence stays auditable. Includes AI Mode's monolithic-vs-maintainable comparison table as the quick reference.

### F. Process amendments
- **F1 — Wave quality gating** (ORCHESTRATION §B2): a unit is session-verified only when ## Result filled + files on disk + QA clean for touched paths + tests green (orchestrator runs QA inline; seconds).
- **F2 — Self-certification protocol** = AI Mode's "confirm compliance in 3 bullets", mapped onto our builder closing paragraph + critic gate tags.
- **F3 — Re-deploy sync rule unchanged**: I edit roster + push repo; YOU re-deploy builder/critic/planner via RikkaHub UI. Never one side only.
- **F4 — Debt ledger** (optional): every justified warning gets one line in `platform/docs/quality-ledger.md` (research: 30-41% tech-debt growth in AI-heavy codebases; visible debt ≠ invisible churn).

### G. Mapping — AI Mode artifact → our mechanism
| AI Mode recommended | Our implementation |
|---|---|
| System Prompt Template (senior-engineer mode) | builder.md v4 quality floor (7 gates) — our profile already carries protocol rules (checkpointing, RESULT-LAST); the template's 4 sections fold into the floor |
| Pre-Flight Code Review Checklist | builder SELF-CERTIFICATION line + critic QUALITY GATES (critic re-runs the checks, doesn't trust the builder) |
| Reusable Prompting Macro | planner QUALITY section + spec template (specs carry the constraints so every dispatch re-injects them) |
| "Designing for Maintainability" guide | knowledge/code-quality.md (gates + comparison table + citations) |
| Cited style guides (Google/PEP 8/Fowler/SOLID) | gate 7 style authority + lint/formatter configs as the executable version |
| aicodeguide Spec-Driven Development + TDD | already our planner-first architecture; TDD-lite closes the test half |

### H. Explicitly NOT adopting (with sources)
- ❌ Hard line-count laws (5, 20, 30, 40) — refuted by every source INCLUDING the pro-rules ones: Google C++ guide itself says "no hard limit"; MIL says "not a hard rule"; dev.to's own top comment; Reddit consensus. Warn-level machine signals only.
- ❌ Hard 79-char cap enforced by rule — formatter-owned line length instead (E501 ignored; formatter pins it). The limit is a config value, not a review argument.
- ❌ Line-counting in human/AI review — the machine counts; reviewers judge 'and' tests.
- ❌ "Under 20 lines" aggressive targets (WordPress 20-lines rule) — cited as reference, not adopted: our codebase is React/TS + FastAPI; 20-line components would force premature fragmentation ("prematurely factoring into tiny functions may harm maintainability" — Reddit).
- ❌ Error-level any of these on day one — everything starts warn-level with baseline triage.

## Rollout order (post-approval)
1. A1–A4 configs → run baseline → triage pre-existing warnings (separate small task; NOT mixed into wave units)
2. E knowledge file
3. B/C/D roster edits + push to Multi-agent-Workflow repo
4. USER re-deploys builder/critic/planner in RikkaHub UI (paste from roster)
5. F1 ORCHESTRATION amendment + push; F4 ledger optional
6. First live test: next phase4-code wave under the new gates

## Open questions
- Q1: Thresholds now 50 (function) / 300 (file) / complexity 10 / depth 3 / params 3 — tuned to the cited sources. OK, or adjust?
- Q2a: Prettier — yes/no? (one-time reformat diff; scoping to src/ possible) · Q2b: printWidth 100 vs strict 80/79?
- Q3: Debt ledger (F4) — adopt or drop?
- Q4: Scope — platform repos only, or also Dp-600/FabricFocus Compose work (gate 7 would cite official Kotlin style; note: that repo builds on-device, gates would run via termux/ACS, limited)?
- Q5: Backend line-length: ruff 100 (matches Prettier 100) vs PEP 8-strict 79/88 (Black/ruff-format default)?
