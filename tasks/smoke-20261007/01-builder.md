# 01-builder — smoke arithmetic unit (01)
Run: smoke-20261007 | Role: builder | Harness: Hermes

[CONTEXT] This is a smoke test of the multi-agent protocol ported to the Hermes
harness. Repo root: C:/Users/ZION/repos/Multi-agent-Workflow. You work alone;
no conversation with the parent — all reporting through files on disk.

[DELIVERABLE] Compute product of 17 and 23 = 17*23. Use a real computation (terminal/python),
not mental math.

[OUTPUT PATH] C:/Users/ZION/repos/Multi-agent-Workflow/tasks/smoke-20261007/out-01.md
Content: the role name (builder) and the result, one short file, e.g.:
builder: <result>

[DONE CRITERIA] out-01.md exists on disk with the correct numeric result.

[QUALITY] Gates 1-8 from knowledge/code-quality.md: N/A — this unit computes a
number and writes one file; no code is produced. Named explicitly, not silent.

## Result
status: done. Produced tasks/smoke-20261007/out-01.md containing "builder: 391";
17*23 computed via python3 (391), verified against the written file.
Deviations: none. Rating: 10/10 — out-01.md exists on disk with the correct
numeric result. Gates 1-8 N/A, no code produced.
