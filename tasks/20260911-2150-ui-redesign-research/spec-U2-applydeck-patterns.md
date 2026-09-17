# SPEC U2 — useapplydeck.com UI/UX pattern teardown (run 20260911-2150-ui-redesign-research)

**Executor profile**: researcher. **Output owner**: this file's `## Result` (fill LAST, after disk verify).

## Context
The user likes useapplydeck.com's UI and wants EaseApply redesigned following that pattern "or even a better one". Business/competitive intel on applydeck is ALREADY DONE (`/workspace/platform/docs/RESEARCH-applydeck.md`) — do NOT redo it. Your unit covers UI/UX PATTERNS ONLY: information architecture, navigation model, layout, component patterns, copy voice, responsive behavior — the raw material a designer will use to write direction briefs. Research ONLY — do not edit any code.

The applydeck LANDING page is already summarized in GOAL.md (§Recon) — do not re-fetch the homepage beyond a quick confirm. Your value is the INTERIOR pages.

## Fetch & read (web_fetch / web_extract; article mode first, text mode if empty)
- https://useapplydeck.com/jobs — job matching/search UI
- https://useapplydeck.com/trackers — application tracker board
- https://useapplydeck.com/documents?tab=resumes — resume builder
- https://useapplydeck.com/mock-interviews and https://useapplydeck.com/tools/mock-interview — interview tooling
- https://useapplydeck.com/extension — extension page
- https://useapplydeck.com/job-search, https://useapplydeck.com/signup — marketing page structure + signup funnel entry
- https://blog.useapplydeck.com (skim: tone/voice only)
If a page returns empty extraction (JS-rendered), say so in the output rather than guessing.

## Read first
- /workspace/multiagent/tasks/20260911-2150-ui-redesign-research/GOAL.md
- /workspace/platform/docs/RESEARCH-applydeck.md (existing business intel — reference, don't duplicate)

## Output file (only this path)
`/workspace/platform/docs/design/02-applydeck-patterns.md` — structure:
1. **IA & navigation model** — top-level nav, app vs marketing surface split, where the product "deck" metaphor shows up, route naming.
2. **Per-page teardown** — for each fetched page: layout skeleton (hero/list/board/etc.), primary components (cards, filters, board columns, score meters, CTAs), density, copy voice examples (short quotes).
3. **Component pattern library** — recurring components we could reuse (job card anatomy, fit-score display, tracker column/card, stat row, empty states, CTA styles) with a 1-line description each.
4. **Responsive / mobile patterns** — what the site does at narrow widths (if determinable from markup), plus what a mobile-first version of each pattern implies.
5. **ADOPT / ADAPT / SKIP table** — every pattern gets a verdict + rationale. RULE: an ADOPT/ADAPT entry must name WHICH EaseApply backend capability it maps to (applydeck features we lack — Chrome extension autofill, resume builder, mock-interview tooling, 100k+ live-jobs claims — must be SKIP with "no backend capability" rationale). Reference endpoints by name from GOAL/01-inventory.
6. **Leakage guard list** — explicit "these applydeck things must NOT appear in our nav" checklist for the critic to use.

## Constraints
- UI patterns only; pricing/market analysis stays out (already done).
- Quote sparingly (fair use, 1-2 lines per pattern); describe rather than copy wholesale.
- Checkpoint to disk after each numbered section.

## Result
(filled by executor LAST after disk verify; 3-6 lines: file path, line count, pages fetched OK vs empty-extraction, pattern counts per verdict)
