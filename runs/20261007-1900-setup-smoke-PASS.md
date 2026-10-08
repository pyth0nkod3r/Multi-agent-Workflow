# Smoke PASS — smoke-20261007

**Date:** 7 Oct 2026 · **Harness:** Hermes Agent v0.21.5 (Windows, desktop) · **Result:** PASS

This run verifies the Multi-Agent Workflow port (pyth0nkod3r/Multi-agent-Workflow
+ Universal-IDE-Setup mirror) works on the Hermes harness.

## What ran

| Unit | Dispatch | Result |
|---|---|---|
| builder-01 (17×23) | delegate_task, 3-wide parallel wave | out-01.md `computed: 391` ✓ |
| builder-02 (2^12−1) | delegate_task, 3-wide wave → **FAILED** (HTTP 429, 15 req/min provider cap, 59.9s) → re-dispatch r1 solo | out-02.md `computed: 4095` ✓ |
| builder-03 (primes<30) | delegate_task, 3-wide parallel wave | out-03.md `computed: 129` ✓ |
| critic (blind) | delegate_task, after all units disk-verified | Verdict: PASS, 0 issues — CRITIC.md |

## Verification performed

- Orchestrator recomputed all three results via Python: 391, 4095, 129 — all match.
- Every output file read from disk: exact 3-line format, correct values.
- All three spec `## Result` placeholders replaced by the builders (RESULT-LAST held).
- Blind critic independently recomputed all three and verified formats (CRITIC.md,
  read back from disk — full verification table, 9/9 checks PASS).

## Lessons recorded during this run

1. **429 rate limit**: the provider caps at 15 req/min; 3 parallel delegate_task
   children + the parent session can trip it. Keep waves ≤2 wide, stagger, or
   re-dispatch failed units after the wave drains (protocol re-dispatch worked).
2. **Roster prepend mapping works**: children received the roster body in their
   `context` field and followed it (zero-fluff output, RESULT-LAST, Result fills).
3. **`hermes mcp add` interactive save breaks over a pty** — non-interactive
   `hermes config set mcp_servers.<name>.url ... --force` is the reliable path.
4. **Git identity was unset on this machine** — set repo-local from the repo's
   own provenance (pyth0nkod3r, noreply email) before any orchestrator commit.
