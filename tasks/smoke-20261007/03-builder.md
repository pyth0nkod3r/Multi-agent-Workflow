# 03-builder — smoke arithmetic unit (03)
Run: smoke-20261007 | Role: builder | Harness: Hermes

[CONTEXT] This is a smoke test of the multi-agent protocol ported to the Hermes
harness. Repo root: C:/Users/ZION/repos/Multi-agent-Workflow. You work alone;
no conversation with the parent — all reporting through files on disk.

[DELIVERABLE] Compute sum of all primes strictly below 30 = sum of primes below 30. Use a real computation (terminal/python),
not mental math.

[OUTPUT PATH] C:/Users/ZION/repos/Multi-agent-Workflow/tasks/smoke-20261007/out-03.md
Content: the role name (builder) and the result, one short file, e.g.:
builder: <result>

[DONE CRITERIA] out-03.md exists on disk with the correct numeric result.

[QUALITY] Gates 1-8 from knowledge/code-quality.md: N/A — this unit computes a
number and writes one file; no code is produced. Named explicitly, not silent.

## Result
DONE. Computed sum of primes strictly below 30 via python (sieve of Eratosthenes): primes [2,3,5,7,11,13,17,19,23,29], sum = 129. Wrote C:/Users/ZION/repos/Multi-agent-Workflow/tasks/smoke-20261007/out-03.md containing "builder: 129" (verified on disk, 13 bytes). Deviations: none. Rating: 10/10 — out-03.md exists with the correct numeric result. Gates 1-8 N/A, no code produced (named explicitly per spec).
