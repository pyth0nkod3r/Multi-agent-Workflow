# RUN — 20261026-1905-shizuku-direct-fallback

Goal: make `shizuku_exec` work on the user's Xiaomi/MediaTek device (Redmi 2409BRN2CG, mt6768,
HyperOS V816, Android 16), where Shizuku's user-service process dies at start-up and every
`shizuku_exec` fails with `shizuku_bind_failed` / `phase: bind_timeout`.

## Diagnosis (verified in this session)

- Shizuku app 13.6.0 installed; server accepts the bind request, the **user service** never hands
  the binder back → `bind_timeout` (not a permission problem: an unauthorised caller fails earlier
  at `bind_threw`).
- Root cause: MediaTek's framework patch inside `LoadedApk.makeApplicationInner` calls
  `Application.getProcessName().equals(...)`; the process name is null in Shizuku's app_process
  user service → NPE → the service dies. Shizuku#1198 (open, unfixed), mechanism quoted by Shizuku
  collaborator vvb2060 in Shizuku#1171#issuecomment-2952634428.
- Same ROM family as the upstream issue's Redmi Note 14 (mt6768).
- rikkahub-agent's own release notes (v2.5.1-agent.1) list this as a known limitation with
  "a fallback is planned" (rikkahub-agent#45).
- Ruled out: harness fault, permissions, battery optimisation, background lock, "Disable child
  process restrictions" (already ON, per user screenshot).

## Workstreams

| Unit | Owner | Status | Output |
| --- | --- | --- | --- |
| spec-01 RikkaHub direct server-process transport + fallback | builder | dispatched (bg 34a004e9) | /workspace/rikkahub-agent |
| spec-02 Shizuku upstream patch (fix the NPE at source) | researcher | dispatched (bg da4c96da) | patch proposal in Result |
| critic review of spec-01 output | critic | pending builder completion | Verdict |

## Verified API facts (for the units)

- `dev.rikka.shizuku:api` 13.1.5. `Shizuku.getBinder()` public;
  `private static ShizukuRemoteProcess newProcess(String[], String[], String)` (reflection route;
  the server AIDL `moe.shizuku.server.*` is not shipped in the client artifact).
- `ShizukuRemoteProcess`: `getInputStream/getOutputStream/getErrorStream`, `waitFor()`,
  `waitForTimeout(long, TimeUnit): Boolean`, `exitValue()`, `destroy()`. Do not use
  `Process.waitFor(long, TimeUnit)` (not overridden → false timeouts).

## Ledger

- 2026-10-26 19:05 — specs written to disk, both units dispatched in background.
