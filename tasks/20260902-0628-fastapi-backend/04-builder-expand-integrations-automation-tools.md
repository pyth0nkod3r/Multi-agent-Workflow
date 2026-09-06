# 04 — Builder: Expand + Integrations + Automation + Tools routers

Contract: `tasks/20260902-0628-fastapi-backend/00-CONTRACT.md` (read first).

## Deliverable
`backend/app/routers/expand.py`, `backend/app/routers/integrations.py`, `backend/app/routers/automation.py`, `backend/app/routers/tools.py` — plus tests in `backend/tests/test_expand_integrations_tools.py`.

## Endpoints

### expand.py — tag [Expand]
- `POST /expand/run` → 200 ExpandRun. Build deterministically: scanned = ["cv/Miracle_Anyanwu_Resume_AUG_2026.pdf", "linkedin/Miracle_Anyanwu_Portfolio.pdf", "projects/Podcast_Downloader_Pipeline_Capstone_Report.pdf"]; evidence = 4 items mirroring client.ts mock (id e1-e4: the 80% bundle cut achievement grounded→experience, the Podcast Pipeline project grounded→projects, AI DevTools Zoomcamp cert grounded→certifications, Twitter-handle correction ungrounded→experience); conflicts = []. Store as db.expand_runs[user_id].
- `POST /expand/evidence/{id}/apply` → 200 {ok: true}. 404 unknown evidence id (check against the stored run; if no run yet, 404 "Run /expand first").

### integrations.py — tag [Integrations] (ALL admin-only — require_admin; the frontend AdminGuards this page)
- `POST /integrations/gmail/run` → 200 GmailSyncRun: scanned 46, scannedAt now, classified = 3 items mirroring client.ts mock (g1 SumUp interview_invite 0.97 suggestedStatus interview applied False; g2 ALDB rejection 0.88 → rejected; g3 Plato unclear 0.52 → applied). Store db.gmail_runs[user_id]. On run, ALSO actually update matching tracker rows' status for g1/g2 (match application rows by job.company) — that's the sync's real behavior; set applied=True on those classifications.
- `POST /integrations/gmail/classifications/{id}/apply` → 200 {ok: true, newStatus}. Applies the suggested status to the matched tracker row (find via stored run). 404 unknown id or no run.
- `GET /integrations/notion/state` → 200 db.notion_state[user_id].
- `POST /integrations/notion/connect` → 200 updated state: connected True, tokenMasked f"secret_{token[:8]}••••••••", databaseUrl set. Body NotionConnectRequest. Never store or return the raw token.
- `POST /integrations/notion/push` → 200 state with lastPushedAt now, pushedJobs = len(jobs), pushedApplications = len(applications). If not connected: 409 "Notion not connected".

### automation.py — tag [Automation]
- `GET /automation/config` → 200 config. NOTE: spec marks config GET as reachable by all users (settings:read) but PATCH admin-only — match that: GET uses get_current_user, PATCH require_admin.
- `PATCH /automation/config` — require_admin. Body AutomationConfig (full object, camelCase). Validate platform in ("lindy","n8n") via schema. Store per-user, return it.
- `POST /automation/test-connection` — require_admin. → 200 {success: True, message: "Connection successful. Webhook responded with 200 OK."} if config webhookUrl is a well-formed http(s) URL else {success: False, message: "Invalid webhook URL"}.

### tools.py — tag [Tools]
- `POST /tools/robots-check` — require_admin. Body RobotsCheckRequest. → 200 {allowed, detail}: disallow when baseUrl contains "irishjobs.ie" or matches r"linkedin\.com/feed" (detail: "robots.txt disallows this path for generic crawlers — the portal skill would be WebSearch-fallback only."), else allow ("robots.txt permits crawling of listing paths with an honest identifying User-Agent.").
- `POST /tools/portals` — require_admin. Body PortalRegisterRequest. → 201 Portal dict {id, label, market, enabled: True, source, notes: "Honest UA; robots-checked at registration." if robots_allowed else "WebSearch fallback only — robots disallows direct crawling."}. Append to db.registered_portals (409 if id already exists there or in globals).
- `GET /tools/templates` → 200 db.templates + db.custom_templates (both lists merged).
- `POST /tools/templates` — require_admin. Body TemplateRegisterRequest. → 201 CustomTemplate {id: f"tpl-{n}", active: False, registeredAt: now, ...body camelCase}. Append to custom_templates.
- `POST /tools/templates/{id}/activate` — require_admin. → 200 merged list. Sets active: the targeted id active=True for its kind, all other templates of the SAME kind active=False. 404 unknown id.

## Test file (backend/tests/test_expand_integrations_tools.py)
Cover:
- expand run: 4 evidence items, e4 ungrounded; apply e1 → ok; apply unknown → 404; apply before run → 404
- gmail run (as admin): classified has 3 rows, g1/g2 applied=True, and the SumUp tracker row's status became "interview"; classification apply on g3 → newStatus "applied"; user_headers → 403
- notion: initial state connected False; connect → tokenMasked starts "secret_" and raw token NOT in response; push before connect → 409; push after connect → pushedJobs == len(jobs)
- automation: GET as user 200; PATCH as user 403; PATCH as admin persists platform "lindy"; test-connection admin → success True
- robots-check: irishjobs.ie → allowed False; https://www.welcometothejungle.com → True; user → 403
- tools portals: register new → 201 enabled True; duplicate id → 409; GET /portals (from builder 02's router) includes it — import TestClient only, no cross-router import
- templates: GET has 2 stock; register cover template → 201 inactive; activate it → its active True AND stock cover (tpl-2) False; unknown id 404

## Done criteria
- `uv run pytest tests/test_expand_integrations_tools.py -q` green
- `uv run python -c "from app.main import app"` OK

## Result
**Status**: DONE (orchestrator-verified — r1 timed out with nothing produced; r2 completed all four routers + tests).

**Produced**: routers expand.py / integrations.py / automation.py / tools.py per spec (gmail sync applies classifications to tracker rows; notion push 409-before-connect; robots-check disallows irishjobs.ie; template activation one-active-per-kind) + tests in test_expand_integrations_tools.py.

**Verification (orchestrator)**: all four compile; merged suite 59/59.

**rating**: 9/10 (Result block unfilled — filled here from verification)

## RE-DISPATCH NOTICE (orchestrator)
First attempt: TIMED_OUT at 900s having produced NOTHING (no router files, no tests, Result unfilled). This is attempt 2. Work file-by-file and verify each with `uv run python -m py_compile app/routers/<file>.py` as you go — do not batch all four routers before any verification.
