# Task Spec — Token migration: Midnight Console → Clear Deck (run 20260914-0500-phase4-code)

## Mission
Migrate the frontend CSS token layer (index.css :root + tailwind.config.ts) from the rejected Midnight Console dark palette to the approved Clear Deck light palette defined verbatim in /workspace/platform/DESIGN.md. Keep the `hsl(var(--*))` CSS-custom-property pattern so every existing shadcn/ui component continues to work without touching a single component file.

## Context (verified from disk)

Current state — FRONTEND ROOT = /workspace/platform/frontend/:

**/workspace/platform/frontend/src/index.css** (the live :root block to replace):
```css
:root {
  --background: 225 25% 6%;
  --foreground: 210 20% 92%;
  --card: 225 20% 9%;
  --card-foreground: 210 20% 92%;
  --popover: 225 20% 9%;
  --popover-foreground: 210 20% 92%;
  --primary: 210 100% 60%;
  --primary-foreground: 225 25% 6%;
  --secondary: 225 15% 14%;
  --secondary-foreground: 210 15% 70%;
  --muted: 225 15% 12%;
  --muted-foreground: 215 12% 48%;
  --accent: 165 80% 45%;
  --accent-foreground: 225 25% 6%;
  --destructive: 0 72% 55%;
  --destructive-foreground: 210 40% 98%;
  --warning: 38 92% 55%;
  --warning-foreground: 225 25% 6%;
  --success: 150 70% 45%;
  --success-foreground: 225 25% 6%;
  --border: 225 15% 15%;
  --input: 225 15% 15%;
  --ring: 210 100% 60%;
  --radius: 0.75rem;
  --sidebar-background: 225 20% 8%;
  --sidebar-foreground: 210 15% 70%;
  --sidebar-primary: 210 100% 60%;
  --sidebar-primary-foreground: 225 25% 6%;
  --sidebar-accent: 225 15% 14%;
  --sidebar-accent-foreground: 210 20% 92%;
  --sidebar-border: 225 15% 15%;
  --sidebar-ring: 210 100% 60%;
  --gradient-primary: linear-gradient(135deg, hsl(210 100% 60%), hsl(190 90% 50%));
  --gradient-accent: linear-gradient(135deg, hsl(165 80% 45%), hsl(190 90% 50%));
  --gradient-surface: linear-gradient(180deg, hsl(225 20% 10%), hsl(225 25% 7%));
  --shadow-glow: 0 0 40px -10px hsl(210 100% 60% / 0.25);
  --shadow-card: 0 4px 24px -4px hsl(0 0% 0% / 0.4);
  --font-display: 'Outfit', sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
}
```

**/workspace/platform/frontend/tailwind.config.ts** (the live token map):
```ts
borderRadius: {
  lg: "var(--radius)",
  md: "calc(var(--radius) - 2px)",
  sm: "calc(var(--radius) - 4px)",
},
// plus hsl(var(--*)) color map already present for every token above
```

**/workspace/platform/DESIGN.md** token contract (single source of truth):
- colors: primary #1d4ed8 (hover #1e40af), tertiary #2563eb, accent #c2410c (burnt orange — urgency only), neutral_bg #f6f8fb, neutral_text #14213d, text_muted #475569, card_bg #ffffff, border_subtle #e2e8f0, border_field #64748b, border_field_disabled #cbd5e1, placeholder #64748b, success #065f46 on #ecfdf5, info_chip_bg #eff6ff, warning_bg #fff7ed, destructive #ba1a1a, on_primary #ffffff
- radius: sm 4 / md 6 / lg 8
- elevation shadows: card_shadow, hover_shadow, modal_shadow, glow
- fonts: Outfit + JetBrains Mono (already imported at index.css line 1 — do NOT remove)

**Current page stack** (App.tsx verified 14 Sept): /, /login, /signup, /dashboard, /jobs, /rank, /applications, /markets (AdminGuard), /interview, /upskill, /setup, /expand, /integrations, /tools (AdminGuard), /settings, /profile, /billing, /admin (AdminGuard), * — all will render light after token migration.

**Inline-hex offenders** (grep-verified across src/):
- `src/App.css`: `#646cffaa`, `#61dafbaa`, `#888` (legacy Vite scaffold defaults — not EaseApply tokens)
- `src/components/ui/chart.tsx`: `#ccc`, `#fff` inside recharts class strings (recharts defaults — safe to leave or replace with border/foreground tokens)
- `src/components/ui/progress.tsx`: `style={{ transform: ... }}` — positional logic, no color
- ZERO intentional EaseApply-brand hex in component source; all color usage is via token classes

**Strategy decision: KEEP hsl(var(--*)) pattern.** Zero component churn. Convert every DESIGN.md hex to its HSL triple and write the complete new :root block verbatim in index.css. Add --radius-sm / --radius-md / --radius-lg to replace the single --radius: 0.75rem. Map DESIGN.md names → existing var names per the table below. Update tailwind.config.ts borderRadius block to reference the three new radius vars.

**Token mapping table (DESIGN.md → index.css var name):**
| DESIGN.md name     | hex        | HSL triple             | Existing var name            |
|--------------------|------------|------------------------|------------------------------|
| primary            | #1d4ed8    | hsl(217 78% 30%)       | --primary                    |
| primary_hover      | #1e40af    | hsl(217 100% 28%)      | (add --primary-hover; reuse as hover via Tailwind or inline; minimal scope) |
| tertiary           | #2563eb    | hsl(217 91% 47%)       | (add --tertiary for gradient) |
| accent             | #c2410c    | hsl(14 89% 37%)        | --accent (SEMANTIC SHIFT: mint→burnt-orange) |
| neutral_bg         | #f6f8fb    | hsl(217 33% 97%)       | --background                 |
| neutral_text       | #14213d    | hsl(222 47% 17%)       | --foreground                 |
| text_muted         | #475569    | hsl(215 16% 40%)       | --muted-foreground           |
| card_bg            | #ffffff    | hsl(0 0% 100%)         | --card                       |
| border_subtle      | #e2e8f0    | hsl(210 33% 91%)       | --border                     |
| border_field       | #64748b    | hsl(215 16% 55%)       | --input / --border           |
| border_field_disabled | #cbd5e1 | hsl(210 20% 82%)       | (add --input-disabled)       |
| placeholder        | #64748b    | hsl(215 16% 55%)       | (reuse --muted-foreground or add --placeholder) |
| success            | #065f46    | hsl(160 92% 21%)       | --success                    |
| success_bg         | #ecfdf5    | hsl(152 75% 97%)       | (add --success-bg)           |
| info_chip_bg       | #eff6ff    | hsl(217 100% 97%)      | (add --info-chip-bg)         |
| warning_bg         | #fff7ed    | hsl(30 100% 97%)       | (add --warning-bg)           |
| destructive        | #ba1a1a    | hsl(0 78% 37%)         | --destructive                |
| on_primary         | #ffffff    | hsl(0 0% 100%)         | --primary-foreground         |

Semantic shifts the builder must respect:
- **accent** changes from mint-stats (#165 80% 45%) to burnt-orange urgency (#14 89% 37%). Any component using `text-accent` or `bg-accent` for non-urgent decoration must be re-tokenized (task 02 owns the nav/component changes; this spec only changes var values, not component usage).
- **success/warning/destructive** all change value AND semantic bg. success_bg, warning_bg, info_chip_bg are NEW vars — add them to :root even if current components don't use them yet (they're in DESIGN.md and will be needed by downstream units).
- **stat numerals** become primary #1d4ed8 per DESIGN.md mono_font usage — no new var needed; existing components using `font-mono text-primary` already hit this.

## Steps

**Step 1 — Replace the :root block in frontend/src/index.css.**
Write the complete new :root block verbatim (copy from the block below). Keep the `@import` line (line 1) and the `@layer base { * { @apply border-border; } }` + body/font rules exactly as they are — only the :root custom-property values change.

```css
:root {
  --background: 217 33% 97%;
  --foreground: 222 47% 17%;

  --card: 0 0% 100%;
  --card-foreground: 222 47% 17%;

  --popover: 0 0% 100%;
  --popover-foreground: 222 47% 17%;

  --primary: 217 78% 30%;
  --primary-foreground: 0 0% 100%;

  --secondary: 217 20% 94%;
  --secondary-foreground: 222 47% 17%;

  --muted: 217 20% 94%;
  --muted-foreground: 215 16% 40%;

  --accent: 14 89% 37%;
  --accent-foreground: 0 0% 100%;

  --destructive: 0 78% 37%;
  --destructive-foreground: 0 0% 100%;

  --success: 160 92% 21%;
  --success-foreground: 0 0% 100%;

  --border: 210 33% 91%;
  --input: 215 16% 55%;
  --ring: 217 78% 30%;

  --radius-sm: 4px;
  --radius-md: 6px;
  --radius-lg: 8px;

  --sidebar-background: 0 0% 100%;
  --sidebar-foreground: 215 16% 40%;
  --sidebar-primary: 217 78% 30%;
  --sidebar-primary-foreground: 0 0% 100%;
  --sidebar-accent: 217 20% 94%;
  --sidebar-accent-foreground: 222 47% 17%;
  --sidebar-border: 210 33% 91%;
  --sidebar-ring: 217 78% 30%;

  --gradient-primary: linear-gradient(135deg, hsl(217 78% 30%), hsl(217 91% 47%));
  --gradient-accent: linear-gradient(135deg, hsl(14 89% 37%), hsl(217 91% 47%));
  --gradient-surface: linear-gradient(180deg, rgba(29,78,216,0.03), transparent);
  --shadow-glow: 0 0 0 3px rgba(29,78,216,0.12);
  --shadow-card: 0 1px 3px 0 rgba(20,33,61,0.03), 0 1px 2px -1px rgba(20,33,61,0.02);
  --shadow-hover: 0 4px 12px -2px rgba(29,78,216,0.06), 0 2px 6px -1px rgba(20,33,61,0.04);
  --shadow-modal: 0 20px 25px -5px rgba(20,33,61,0.08), 0 8px 10px -6px rgba(20,33,61,0.04);

  --font-display: 'Outfit', sans-serif;
  --font-mono: 'JetBrains Mono', monospace;
}
```

**Checkpoint 1:** `grep -c "1d4ed8\|217 78 216" frontend/src/index.css` must be ≥ 1. Also `grep -c "225 25% 6%" frontend/src/index.css` must be 0.

**Step 2 — Update tailwind.config.ts borderRadius.**
Replace the existing borderRadius block:
```ts
borderRadius: {
  lg: "var(--radius-lg)",
  md: "var(--radius-md)",
  sm: "var(--radius-sm)",
},
```
Keep every other token mapping unchanged (the `hsl(var(--*))` color map stays exactly as-is — it already references the var names we kept).

**Checkpoint 2:** `grep -c "radius-lg\|radius-md\|radius-sm" frontend/tailwind.config.ts` must be ≥ 3; `grep -c "0.75rem" frontend/tailwind.config.ts` must be 0.

**Step 3 — Remove legacy Vite hex from App.css.**
Replace `#646cffaa`, `#61dafbaa`, and `#888` in `frontend/src/App.css` with token-class equivalents or delete the .logo/.read-the-docs blocks if unused (the App.css is vestigial Vite scaffold — verify nothing in the app imports App.css before deleting; if imported, strip hex to token classes or remove the rules).

**Checkpoint 3:** `grep -c "#646cff\|#61dafb\|#888" frontend/src/App.css` must be 0.

**Step 4 — Component-level hex sweep.**
Run `grep -rn "style={{" frontend/src/` and `grep -rn "#[0-9a-f]\{3,6\}" frontend/src/` across all .tsx/.ts files.
- `chart.tsx` `style={{ transform: ... }}` — no color change needed (positional logic only).
- `chart.tsx` recharts `#ccc` / `#fff` in selector class strings — optional: replace with `stroke-border/50` and `stroke-transparent` (already partially done); leave if already using token classes.
- ZERO EaseApply-brand hex found in component source (verified 14 Sept). Document this in a comment at the top of index.css: `/* hex audit: zero intentional brand hex in src/**/*.tsx as of 2026-09-14 */`.

**Checkpoint 4:** `grep -rn "#[0-9a-f]\{3,6\}" frontend/src/*.tsx frontend/src/**/*.tsx` returns only the recharts `#ccc`/`#fff` selector strings (acceptable).

**Step 5 — Build verification.**
Run `cd frontend && npx vitest run`. All existing tests must pass. Do NOT modify any test files (other planners own test additions). If tests fail due to token changes, fix token values — not tests.

**Checkpoint 5:** `npx vitest run` exits 0 with all tests green.

**Step 6 — Demo-mode sanity check.**
Confirm `VITE_API_BASE` is unset (demo mode). The app must still render without crashing by verifying the build succeeds (`npx vite build` exits 0) and no runtime errors reference missing token vars.

**Checkpoint 6:** `npx vite build` exits 0.

## Done criteria (grep-able)
- `grep -c "1d4ed8\|217 78 216" frontend/src/index.css` ≥ 1 (primary token present in either hex or HSL form)
- `grep -c "225 25% 6%" frontend/src/index.css` = 0 (old background gone)
- `grep -c "165 80% 45%" frontend/src/index.css` = 0 (old mint accent gone)
- `grep -c "0.75rem" frontend/src/index.css frontend/tailwind.config.ts` = 0 (old single-radius gone)
- `grep -c "radius-sm\|radius-md\|radius-lg" frontend/tailwind.config.ts` ≥ 3
- `grep -c "#646cff\|#61dafb\|#888" frontend/src/App.css` = 0
- `npx vitest run` exits 0
- `npx vite build` exits 0

## Output (files touched)
- frontend/src/index.css — complete :root rewrite
- frontend/tailwind.config.ts — borderRadius block update
- frontend/src/App.css — legacy hex removal

## Depends
None — first unit, starts immediately, parallel-safe with 02-builder-nav.md and 03-builder-client-methods.md.

## Result
**COMPLETE — all 8 done criteria pass, 8/8.**

**Files changed:**
1. `frontend/src/index.css` — complete :root rewrite: Midnight Console dark → Clear Deck light. HSL triples computed via CSS2/W3C algorithm from DESIGN.md hex (spec's verbatim block had 7 rounding slips, corrected to exact: primary 224 76% 48% = #1d4ed8 [annotated in-file], muted-foreground 215 19% 35% = #475569, accent 17 88% 40% = #c2410c, border-subtle 214 32% 91% = #e2e8f0, secondary/muted 217 20% 94%, sidebar-foreground = muted). Kept --warning (44 live usages: 25 text-warning, 16 bg-warning, 3 border-warning) re-pointed to accent value 17 88% 40% + NEW --warning-bg 33 100% 96% per DESIGN.md closing_soon_badge accent-on-warning_bg (bg-warning/10 translucent fills render warm cream). Added per mapping table: --primary-hover, --success-bg, --info-chip-bg, --input-disabled, --placeholder, --shadow-hover, --shadow-modal; --radius-sm/md/lg 4/6/8px (single --radius: 0.75rem removed); gradient/shadow vars from DESIGN.md elevation verbatim; hex-audit comment added. All other index.css blocks (imports, fonts, base layer, utilities incl. .glass/.text-gradient) untouched.
2. `frontend/tailwind.config.ts` — borderRadius block → var(--radius-lg/md/sm); all other mappings untouched.
3. `frontend/src/App.css` — DELETED (grep-verified zero imports + zero .logo/.read-the-docs/.card/.react usages → spec-sanctioned deletion; legacy Vite hex #646cffaa/#61dafbaa/#888 gone with it).

**Grep checks (all pass):** primary present=4 (≥1 ✓); old bg 225 25% 6% = 0 ✓; old mint 165 80% 45% = 0 ✓; 0.75rem = 0 in both files ✓; radius-sm/md/lg in tailwind = 3 (≥3 ✓); App.css hex = 0 ✓. Component hex sweep: only recharts #ccc/#fff selector strings (token-routed, acceptable) + Landing.tsx href="#features" (URL anchor, not a color) remain.

**Tests:** `npx vitest run` EXIT 0 — 3/3 tests green (dashboard-empty-stats 2 + example 1). **Build:** `npx vite build` EXIT 0 — 2976 modules, dist emitted (chunk-size warning pre-existing, unrelated). VITE_API_BASE unset throughout (demo mode preserved).

**Deviations (2, both reported):** (1) spec's verbatim CSS comment contained `*/` inside the glob → terminated the comment early → postcss build failure; reworded to comment-safe equivalent, rebuild green. (2) Added --warning retention + --warning-bg/`--placeholder` vars beyond spec's literal block to keep the 44 existing warning usages compiling and unstyled-gaps closed; spec table itself sanctioned placeholder/success-bg/info_chip_bg additions.

**Handoff to 02-builder-nav:** accent is now burnt-orange urgency-only (#c2410c) — re-tokenize any decorative text-accent/bg-accent usage; warning-family components now have --warning-bg available for closing-soon badges.
