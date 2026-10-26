# Spec 02 — Shizuku upstream: patch the MediaTek/Xiaomi user-service start-up NPE

Run id: 20261026-1905-shizuku-direct-fallback
Owner: researcher sub-agent. Output: a patch-ready proposal, no code committed to any repo.

## Goal

`Shizuku` is open source (https://github.com/RikkaApps/Shizuku, API + server code in
https://github.com/RikkaApps/Shizuku-API). The user-service process fails to start on
Xiaomi/MediaTek ROMs, which is the root cause of `shizuku_bind_failed` / `phase: bind_timeout`
in rikkahub-agent (issue #45). Fixing it at the source helps every Shizuku consumer, not just
RikkaHub.

## Verified facts

- Upstream issue: https://github.com/RikkaApps/Shizuku/issues/1198 — "User services don't work on
  MediaTek devices", opened by PookaMustard, still unresolved in the current release (13.6.0).
- The device in question: Redmi `2409BRN2CG`, `ro.board.platform=mt6768`, `ro.soc.manufacturer=Mediatek`,
  `ro.soc.model=MT6769`, HyperOS "V816", Android 16 (SDK 36). Same platform family as the
  issue's Redmi Note 14.
- MediaTek's framework patch inside `LoadedApk.makeApplicationInner` (quoted by Shizuku collaborator
  vvb2060 in https://github.com/RikkaApps/Shizuku/issues/1171#issuecomment-2952634428):

  ```java
  if (!this.mApplicationInfo.isSystemApp() && !this.mApplicationInfo.isInstantApp() && !this.mPackageName.equals("android")) {
      String procName = Application.getProcessName();
      if (procName.equals(this.mPackageName) && appContext != null) {
          ResOptExtFactory.getInstance().makeResOptExt().preloadRes(appContext, getResources());
      }
  }
  ```

  `Application.getProcessName()` is null in Shizuku's user-service process (it is started through
  `app_process`, so no normal `Application` attach has happened), so `procName.equals(...)` throws
  NullPointerException, `UserService.create()` catches it, logs
  "unable to start service ..." and returns null → the user service never registers → bind timeout.
- Current upstream code path (master of RikkaApps/Shizuku-API, `server-shared/src/main/java/rikka/shizuku/server/UserService.java`):
  it calls `LoadedApk.makeApplication(boolean, Instrumentation)` reflectively and does **not** set a
  process name before that call. Rikka's analysis in #1198 suggests calling
  `ActivityThread#bindApplication()` first (so `ActivityThread#mBoundApplication` is non-null) or
  setting the field manually on the current thread's `Context#mCurrentThread`.

## What to produce

1. **The exact upstream patch**, as a unified diff against
   `RikkaApps/Shizuku-API` → `server-shared/src/main/java/rikka/shizuku/server/UserService.java`
   (and any other file you conclude must change), that prevents the NPE.
   - Decide and justify the mechanism: set a non-null process name before `makeApplication`
     (e.g. via `Application.setProcessName` if reachable, or the private static field), and/or
     pre-populate `ActivityThread#mBoundApplication`, and/or catch the NPE narrowly and continue.
   - Verify against real AOSP sources what `Application.getProcessName()` reads and how it is set
     (quote the AOSP source and the field/method used, with the URL). Do not guess.
   - Prefer a fix that does NOT depend on the app's package name matching the process name
     (the user-service process is `<pkg>:user_service`), so MTK's resource-preload branch stays skipped.
2. **Why it is safe**: behaviour on stock Android and on the patched MTK ROM, plus what happens if
   `makeApplication` still fails afterwards (the existing `catch (Throwable)` path).
3. **A reproduction/verification recipe** the user can run on the device to prove the patch works:
   which Shizuku build to produce, how to build it (gradle tasks, signing), how to start the patched
   server, and which logcat line proves the user service now starts (and the exact NPE fingerprint
   to look for before the patch).
4. **Upstream submission notes**: which repo/branch the patch belongs in (Shizuku vs Shizuku-API),
   how the two are versioned together, and a ready-to-post issue/PR body for
   https://github.com/RikkaApps/Shizuku/issues/1198.
5. **A fallback recommendation**: if the Shizuku-side fix is not viable (e.g. it needs OEM-specific
   code), state that plainly and say the rikkahub-agent-side direct-transport fallback is the
   primary remedy.

## Constraints

- Verify claims against primary sources (AOSP/android.googlesource, the GitHub repos, the issue
  threads). Quote URLs for every factual claim about framework internals.
- No fabricated framework symbols: if you cannot confirm that `Application.setProcessName` exists or
  which field backs `getProcessName()`, say so and propose a reflection-based variant instead.
- Keep the proposal minimal — a few lines in one file — and explain why a bigger change is not needed.
- Do not clone or modify any repository; deliver the diff as text in your Result.

## Result format

End with `## Result` containing: the recommended fix in one paragraph, the unified diff, the
verification recipe, and a confidence statement with the evidence you used.
