# ESCALATION.md — 20260907-0405-deploy-prep

## What failed
Four builder sub-agents across two attempts (f777147c, 1d87925e, 48bc43dd,
b4858f08) all reported SUCCEEDED but died pre-writeback:
- attempt-1: zero disk output both units
- attempt-2: backend got only `import os` into main.py (step 1.5/6); frontend zero output
All four had budgets of 1500s and specs with inline context + mandated
incremental Result writes. v2 specs removed discovery reading entirely.

## Analysis
Worker runs on this harness are dying after 1–3 tool calls regardless of spec
hardening. SUCCEEDED status is NOT a completion signal (observed 3×). Disk
state (git status + Result blocks) is the only truth.

## Resolution
Orchestrator executes both units inline (specs 01/02 are fully specified —
execution is mechanical). Blind critic still dispatched at the end; if it dies
pre-verdict, orchestrator completes verification with explicit attribution
(same as 20260906-2140-b2-s5-drafts/02-critic.md).

## Date
2026-09-07 ~05:20 UTC