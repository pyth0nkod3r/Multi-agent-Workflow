# Sub-agent profiles — deploy / restore / sync

RikkaHub-agent resolves `subagent_dispatch(agent="<name>")` against profiles
stored app-side (DataStore). This file + `roster/` are the canonical copies.

## Canonical vs deployed
- `roster/<role>.md` body (everything after the `# ` header line) = the exact
  system prompt deployed into the profile of the same name.
- Deployed set (v2, 2026-09-07): builder, critic, researcher, planner, merger. v2 = hard checkpoint/resume rules (checkpoint after every step, <2min foreground commands + background for long jobs, 60% early-stop, resume-by-default).
- When a prompt evolves: edit roster/*.md FIRST, then re-deploy (below), then
  bump the version line in the roster file header. Never edit only one side.

## Deploy / restore (UI automation recipe)
Path: RikkaHub Agent → Chat Options (top bar) → menu drawer → Settings →
scroll → Sub-agent profiles → Add profile.
Fields: Name = role name (lowercase); Description = the one-liner below;
Enabled = on; Model = unset (inherit parent); System prompt = roster body.
Descriptions (paste-ready):
- builder: Multi-agent protocol builder: executes one self-contained task spec (usually under tasks/<runid>/ in the workspace) and reports a Result block. Dispatch by name for all implementation builder work.
- critic: Blind quality gate for multi-agent runs: reviews builder output against its spec's done criteria and returns a Verdict. Dispatch by name after builders report.
- researcher: Web reading and source gathering: fetch, extract, and fact-check from live sources, then report findings. Dispatch by name for research or source-verification work.
- planner: Decomposes one goal into self-contained task spec files under tasks/<runid>/ in the workspace. Dispatch by name to plan a fan-out run; the specs it writes are the builders' contracts.
- merger: Combines parallel multi-agent outputs into one coherent artifact (runs/<runid>-final.md), applying critic blockers first. Dispatch by name at the merge step.

## Verify after deploy
Probe: subagent_dispatch(agent="builder", task="Verification probe... reply
exactly: PROFILOK", timeout_seconds=90, max_trips=2) → expect result PROFILOK.

## Engine facts (source-verified 2026-09-06)
- Profile systemPrompt is PREPENDED to the task text (first user message) —
  SubAgentEngine.kt effectiveTask. NOT a system role.
- The per-dispatch `system_prompt` argument is IGNORED by the engine — never
  use it. One-off personas go in the task text or a new profile.
- `model_id` (per dispatch) wins over the profile's model; profile unset =
  inherit parent.
- timeout_seconds default 300, max 1800; max_trips default 12, max 30.
- Runs are in-memory only (SubAgentRegistry): cascade-cancelled with the
  parent turn, NOT resumable. Continuation = disk state + cheap re-dispatch.
