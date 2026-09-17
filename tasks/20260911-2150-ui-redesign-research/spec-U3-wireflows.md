# SPEC U3 — Flow wireframes via pen CLI (run 20260911-2150-ui-redesign-research)

**Executor profile**: designer (design-side). **Output owner**: this file's `## Result` (fill LAST, after disk verify).

## GATE
Do NOT start until BOTH exist on disk:
- /workspace/platform/docs/design/01-functionality-inventory.md
- /workspace/platform/docs/design/02-applydeck-patterns.md

## Context
EaseApply (job-application platform, roles free/pro/admin) is getting a full UI redesign. Phase 1 of the design pipeline = FLOW ONLY, in grayscale — no colors, no fonts, no look decisions. You produce wireflow sheets (pen CLI) covering the key user journeys for desktop AND mobile. These are flow proposals for the user to approve — NOT implementation. You must not touch frontend/backend code.

## Read first
- GOAL.md (esp. unit sketch + key flows list)
- The two gate artifacts above (your flows must match the REAL menu: only items that exist in 01-inventory's KEEP list; applydeck-only features from 02's SKIP list are FORBIDDEN in the flows)
- /workspace/multiagent/knowledge/design-tokens.md only if needed for context (do NOT use colors this phase)

## Tool incantation (verified working)
Run from the output dir via workspace_shell:
```
cd /workspace/platform/docs/design/03-wireflows && source /workspace/.secrets/pen-cli-key.env && unset PEN_AGENT_API_KEY && pen --out <name>.pen --agent codex --model gpt-5.5 --prompt "<prompt>" --export <name>.png
```
- Each pen run takes ~1-3 min. If a run fails with an auth error, retry ONCE after re-running the source command; if it still fails, record the error and continue with remaining runs.
- PEN_AGENT_API_KEY must stay UNSET (it invalidates the ChatGPT OAuth) — the unset above handles it; never `export PEN_AGENT_API_KEY=...`.
- Auth file /root/.pencil/agent-auth valid to ~21 Sept 2026.

## Deliverables (6 pen runs, grayscale wireframe style)
Prompts must: describe EaseApply's REAL functionality per the inventory; request WIREFRAME/grayscale style ("low-fidelity wireframe, grayscale only, no brand colors, focus on layout, hierarchy and flow"); name each screen in the sheet; annotate arrows/flow order.
1. `01-public-onboarding.pen/png` — Landing (value prop per applydeck ADOPT verdicts), Signup, Login, Setup wizard (profile sections). Desktop.
2. `02-core-app.pen/png` — Dashboard (stats), Jobs list + filters, Job detail with Rank score, Apply flow WITH human-review checkpoint before submission. Desktop.
3. `03-tracker-applications.pen/png` — Applications tracker (status board per backend statuses), application detail (status history, follow-up draft), outcome recording. Desktop.
4. `04-secondary.pen/png` — Interview prep, Upskill/Expand, Profile, Settings, Integrations, Billing/plans (per inventory verdicts — only KEEP items). Desktop.
5. `05-admin.pen/png` — Admin console (users management, system stats, portal health) — operator surface only, no job-seeker screens. Desktop.
6. `06-mobile.pen/png` — Mobile (390px-width) equivalents of the core journeys: onboarding, dashboard, jobs+apply w/ checkpoint, tracker, profile/settings — plus a bottom-tab navigation bar reflecting the real menu.

## Verify + checkpoint (after EVERY pen run)
- `ls -la` the output dir; confirm the .pen AND .png both exist and PNG > 30KB (an empty/error export is usually tiny).
- Append a one-line progress note to /workspace/platform/docs/design/03-wireflows/INDEX.md after each run (file, screens covered, size). INDEX.md is yours too.
- Never claim a run succeeded without the ls proof in hand.

## Result
(filled by executor LAST after disk verify of ALL six; 3-8 lines: files + PNG sizes, any runs that failed twice, screen count)
