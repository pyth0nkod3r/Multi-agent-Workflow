# Task Spec — GROW group pages rebuild (run 20260914-0500-phase4-code)
## Mission
Rebuild 5 GROW-group pages (/interview, /upskill, /expand, /tools, /integrations) so each is wired to its real client methods from 03-builder-client-methods, styled with Clear Deck tokens only (zero hard-coded hex), and carries the role/PRO gating, honest microcopy, and empty/loading/error states per the approved design contract. The builder will rewrite each page file in place. Do NOT touch backend code.

## Context (paths + exact current state)
- `/workspace/platform/DESIGN.md` — Clear Deck token contract (primary #1d4ed8, accent #c2410c, neutral_text #14213d, text_muted #475569, card_bg #ffffff, border_subtle #e2e8f0, mono font JetBrains Mono). All colors are accessed via Tailwind classes (primary, accent, muted-foreground, foreground, border, card, secondary, destructive, success, warning). The active page already uses shadcn token names — DO NOT rename them, just reference the same tokens.
- `/workspace/platform/docs/design/05-ia-menu-spec.md` — §2 sidebar groups: MAIN / GROW / ACCOUNT. §4 route map rows list every backing endpoint. §5b lists new surfaces. §6 hide-entirely policy.
- `/workspace/platform/docs/design/06-stitch-registry.md` — Stitch screen IDs: Interview Prep 125bb1f1, Upskill c75b0073. (Expand, Tools, Integrations use component patterns from the approved token set.)
- `/workspace/multiagent/tasks/20260914-0500-phase4-code/01-builder-tokens.md` — Token migration already applied to index.css/tailwind config. Builder reads it for token name confirmation; do NOT edit token files.
- `/workspace/multiagent/tasks/20260914-0500-phase4-code/02-builder-nav.md` — Nav shell rebuilt with GROW visibility. Builder reads it for layout context.
- `/workspace/multiagent/tasks/20260914-0500-phase4-code/03-builder-client-methods.md` — EXACT client method names to use (use ONLY these):
  - `getInterviewPack(applicationId)`, `mockInterviewTurn(applicationId, answer, state?)`
  - `analyzeUpskill()`
  - `listDocuments()`, `uploadDocument(file, kind)`, `deleteDocument(id)`
  - `getPlans()`
  - `getCompanyResearch(jobId)`
  - `markFollowUpSent(id)`
  - `verifyTemplateCompile(id)`, `testPortal(id, query?)`, `submitPortal(draft)`, `submitChannel(draft)`, `listSubmissions(filters?)`, `approveSubmission(id)`, `rejectSubmission(id, note?)`
  - `register`, `getMe`, `updateMe`
  - `getJobs(filters?)` — `q` param for search
  - `runRank(opts)` — `closingSoon` in response
- Frontend pages directory: `/workspace/platform/frontend/src/pages/`
- Existing pages you will REWRITE in place:
  - `InterviewPrep.tsx` (191 lines, currently imports `mockStarBank`/`mockQuickAnswers` from mock-data)
  - `Upskill.tsx` (149 lines, currently imports `mockUpskillPlan` from mock-data)
  - `Expand.tsx` (133 lines)
  - `Tools.tsx` (275 lines)
  - `Integrations.tsx` (188 lines)
- Existing components you will USE (do NOT rewrite):
  - `AppLayout`, `AppSidebar`, `StatCard`, `JobCard`, `ApplicationCard`, `FormField`, `GenerationPreview`, `NavLink`, `AdminGuard`, `AuthGuard`, `UserProfileForm`, `UsageMeter` — all under `src/components/`
  - shadcn/ui components: `Button`, `Card`, `CardContent`, `CardHeader`, `CardTitle`, `Badge`, `Input`, `Textarea`, `Select`, `Tabs`, `Table`, `Progress`, `Skeleton`, `Alert`, `Toast` (use `useToast`)
- Auth context: `useAuth` from `@/contexts/AuthContext` — exposes `user`, `useGeneration`, `isPro`, `proRole`, etc. (use whatever the existing context exposes for role checks).
- Types from 03: `InterviewPack`, `MockInterviewTurn`, `UpskillAnalyzeResponse`, `UpskillGap`, `UpskillCovered`, `UpskillPlanItem`, `Document`, `DocumentKind`, `Plan`, `PlansResponse`, `CompanyResearch`, `ClosingSoonItem`, `RankRun`, `Submission`, `SubmissionKind`, `TemplateVerifyResult`, `PortalTestResult`.

## Steps

### Step 1 — /interview rebuild per Stitch 125bb1f1
Write `src/pages/InterviewPrep.tsx`:
1. Remove all `mock-data` imports. Import only from `@/lib/api/client`, `@/lib/api/types`, `@/components/`, `lucide-react`, `framer-motion`, and shadcn/ui.
2. Top section: page heading + one sentence explaining this page (honest: "Interview prep packs generated from your tracked applications").
3. **Packs list**: dropdown/select of the user's applications (call `api.getApplications()`, map to a `<select>` or shadcn `Select`). Show application company + role.
4. **Generate pack** (free tier): when user picks an application and clicks "Generate pack", call `api.getInterviewPack(applicationId)`. Show `UsageMeter` alongside (metered). Show loading state. Show `InterviewPack` sections:
   - `likelyQuestions` (list)
   - `starMap` (table: question / suggestedExperience / coverage)
   - `consistencyBrief` (submittedDocuments list + claimsToMatch + cvLanguage)
   - `toughQuestions`
   - `questionsToAsk`
   - `logistics` (format / duration / interviewerFocus)
5. **Honest microcopy**: if `likelyQuestions` is template-based (today it is), show a small muted chip below the pack heading: "Rule-based prep pack — AI-quality content coming." Use `text-muted-foreground` and `Badge variant="secondary"` or a `<span className="text-xs text-muted-foreground">`.
6. **Mock roleplay UI** (PRO-gated): conditionally show "Mock interview" section only if user is pro (`useAuth` role check). Calls `api.mockInterviewTurn(applicationId, answer, state)`. Stateful: maintain `turns[]`, `currentTurn`, `complete`, `asked[]`. Each turn shows interviewerMessage + coaching. Submit answer via textarea → append turn. Show `PRO` chip on the mock section heading. `UsageMeter` on each turn (metered).
7. Empty state: if no applications, show sentence + CTA to `/applications`.
8. Loading/error states per AppLayout pattern.
9. Verify file on disk (`wc -l src/pages/InterviewPrep.tsx`).

### Step 2 — /upskill rebuild per Stitch c75b0073
Write `src/pages/Upskill.tsx`:
1. Remove `mock-data` imports. Import client + types + components.
2. **PRO gate**: entire page gated — if user is not pro, show a locked message with `PRO` chip and a link to `/settings?tab=plan`. Do NOT render the analysis UI for free users.
3. Page heading + description.
4. **Analyze button** → calls `api.analyzeUpskill()`. Show `UsageMeter` (metered). Loading spinner while running.
5. **Results** (`UpskillAnalyzeResponse`):
   - Stats row: `analyzedJobs` count (font-mono).
   - **Gaps** section: `gaps` list with `skill` + `demandCount` (mono numeral).
   - **Covered** section: `covered` list.
   - **Learning plan** section: `plan` items, each with `priority` (font-mono), `skill`, `demandCount`, and `resources` (title + provider + url).
6. **Honest microcopy**: chip near the analyze button: "Gap analysis is rule-based today — smarter recommendations coming."
7. Empty state: before first run, show sentence + analyze CTA.
8. Error state: show `LoadError` or inline error.
9. Verify on disk.

### Step 3 — /expand wiring
Write `src/pages/Expand.tsx`:
1. Import client + types. Use `api.listDocuments()`, `api.uploadDocument(file, kind)`, `api.deleteDocument(id)`.
2. Page heading + description ("Profile enrichment from uploaded documents").
3. **PRO gate**: entire page gated — free users see locked message.
4. **Documents list**: call `api.listDocuments()`, render table/list with `fileName`, `kind`, `sizeBytes` (formatted), `uploadedAt`. Each row has "Delete" button → `api.deleteDocument(id)` → refetch.
5. **Upload form**: `<input type="file">` + kind `<Select>` (cv, linkedin, diploma, other). On submit → `api.uploadDocument(file, kind)` → refetch list. Show `UsageMeter` (metered). Enforce 10MB/file, 50MB/user — surface backend error messages as-is.
6. **Evidence extraction** (new): a "Run extraction" button that calls `api.runExpand()` (already exists in client from 03). Show loading → results. (If 03's `runExpand` returns a list of evidence items, render them.) Each evidence item has an "Apply to profile" button → `api.applyEvidence(evidenceId)`.
7. Empty states for documents list and evidence list.
8. Verify on disk.

### Step 4 — /tools rebuild (AdminGuard removed per 02 spec)
Write `src/pages/Tools.tsx`:
1. Import client + types. Use `api.listSubmissions()`, `api.submitPortal(draft)`, `api.submitChannel(draft)`, `api.verifyTemplateCompile(id)`, `api.testPortal(id, query?)`, `api.approveSubmission(id)`, `api.rejectSubmission(id, note?)`.
2. **PRO gate**: page visible to pro + admin. Admin extras gated with `isAdmin`.
3. **Templates section**: call `api.getCustomTemplates()`. List templates with name + status. Each has "Activate" (`api.activateTemplate(id)` already in client from 03) and "Verify compile" (`api.verifyTemplateCompile(id)`). Show compile result inline (`TemplateVerifyResult`): verified bool, pages, textLayerHealthy, detail, logTail.
4. **Submissions section**: call `api.listSubmissions()`. Show `Submission[]` table: kind, status (badge), submittedBy, submittedAt, payload summary. Each pending item has Approve / Reject actions (pro metered + admin). Approve → `api.approveSubmission(id)` → refetch. Reject → inline note input → `api.rejectSubmission(id, note)` → refetch.
5. **Pro submit flows**: two small forms:
   - Portal draft: label, baseUrl, market, source (cli/websearch), robotsAllowed checkbox → Submit → `api.submitPortal(draft)`.
   - Channel draft: label, platform (telegram/whatsapp), chatRef, notes → Submit → `api.submitChannel(draft)`.
6. **Admin extras** (only for admin): portal sentinel-test. List portals (from `api.getPortals()`), each with "Test" button → `api.testPortal(id, query?)` → show verdict + detail inline.
7. PRO chips on the Tools nav item (layout-level, per 02-nav spec — no need to add chips here; sidebar renders them).
8. Verify on disk.

### Step 5 — /integrations rebuild
Write `src/pages/Integrations.tsx`:
1. Import client + types. Use `api.runGmailSync()`, `api.applyGmailClassification(id)`, `api.getNotionState()`, `api.connectNotion()`, `api.pushToNotion()`.
2. **PRO gate**: entire page gated — free users see locked message.
3. **Gmail section**:
   - "Sync Gmail" button → `api.runGmailSync()` (approval-first: backend returns classifications list, NOT auto-applies).
   - Render classifications list with per-classification "Apply" button → `api.applyGmailClassification(id)`.
   - Show sync status, last-synced timestamp.
4. **Notion section**:
   - State: `api.getNotionState()` — show connected/not-connected status.
   - Connect: `api.connectNotion()` → opens OAuth flow (handle URL or token from response).
   - Push: `api.pushToNotion()` — show last pushed, errors.
5. **Honest microcopy**: chip: "Integrations are opt-in and approval-first. No data leaves your account without your say."
6. `UsageMeter` where metered.
7. Verify on disk.

## Done criteria (grep-able per page)
Run these after the builder finishes:
```bash
# 1. No direct mock-data imports in pages
grep -rn "from.*mock-data" src/pages/ | grep -v "lib/api/mock-data" || true
grep -rn "mockStarBank\|mockQuickAnswers\|mockUpskillPlan\|mockInterviewPack\|mockInterviewTurns\|mockUpskillAnalyze\|mockDocuments\|mockPlans\|mockCompanyResearch\|mockClosingSoon\|mockSubmissions\|mockTemplateVerifyResult" src/pages/ || true

# 2. Each page uses its named client methods (grep for method names)
grep -l "getInterviewPack\|mockInterviewTurn" src/pages/InterviewPrep.tsx
grep -l "analyzeUpskill" src/pages/Upskill.tsx
grep -l "listDocuments\|uploadDocument\|deleteDocument\|runExpand\|applyEvidence" src/pages/Expand.tsx
grep -l "listSubmissions\|submitPortal\|submitChannel\|verifyTemplateCompile\|testPortal\|approveSubmission\|rejectSubmission" src/pages/Tools.tsx
grep -l "runGmailSync\|applyGmailClassification\|getNotionState\|connectNotion\|pushToNotion" src/pages/Integrations.tsx

# 3. No hard-coded hex in the 5 page files (token names only)
grep -rn "#[0-9a-fA-F]\{6\}" src/pages/InterviewPrep.tsx src/pages/Upskill.tsx src/pages/Expand.tsx src/pages/Tools.tsx src/pages/Integrations.tsx || true

# 4. PRO chip / honest microcopy present
grep -n "Rule-based\|AI quality coming\|Rule-based prep pack" src/pages/InterviewPrep.tsx
grep -n "Rule-based\|smarter recommendations coming" src/pages/Upskill.tsx
grep -n "approval-first\|No data leaves" src/pages/Integrations.tsx
```

## Output (files touched)
- `src/pages/InterviewPrep.tsx` — full rewrite
- `src/pages/Upskill.tsx` — full rewrite
- `src/pages/Expand.tsx` — full rewrite
- `src/pages/Tools.tsx` — full rewrite
- `src/pages/Integrations.tsx` — full rewrite

## Depends
- `03-builder-client-methods` (methods must exist before pages call them)
- `01-builder-tokens` (token names + values confirmed)
- `02-builder-nav` (layout shell + role checks)

## Result
**DONE — 5/5 pages rebuilt, all grep Done criteria pass on disk.**

Files (all full rewrites in `frontend/src/pages/`):
- `InterviewPrep.tsx` (358 lines) — application Select via `api.getApplications()` + "Generate pack" → `api.getInterviewPack(appId)`; renders all 7 pack sections (likelyQuestions list, starMap Table, consistencyBrief w/ submittedDocuments+claimsToMatch+cvLanguage, toughQuestions, questionsToAsk, logistics badges); "Rule-based prep pack — AI-quality content coming." microcopy chip under pack heading; PRO-gated mock roleplay section calling `api.mockInterviewTurn(appId, answer, state)` with turns[]/coaching/complete + UsageMeter per spec; empty state → CTA /applications; LoadError + spinner states; UsageMeter next to generate.
- `Upskill.tsx` (208 lines) — whole-page PRO gate (PRO chip + lock + link `/settings?tab=plan`), `api.analyzeUpskill()` w/ useGeneration metering, analyzedJobs mono stat row, gaps/covered sections w/ mono demandCount, learning plan w/ mono priority + resources; "Gap analysis is rule-based today — smarter recommendations coming." chip (empty state + results view); inline error state.
- `Expand.tsx` (354 lines) — PRO gate; documents Table (fileName/kind/sizeBytes fmt/Uploaded) + per-row delete → `api.deleteDocument` → refetch; upload form (native file input + DocumentKind Select: cv/linkedin/diploma/reference/portfolio — client type includes all 5, spec's "other" not a valid kind) → `api.uploadDocument` → refetch, 10MB/file + 50MB/user enforced client-side + backend errors surfaced as-is; "Run extraction" → `api.runExpand` → evidence cards w/ grounded badges + "Apply to profile" → `api.applyEvidence`; conflicts card; UsageMeter; LoadError/spinner/empty states.
- `Tools.tsx` (541 lines) — PRO gate (pro+admin; admin extras gated by isAdmin); Templates w/ Activate + "Verify compile" → `api.verifyTemplateCompile` showing verified/pages/textLayerHealthy/detail/logTail inline; Submissions Table (kind/status badge/submittedBy/submittedAt/payload) w/ Approve → `api.approveSubmission` / Reject w/ inline note → `api.rejectSubmission(id, note)` (useGeneration-metered); portal draft form (label/baseUrl/market/source Select/robotsAllowed checkbox) → `api.submitPortal`; channel draft form (label/platform/chatRef/notes) → `api.submitChannel`; admin portal sentinel-test section via `api.getPortals` + `api.testPortal` w/ verdict+detail+query+checkedAt inline. Old registerPortal/registerTemplate/checkRobots UI replaced.
- `Integrations.tsx` (268 lines) — PRO gate; Gmail "Sync Gmail" → `api.runGmailSync` (approval-first: per-classification "Apply" → `api.applyGmailClassification`, scanned count + last-synced timestamp, empty state); Notion `api.getNotionState`/`connectNotion(token, databaseUrl)`/`pushToNotion` w/ connected badge, masked token, last push, push counters; "Integrations are opt-in and approval-first. No data leaves your account without your say." chip; UsageMeter.

Verification (run from frontend/ 14 Sept ~08:35): all 4 grep groups from the spec pass (no mock-data imports/symbols in pages; method names present per page; zero 6-digit hex in the 5 files; all 3 microcopy greps hit). `tsc --build tsconfig.app.json` exits 1 with 17 PRE-EXISTING errors — none in my 5 files (they are in AppSidebar.tsx [builder-02's], security.ts, AdminDashboard.tsx, Jobs.tsx, Profile.tsx, Setup.tsx). My 5 files type-check clean.

Deviations: (1) Expand upload kind Select uses the client's DocumentKind union (cv/linkedin/diploma/reference/portfolio) instead of spec's "(cv, linkedin, diploma, other)" — "other"/"portfolio" mismatch resolved in favour of the existing API type. (2) Spec's client list named `connectNotion()` bare but actual signature is `connectNotion(token, databaseUrl)` — followed the real signature. (3) InterviewPrep heading uses "AI-quality" (hyphenated) per its grep line; Upskill uses "smarter recommendations coming". (4) PRO chips are in-page on locked states; sidebar-level chips are builder-02's per spec §4.7. No router/layout/sidebar files touched.
