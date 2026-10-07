# LLM Admin Live-Provider Plan (Phases 3a / 3b / 3c)

> Planner capture of the orchestrator's combined LLM integration plan (user-confirmed "Add it to plan", 23 Sept 2026).
> Source of truth for the failover/breaker pattern: /workspace/multiagent/runs/llm-intel-applykit-careerops.md
> Extends: docs/RESEARCH-llm-generation-spec.md (BE-3, approved) and docs/RESEARCH-llm-integration-comparison.md §6.
> Status: PLANNED (captured verbatim from orchestrator chat — do not re-design).

## 1. Phase 3a — Failover + breaker + quota-after-success (~12-15h builder time, 3 sub-units)

### 1.1 Fallback config
- `LlmSettings` gains `fallbacks: tuple[ProviderConfig, ...]` parsed from `LLM_FALLBACKS` env — JSON list of `{provider, model, base_url, api_key}` entries.

### 1.2 Outer provider loop in LLMService
- Retry budget on the primary provider; on exhaustion (TransportError / 429 / 5xx) advance to the next provider in the chain.
- Usage record already carries a `provider` field — NO schema change.

### 1.3 In-process circuit breaker (per provider)
- 3 consecutive failures → breaker OPEN for 120s → half-open probe → close on success.
- Auth failure (401) trips the breaker instantly.

### 1.4 CRITICAL BUG FIX — quota-after-success
- Move `database.consume(user["id"], 1)` from BEFORE `generate_json` to AFTER a successful return.
- Outage = free retry; the user's quota stays intact.
- Per `db.py` docstring: "Record metered generations AFTER a successful operation".

### 1.5 Failure-mode matrix
| Failure | Behaviour |
|---|---|
| 429 | `QuotaExhaustedError` → failover |
| 5xx / timeout | `TransportError` → retry then failover |
| 401 | breaker trip (instant) |
| 404 "model not found" | failover + admin-visible in usage |
| context-too-long | NO failover (422 — same error class regardless of provider) |
| all providers down | 503 + quota refunded (per 1.4) |

### 1.6 Health surface
- `GET /settings/llm-status` gains a health field computed from `llm_usage`: last-24h success rate + last failure timestamp per provider.
- LlmTab renders 🟢 / 🟡 / 🔴 per provider.

### 1.7 Honest degradation rule (anti-fabrication doctrine)
- In production, NEVER silently mock user-facing generation.
- `evaluate-fit` / `cover-letter` → honest 503 when LLM unavailable.
- `cv` highlights → `real: false` UI label (low-stakes; follows the existing compile_drafts pattern).

### 1.8 Tests (~8)
- Failover path; breaker trip + half-open; quota-refund-on-failure; health aggregation.

### 1.9 Sub-units (builder dispatch shape)
- 3a-1: LlmSettings.fallbacks + LLMService outer provider loop + failure-mode matrix + quota-after-success fix + tests.
- 3a-2: circuit breaker (per-provider, open 120s, half-open, 401 instant) + tests.
- 3a-3: llm-status health aggregation + LlmTab 🟢/🟡/🔴 + tests.

## 2. Phase 3b — Admin Live-Provider Slice 1: READ-ONLY + test probe (~4h)

- Migration 008: `llm_runtime_config` singleton table (id=1 enforced by CHECK, active_provider, active_model, active_base_url, fallback_chain JSONB, forced BOOLEAN, updated_by, updated_at).
- `GET /admin/llm/config` — returns the singleton + supported providers catalog. NO writes.
- `POST /admin/llm/test` — runs a 5-token echo probe with a 10s timeout; costs ~$0; does NOT burn quota; does NOT write llm_usage.
- AdminShell gains an LLM tab (or Platform → LLM sub-tab): renders status, health, supported provider catalog, "Test provider" button.
- Follows the env-gated hide pattern from `services/lemonsqueezy.py` (`LS_ACCOUNT_SETUP_NOT_DONE`).
- `require_admin` on BOTH endpoints.
- ~10 tests.

## 3. Phase 3c — Admin Live-Provider Slice 2: WRITE + AUDIT (~4-6h)

- `PATCH /admin/llm/config` — partial config update (reorder fallback chain, change active provider/model); all-or-nothing semantics.
- `POST /admin/llm/reset` — wipes the singleton row; env-var mode takes over again.
- `llm_admin_audit` append-only log: admin_id, action, before JSONB, after JSONB, ts.
- Concurrency-safe update: singleton pattern + version column.
- Resolution precedence (per request): admin runtime override (when `forced=True`) > env-var mode.
- Force-failover confirmation modal: "no health data on this provider yet" warning when the target provider has no usage history.
- Drag-and-drop fallback chain UI.
- ~8 tests.

## 4. Updated combined timeline (+8-10h slip)

- Phases 1-7 of the original plan unchanged in content/order: LS commit → Groq Wave 1 → Phase 3a → Phases 4-7.
- Phases 3b and 3c inserted after 3a.
- Critical path: UNCHANGED (admin surface is small and scope-bounded).
- No new infra beyond migration 008.

## 5. What this does NOT require (scope anchors)

- NO per-user model picker (user said no).
- NO per-operation model split (discussed, not on roadmap).
- NO DB schema migration beyond migration 008.
- NO new env vars (key management stays Render-side intentionally — LLM_FALLBACKS is the one addition, per Phase 3a; no other env surface grows).
- NO SDK swaps — still httpx + OpenAI-compatible schema.
- NO breaking change to existing endpoints.
- NO Docker / no new infra.

## References
- /workspace/multiagent/runs/llm-intel-applykit-careerops.md (failover/breaker source intel)
- /workspace/platform/docs/RESEARCH-llm-integration-comparison.md §6 (updated by this run)
- /workspace/platform/docs/RESEARCH-llm-generation-spec.md (BE-3)
- /workspace/platform/backend/app/services/llm_service.py, llm_usage.py; app/routers/fit.py, settings.py; app/schemas/llm_fit.py (current wired code)
- /workspace/platform/backend/app/services/lemonsqueezy.py (env-gated hide pattern)

## Result
DONE (planner run 2, 23 Sept 2026). Combined LLM admin Live-Provider plan captured to disk and pushed to origin/main. Files: plan.md (96 lines, 5 sections + References + Result); docs/RESEARCH-llm-integration-comparison.md (Phase A-3a + 3b/3c sections added under §6); docs/RESEARCH-QUEUE.md (entry 10); HANDOVER.md §5 row added. Commit: e9853cd. Resolved after attempt 1 partial-writeback death (files written, commit/push + Result unfilled) per amendment 8h RESUME rule.

Next builder unit (when user says "go"): planner sub-agent dispatches the 3 sub-units of Phase 3a (3a-1 LLMService provider loop+quota-after-success+matrix; 3a-2 circuit breaker; 3a-3 health aggregation+UI badge). All three are backend-only, disjoint file ownership, sequential within the phase (3a-2 depends on 3a-1's loop, 3a-3 depends on 3a-1's quota fix). Build budget: 1800s each per amendment 8f. Resume rule applies if any builder dies mid-unit (amendment 8 RESUME-BY-DEFAULT).

## 6. Addition (24 Sept): launch hygiene — demo-seed policy (todo #9)
Outside the LLM plan's original scope; rides the same run (20260924-1100-3c-seedpolicy). Real signups start clean — ensure_user_seed gates to demo accounts only (@demo.com + u-user/u-pro/u-admin; env DEMO_SEED_ALL bypass for dev/tests). Slots after 3c. Spec: /workspace/platform/tasks/20260924-1100-3c-seedpolicy/spec-9-seed-policy.md

## Result (plan capture)
DONE — 24 Sept, orchestrator INLINE (3 planner dispatches died as model-offline fast-fails: 0 trips, 0 writes each). RESEARCH-QUEUE entry 11 + HANDOVER §5 row + this section added; platform-repo docs committed and pushed (see git log for the docs commit). INFRA-DEC-10 note: Oracle A1 now the listener host of choice — HANDOVER bot-listeners row updated in the same pass.
