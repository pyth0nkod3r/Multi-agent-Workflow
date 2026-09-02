# ORCHESTRATION — Run Protocol

The main chat assistant is the orchestrator. It never delegates orchestration
decisions; workers execute bounded tasks and report results.

## A. Standard run (large tasks)

1. **Triage** — print the triage block (CONTRACT.md Rule 0).
2. **Plan** — either plan directly or dispatch a `planner` sub-agent. The plan is
   task files: `tasks/<runid>/<NN>-<role>.md`. `<runid>` = `YYYYMMDD-HHMM-<slug>`.
   Each task file is SELF-CONTAINED: a sub-agent will read only that file.
3. **Fan out** — dispatch builders in parallel (one `subagent_dispatch` call per
   unit, issued in a single block). Each dispatch gets:
   - `task`: "Read /workspace/multiagent/tasks/<runid>/<file>.md, execute it
     exactly, write your output where the spec says, REPLACE the ## Result
     placeholder in the file with your actual result."
   - `system_prompt`: the roster prompt for its role
   - `tools`: only what the role needs
   - `label`: `<runid>:<role>-<NN>`
   - Concurrency: ≤3 parallel by default (raise only with user consent; global cap 16).
   - While dispatching, append one line per dispatch to `tasks/<runid>/RUN.md`:
     `timestamp · label · subagent id · status`. Retries get new lines. The run
     must be reconstructable from RUN.md alone (observability / audit trail).
   - Tournament units (CONTRACT Rule 2): dispatch 2–3 variant builders for the
     SAME unit, deliberately different approaches.
4. **Collect** — read each task file's `## Result`. Blocked/failed units are
   re-dispatched once with the failure appended; second failure = escalate.
5. **Critique** — dispatch a `critic` blind: give it the spec paths + output paths,
   never builder identities or the run history. In tournament units the critic
   ranks the variants side by side and names a winner.
6. **Merge** — `merger` (or the orchestrator for small merges) combines into
   `runs/<runid>-final.md` and applies the critic's blockers first.
7. **Report** — to the user: what ran, what the critic said, where the output is.
   Restate actual verification performed. Before reporting, apply the self-rating
   gate (CONTRACT Rule 3): rate the merged result 1–10 against the goal; a "no"
   means fix it first, don't report yet.

## B. Medium tasks

Solo execution, but with one cold critic pass before reporting done — either a
sub-agent critic or a deliberate self-review against the spec, stated explicitly.

## C. Failure design (from MindStudio, adapted)

- **Timeout**: `subagent_dispatch` has `timeout_seconds`; set per unit (default 600).
- **Bad output**: critic FAIL → one revision loop (max 3 total, Rule 4), revision
  task file includes the critic's issues verbatim.
- **Escalation dump**: when the iteration cap trips, write `tasks/<runid>/ESCALATION.md`
  BEFORE escalating — critic issues verbatim, best-attempt paths, what was tried.
  The human (or a future run) resumes from that file; nobody re-burns tokens
  rediscovering the failure.
- **Blocked dependency**: dependent task files carry `Depends: <NN>`; the
  orchestrator holds them until the dependency's `## Result: done` exists.
- **Partial completion**: task files are idempotent — a re-dispatch REPLACES the
  `## Result` section; nothing depends on in-place edits elsewhere.
- **Statelessness**: never assume a worker remembers anything; every dispatch
  re-states the file paths and the format.

## D. State hygiene

- One run = one directory under `tasks/`. Never mutate a completed run's task files
  (append-only corrections in a `## Addendum`).
- `runs/` holds only merged artifacts a human might read.
- After the run: the orchestrator distils durable lessons into memory (memory_tool)
  or `~/learnings/`, and deletes nothing — task files are the audit trail.

## E. Smoke test (grigorev-style verification)

Run when the system is installed or after harness changes:

1. Create `tasks/smoke-<date>/` with three task files: `01-builder.md`,
   `02-builder.md`, `03-builder.md`. Each spec: compute one distinct arithmetic
   result (e.g. 17×23, 2^12−1, sum of primes <30) and write it to
   `tasks/smoke-<date>/out-<NN>.md` with the role name and result.
2. Dispatch the three builders in parallel (single block).
3. Verify all three output files exist with correct arithmetic (orchestrator
   recomputes).
4. Dispatch one `critic` over the three specs + outputs.
5. PASS → record `runs/smoke-<date>-PASS.md`. Any failure → fix the protocol, rerun.

## F. Knowledge & context layer (memory tiers, file-based)

Workers are stateless, so durable context lives in files — three tiers, per the
context-engineering model in the harmonization stack:

- **Project context** — conventions live where the project lives (e.g.
  `/workspace/ai-job-search/RIKKAHUB.md`). Task specs POINT at them, never copy them.
- **Task context** — the task spec itself (self-contained).
- **Collaborative context** — other agents' outputs. Cross-unit dependencies are
  expressed as file paths (`Depends: <NN>`); a dependent worker reads the upstream
  `## Result` straight off disk — no agent-to-agent messaging.

Durable research findings go to `/workspace/multiagent/knowledge/<topic>.md`,
written by researchers; later runs reference them instead of re-researching.
`knowledge/README.md` is the index. Entries stay SHORT — summary + pointers, not
dumps — and a task spec references only the 1–3 files relevant to it. The memory
layer is a filter, not an archive: fetching the few relevant facts beats carrying
the whole history.
