# 03 — Builder: Fit/Generation + Applications + Rank routers

Contract: `tasks/20260902-0628-fastapi-backend/00-CONTRACT.md` (read first).

## Deliverable
`backend/app/routers/fit.py`, `backend/app/routers/applications.py`, `backend/app/routers/rank.py` — plus tests in `backend/tests/test_fit_applications_rank.py`.

## Endpoints

### fit.py — tag [Fit & Generation]
- `POST /jobs/{jobId}/evaluate-fit` → 200 FitEvaluation. Job from db.jobs[user_id] (404 if missing). Response: {score: job.matchScore, verdict: job.fitVerdict, breakdown: job.scoreBreakdown, gates: job.gates, strengths: [2 bullets derived from job.skills — e.g. f"Core stack overlap: {skills[0]}, {skills[1]}"], gaps: [languageNote if gates.language == "FLAG" else "No blocking language requirement"], recommendation: per verdict bands (>=75 "Strong fit — proceed to drafting.", >=60 "Good fit — apply, address gaps in the cover letter.", >=45 "Moderate fit — consider carefully.", else "Weak fit — probably skip unless strategic.")}.
- `GET /jobs/{jobId}/salary-benchmark` → 200 {company: job.company, found: True, categoryIndex: 108, overallIndex: 104, baselineNote: "Index 100 = median salary, mock baseline"}. (Query param `city` accepted, ignored.)
- `POST /jobs/{jobId}/cover-letter` → 200 {coverLetter: "<3-paragraph template naming {job.title} at {job.company} and the user's name from db.users>"}.
- `POST /jobs/{jobId}/cv` → 200 {cvUrl: "", highlights: [3 strings: f"Tailored skills section emphasizing {skills[0]} and {skills[1]}", "Requirement coverage: every stated requirement addressed — matched or honestly gapped", "Posting's own terms used in section headings where truthfully applicable"]}.
- `POST /jobs/{jobId}/review-drafts` → 200 list of 4 ReviewNote dicts: [{severity:"praise", text:"Opening names the specific role and company — not a template."}, {severity:"suggestion", text:f"Second paragraph claims experience; ground it with one concrete artifact."}, {severity:"suggestion", text:"One posting requirement is unaddressed — bridge or acknowledge honestly."}, {severity:"blocker", text:"No fabricated metrics detected — keep it that way; the Factual Grounding Audit strips unsupported claims."}]. Body has coverLetter/cvHighlights (accepted, drive nothing in mock).
- `POST /jobs/{jobId}/compile` → 200 {cvPdfUrl: "/artifacts/mock-cv.pdf", coverPdfUrl: "/artifacts/mock-cover.pdf", cvPages: 2, coverPages: 1, verificationPassed: True}. (Mock: real lualatex/xelatex compile is a later phase.)

All job lookups: 404 unknown jobId.

### applications.py — tag [Applications]
- `GET /applications` → 200 list; optional `status` query filter.
- `POST /applications` → 201. Body ApplicationCreateRequest: job_id must exist (404). action=="queued" → status "drafted"; action=="send" → status "applied", appliedAt=now. Build the row from the job (customCoverLetter placeholder mentioning title+company, recipientEmail f"hiring@{company.lower()}.com", recipientName "Hiring Manager", channel "email" for send / "portal" for queued, followUpsSent 0). 409 if a row with same jobId exists.
- `PATCH /applications/{id}/status` → 200 updated row. Body StatusUpdateRequest. Set appliedAt/interviewAt/offerAt on first transition; lastContactAt when status is final. 404 unknown id.
- `POST /applications/{id}/outcome` → 200 updated row. Body OutcomeUpdateRequest. Map event→status exactly like client.ts's recordOutcome (applied→applied, interview_invite/interview_stage→interview, offer_received→offer, hired→hired, offer_declined→offer_declined, rejected→rejected, no_response→no_response, withdrawn→withdrawn). notes: append dated entry to row["notesLog"] (list of {at, text}); never overwrite prior entries (idempotent append). interviewAt/offerAt set on first transition only.
- `GET /applications/follow-up-candidates` → 200 rows where status NOT final AND status != "drafted" AND daysQuiet >= 10 AND followUpsSent < 2, each with added "daysQuiet" int. daysQuiet = floor((now - lastContactAt/appliedAt)/86400s); seeded rows a2 (2026-03-05) qualifies via date math — do NOT hardcode.
- `POST /applications/{id}/follow-up-draft` → 200 FollowUpDraft {applicationId, subject (email channel only, f"Re: {job.title} application — {user name}"), body (60-120 words: greeting to contactPerson or "the team", one sentence restating interest in job.title at company, one concrete value sentence referencing the user's production pipeline, one polite timeline question, sign-off with user name), channel: row.channel or "email"}.

### rank.py — tag [Rank]
- `POST /rank` → 200 RankRun. Body RankRequest (scope/focus/topN). Algorithm: candidates = jobs minus (jobIds present in applications) minus (fitVerdict == "poor_fit"); if focus given, filter title/skills containing focus (case-insensitive) — if that filter empties the pool, fall back to unfiltered; sort by matchScore DESC; topN slice (default 5). Build RankedJob dicts: rank 1-based, strengths [f"Core stack overlap: {skills[0]}, {skills[1]}" (guard short lists), "End-to-end production ownership (Pipeline + cube-os agent)"], gaps [languageNote if FLAG else "No blocking language requirement"]. excluded = the removed ones with reasons ("Already tracked in applications" / "Poor fit (32/100) — below the 45 gate"). generatedAt now ISO; scope/focus echoed. Store as db.rank_runs[user_id] (latest only).

## Test file (backend/tests/test_fit_applications_rank.py)
Cover:
- evaluate-fit on job 2 (Helsing, FLAG language): gates.language FLAG, languageNote present, recommendation mentions "Strong fit" (88)
- evaluate-fit 404 on unknown job
- salary-benchmark found=True; cover-letter contains "Helsing"; cv highlights has 3 items; review-drafts returns 4 notes with a blocker; compile returns cvPages=2/coverPages=1
- applications: GET 5 rows; POST queued → status drafted; POST send for a new job → applied + appliedAt; duplicate jobId → 409
- PATCH status drafted→applied sets appliedAt; →rejected sets lastContactAt; unknown id 404
- outcome interview_invite sets interviewAt once (second call doesn't change it); notes appended not overwritten
- follow-up-candidates: computed from dates (assert a2 present or absent per its date — compute expected in the test with the same formula, don't hardcode); drafted row a3 NEVER appears
- follow-up-draft: body word count 60-120 (strip and split), channel respected
- rank: default scope runs, poor_fit ALDB excluded, tracked jobIds excluded, ordering desc, topN respected (topN=2 → 2 rows); focus="python" narrows to matching titles/skills

## Done criteria
- `uv run pytest tests/test_fit_applications_rank.py -q` green
- `uv run python -c "from app.main import app"` OK

## Result
**Status**: DONE (orchestrator-verified — builder reported SUCCEEDED but ended its narration mid-sentence without filling this block; artifacts + full-suite run confirm completion).

**Produced**: routers fit.py / applications.py / rank.py per spec; tests/test_fit_applications_rank.py. One self-caught test-logic fix noted in narration (job 5 already tracked via seeded row a4; expectations corrected to pool 4/6/8).

**Verification (orchestrator)**: full merged suite `uv run pytest -q` → 40 passed (includes builder-01's 21 + builder-03's tests, real app.main — the guarded conftest switched to the full app, confirming all sibling routers import).

**rating**: 9/10 (placeholder left unfilled is the only miss)
