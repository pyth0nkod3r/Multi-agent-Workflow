# Workspace policy — Multi-Agent Workflow on this machine (Hermes harness)

Ported from the Universal IDE Setup (pyth0nkod3r/Multi-agent-Workflow-Universal-IDE-Setup).
The active agent is the **Universal Orchestrator**. Protocol documents live in this repo:

- `CONTRACT.md` — triage block, roster roles, tournament fan-out, self-rating gate, iteration cap
- `ORCHESTRATION.md` — run protocol: `tasks/<runid>/`, fan-out, watchdogs, recovery, state hygiene
- `knowledge/code-quality.md` — quality gates (14: 1-7 builder floor, 8 zero-day audit, 9 SEC, 10 git-provenance, 11-14 domain gates) + QA commands
- `roster/<role>.md` — canonical role prompts (builder, critic, researcher, planner, designer, merger)

## Harness mapping

This machine runs **Hermes Agent** (desktop app), not RikkaHub/VS Code:

| RikkaHub mechanism | Hermes equivalent |
|---|---|
| `subagent_dispatch(agent=...)` | `delegate_task(tasks=[...])` — parallel children, ONE call, ≤3 concurrent |
| profile systemPrompt prepended | roster body in child `context`, or child reads `roster/<role>.md` first |
| `schedule_job` watchdog (one-shot llm) | `cronjob_manage(action='create', schedule='in 20m')` one-shot, self-contained prompt |
| per-dispatch `model_id` | children inherit parent model |
| in-memory registry (not resumable) | disk checkpoints are the only durable state |

## Paths

Absolute paths in task specs use `C:/Users/D3V_001/repos/Multi-agent-Workflow/...`
(the original `/workspace/multiagent/...` prefix maps here). MSYS bash accepts
`C:/...` and `/c/Users/...` forms. Never rewrite repo files merely to change paths.

## Python

Use `uv` for every Python project, dependency change, and standalone script —
`uv init`, `uv add`, `uv run`. No bare `pip`/`python` unless the user explicitly asks.
uv lives at `C:/Users/D3V_001/.local/bin/uv.exe` (in PATH for new shells).

## Safety

Keep secrets out of source control (env vars / OS credential store). Builders never
`git commit`/`git push` — orchestrator-only commits with `[run <runid>]` markers,
serialized single writer, verify-before-fix.
