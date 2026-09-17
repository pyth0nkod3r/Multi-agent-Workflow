# SPEC U4 — Direction briefs + IA/menu spec (run 20260911-2150-ui-redesign-research)

**Executor profile**: designer (design-side). **Output owner**: this file's `## Result` (fill LAST, after disk verify).

## GATE
Do NOT start until BOTH exist on disk:
- /workspace/platform/docs/design/01-functionality-inventory.md
- /workspace/platform/docs/design/02-applydeck-patterns.md

## Context
EaseApply UI redesign, Phase 1→2 boundary. You produce (a) 2-3 LOOK direction briefs the user will choose from, and (b) the authoritative information-architecture/menu spec that the whole redesign will be built against. The user likes applydeck's UI pattern but wants "the same or better", with menus strictly matching backend reality. Planning ONLY — no code edits, no Stitch generation (that happens after the user picks).

## Read first
- GOAL.md; the two gate artifacts (01 = menu truth; 02 = pattern verdicts)
- /workspace/staging-ai-design/templates/direction-brief.template.md — follow its structure EXACTLY (Product frame; per-direction: base/primary/accent/neutrals, type pairing, vibrancy mechanism, shape/motion signature, Stitch-ready prompt block; Decision section left as "pending user")
- /workspace/multiagent/knowledge/design-tokens.md (token vocabulary)
- EaseApply live-token fallbacks (from GOAL, usable as one direction's palette if it fits): primary #3399ff, accent #17cfa1, bg #0b0d13, card #12151c, text #e7ebef, destructive #df3a3a, warning #f6a823, success #22c373; fonts Outfit / JetBrains Mono.

## Output A — /workspace/platform/docs/design/04-direction-briefs.md
- Fill the template's Product frame for EaseApply (one-liner, audience, 3-5 personality adjectives, anti-personality — ground adjectives in 02's copy-voice findings).
- Directions: offer 2-3, each with a NAME and distinct personality, e.g. one leaning applydeck-inspired ("deck"-style clean productivity), one distinctly EaseApply-own. Each must obey: 1 primary + 1 accent + 2 neutrals MAX; 1 display + 1 body font; explicit vibrancy mechanism (gradient hero / glow / duotone cards / etc.); one shape/motion signature line; a copy-paste Stitch-ready prompt block (per template).
- For EVERY palette: compute WCAG contrast ratios (primary-on-bg, text-on-bg, text-on-card, accent-on-bg) and PRINT the math (4.5:1 body text, 3:1 large text/UI). No ratio below 4.5:1 for body text — adjust the palette until it passes. Show the numbers, don't claim "accessible".
- End with a RECOMMENDED direction + 3-bullet rationale (user still picks; never fewer than 2 options).

## Output B — /workspace/platform/docs/design/05-ia-menu-spec.md
- **Sitemap** — full tree: public (landing/signup/login) → app (all KEEP routes) → admin island.
- **Desktop navigation** — sidebar or top-nav decision (justify via 02's findings + our 15-19 route count), grouped sections, exact labels, icons noted, active-state rule.
- **Mobile navigation** — bottom tab bar (max 5 tabs; overflow → drawer/"More"), same labels, per-tab target routes.
- **Route map table** — Menu item | Route (matches frontend route names in 01) | Backing endpoint(s) (names from 01) | Role visibility (free/pro/admin) | Notes. EVERY row's endpoints must exist in 01-inventory. Zero nav items without endpoints.
- **Explicitly excluded** — DEFER/REMOVE items from 01 + applydeck SKIP list from 02, each 1-line why (this is the leakage guard documentation).
- **Empty/deferred-feature policy** — where the nav should surface "coming soon" vs hide entirely (recommendation: hide entirely; note exceptions).

## Constraints
- Menu↔backend 1:1 — the critic will grep every nav row against openapi.yaml; inventing an endpoint = automatic fail.
- sRGB/hex only in briefs (no oklch/P3).
- Checkpoint after each output file's major section.

## Result
(filled by executor LAST after disk verify of BOTH outputs; 3-6 lines: two file paths + line counts, direction names offered, contrast-math pass confirmation per direction)
