# Multi-Agent Workflow — RikkaHub-agent Edition

A multi-agent operating protocol for the RikkaHub-agent harness (ExTV fork, full agent mode),
adapted 2 Sept 2026 from four sources, each contributing one layer:

| Layer | Source | Contribution |
|---|---|---|
| Contract — when to spawn at all | jbarbier/CLAUDE.md | Triage block; fan-out + harsh critic; self-rating loop |
| Architecture, roles & memory | Vedant Parmar (Multi-Agent Channel) | Role roster; PRP plan-before-execute; context tiers (project/task/collaborative); iteration caps |
| Execution — orchestrator mechanics | MindStudio guide | Orchestrator/worker; disk task queue; review gates; failure design |
| Distribution — persistent setup | alexeygrigorev/.agents | Version-controlled configs; shared skills; parallel smoke test |

## Core translation decision

RikkaHub-agent has no resident orchestrator daemon. **The orchestrator is the main chat
assistant; workers are stateless sub-agents created via `subagent_dispatch`.** Because
workers cannot see the orchestrator's conversation, the only reliable channel is
**files on disk**:

- Task input: written to `tasks/<runid>/<NN>-<role>.md` (self-contained spec)
- Task output: sub-agent replaces the `## Result` placeholder in the same file
- Orchestrator reads results, merges into `runs/`; dispatches logged to RUN.md

This replaces MindStudio's "task queue + shared state database" with a git-friendly
markdown queue — same mechanics, zero infrastructure.

## Files

- `CONTRACT.md` — triage rules and the agent roster (read first)
- `ORCHESTRATION.md` — the run protocol step by step
- `tasks/` — per-run task files
- `runs/` — merged final outputs
- `skills/multiagent/SKILL.md` — thin pointer installed into the harness so any chat can invoke the protocol

## Verify

Run the smoke test per `ORCHESTRATION.md §Smoke test`: three parallel builders each
write a task file, a critic reviews blind, orchestrator merges. Success = 3 valid
files + critic PASS + merged run file.
