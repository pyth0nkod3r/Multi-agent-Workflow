# 02-builder — smoke arithmetic unit (02)
Run: smoke-20261007 | Role: builder | Harness: Hermes

[CONTEXT] This is a smoke test of the multi-agent protocol ported to the Hermes
harness. Repo root: C:/Users/ZION/repos/Multi-agent-Workflow. You work alone;
no conversation with the parent — all reporting through files on disk.

[DELIVERABLE] Compute 2^12 minus 1 = 2**12-1. Use a real computation (terminal/python),
not mental math.

[OUTPUT PATH] C:/Users/ZION/repos/Multi-agent-Workflow/tasks/smoke-20261007/out-02.md
Content: the role name (builder) and the result, one short file, e.g.:
builder: <result>

[DONE CRITERIA] out-02.md exists on disk with the correct numeric result.

[QUALITY] Gates 1-8 from knowledge/code-quality.md: N/A — this unit computes a
number and writes one file; no code is produced. Named explicitly, not silent.

## Result
DONE. Computed 2**12-1 = 4095 via real execution (python), wrote
tasks/smoke-20261007/out-02.md containing "builder: 4095" (verified on disk).
Deviations: none. Rating: 10/10 — done criteria met (out-02.md exists with the
correct numeric result). Gates 1-8 N/A — no code produced, named explicitly per
spec.
