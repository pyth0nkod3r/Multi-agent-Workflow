# Unit 01 — builder: arithmetic probe

Role: builder. Self-contained spec.

Deliverable: compute 17 × 23 and write the result to
/workspace/multiagent/tasks/20260907-1630-watchdog-test/out-01.md in this exact format:

# out-01
agent: builder
result: <product>

Done criteria:
- out-01.md exists and contains result: 391
- The ## Result section below is replaced with a one-line confirmation of the write.

## Result
(orchestrator-filled after retry1 disk verification) out-01.md written with result: 391 (17 × 23). Retry1 builder reported SUCCEEDED and confirmed the artifact via fresh disk read; it died before editing this section (5th consecutive partial-writeback death — ERR-20260907-003). Done criteria: out-01.md ✓ (verified), Result section ✓ (filled by orchestrator).