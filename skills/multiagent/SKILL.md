---
name: multiagent
description: Run a multi-agent workflow on this RikkaHub-agent harness — triage first, plan into task files, fan out sub-agent builders, blind critic pass, merge. Use for large or parallelizable tasks; solo for small ones.
---

# Multi-Agent Workflow

You are the orchestrator. Full rules live in `/workspace/multiagent/`:

1. Read `/workspace/multiagent/CONTRACT.md` — triage block + role roster (system
   prompts and output formats for researcher / planner / builder / critic / merger).
2. Read `/workspace/multiagent/ORCHESTRATION.md` — the run protocol (task files
   under `tasks/<runid>/`, parallel dispatch, critic loop, merge into `runs/`).
3. Apply triage FIRST. Small = solo, never spawn. Large = full protocol.

Quick dispatch shape for a worker:

- task: "Read /workspace/multiagent/tasks/<runid>/<NN>-<role>.md, execute it
  exactly, write output where the spec says, REPLACE the ## Result placeholder
  in the file with your actual result."
- agent: the profile name for the role (builder / critic / researcher / planner
  / merger). Deployed prompts + engine facts: `/workspace/multiagent/PROFILES.md`.
  Do NOT pass system_prompt (ignored by the engine).
- tools: only what the role needs
- label: "<runid>:<role>-<NN>"
- parallel builders go out in ONE block; ≤3 concurrent by default

v2 recovery (ORCHESTRATION §A step 4 + §B2):
- MANDATORY after every dispatch wave: schedule a one-shot watchdog `llm` job
  (longest unit timeout + 5 min, min +15 min) that reads RUN.md + task files
  from disk and re-dispatches cut-off units with resume wording.
- MANDATORY at the start of any user message: scan tasks/ for active runs with
  unfilled ## Result placeholders and recover them BEFORE handling the request.

Hard limits: max 3 critic/revision iterations, then escalate to the user.
Never let a worker decide workflow direction — the orchestrator does that.

Also in CONTRACT.md: tournament fan-out (2–3 variant builders on ONE judgment-heavy
unit, critic ranks blind), the self-rating sanity gate (rate 1–10 before reporting
done), and global worker rules — zero-fluff output, no worker-to-worker chat,
scoped reading, patch discipline. Durable research goes to /workspace/multiagent/
knowledge/ (index in knowledge/README.md). Log every dispatch line to
tasks/<runid>/RUN.md; on iteration-cap escalation, write ESCALATION.md first.
