# Spec 01 — RikkaHub: direct server-process transport as a Shizuku fallback

Run id: 20261026-1905-shizuku-direct-fallback
Target repo (clone): /workspace/rikkahub-agent  (upstream: https://github.com/ExTV/rikkahub-agent, branch master @ a88f585)
Owner: builder sub-agent. Orchestrator commits only.

## Goal

`shizuku_exec` fails on Xiaomi/MediaTek ROMs with `shizuku_bind_failed` / `phase: bind_timeout`.
The Shizuku *server* is alive and accepts the bind request, but Shizuku's **user service** process
dies during start-up, so the binder is never handed back. Add a second transport that talks to the
Shizuku **server** directly (fork a shell process through the server binder), which never loads the
host app's `Application` and therefore never hits the ROM's patched `LoadedApk.makeApplicationInner`.

The upstream maintainer already promised this ("A fallback is planned", rikkahub-agent issue #45).

## Verified facts (do not re-derive; these were confirmed against the published artifact)

- Client artifact in use: `dev.rikka.shizuku:api` **13.1.5** (`gradle/libs.versions.toml:76`, `shizuku-api`).
- `rikka.shizuku.Shizuku` (api artifact) exposes:
  - `public static IBinder getBinder()`
  - `private static ShizukuRemoteProcess newProcess(String[] cmd, String[] env, String dir)`
    — descriptor confirmed from the class constant pool:
    `([Ljava/lang/String;[Ljava/lang/String;Ljava/lang/String;)Lrikka/shizuku/ShizukuRemoteProcess;`
  - The method is private on purpose: `moe.shizuku.server.IShizukuService` / `IRemoteProcess` are NOT
    shipped in the client artifact, so reflection is the only client-side route.
- `rikka.shizuku.ShizukuRemoteProcess` (public, in the artifact) extends `java.lang.Process` and adds:
  - `getInputStream()`, `getOutputStream()`, `getErrorStream()` — ParcelFileDescriptor-backed
  - `waitFor(): Int` — binder wait, returns exit code
  - `waitForTimeout(long timeout, TimeUnit unit): Boolean` — **use this one**
  - `exitValue(): Int`, `destroy()`, `alive(): Boolean`
- Do NOT use `Process.waitFor(long, TimeUnit)` on it: `ShizukuRemoteProcess` does not override that
  overload, so the default polling implementation runs instead of the binder wait and a finished
  command looks like a timeout.
- Existing package: `me.rerere.rikkahub.shizuku` —
  `ShizukuManager.kt`, `ShizukuCommandRunner.kt`, `ShizukuUserService.kt`, `ShizukuStatus.kt`,
  AIDL `app/src/main/aidl/me/rerere/rikkahub/shizuku/IShizukuUserService.aidl`.
- `ShizukuCommandRunner.run(command, timeoutMs, maxStdoutBytes, maxStderrBytes)` returns a
  `JsonObject` shaped `{success, exit_code, stdout, stderr}` or `{error: command_timeout, partial_stdout, partial_stderr}`.
  Reuse `me.rerere.rikkahub.data.ai.tools.local.BoundedOutputStream` for bounded capture.
- Tests are plain JUnit on the JVM (no Robolectric): `app/src/test/java/me/rerere/rikkahub/shizuku/`
  contains `ShizukuBindFailureTest.kt`, `ShizukuCommandRunnerTest.kt`, `ShizukuExecResponseParsingTest.kt`,
  `ShizukuStatusMapperTest.kt`.
- R8/minify: the release build shrinks. `rikka.shizuku.Shizuku.newProcess` is private and only used
  reflectively, so it MUST get a keep rule or it will be renamed/stripped in release builds.

## Required work

### 1. New file `app/src/main/java/me/rerere/rikkahub/shizuku/ShizukuServerProcess.kt`

An internal object that runs a command through the Shizuku server's own process API.

- Resolve the private method once and cache it:
  `Shizuku::class.java.getDeclaredMethod("newProcess", Array<String>::class.java, Array<String>::class.java, String::class.java)`
  + `isAccessible = true`. If the lookup fails, return a structured error (never throw out of the tool).
- Call it with `arrayOf("sh", "-c", command)`, `null` env, `null` dir.
- Close the child's stdin immediately (`outputStream.close()`) so a command that reads stdin gets EOF
  instead of blocking until the timeout.
- Drain stdout and stderr concurrently on daemon threads into `BoundedOutputStream`
  (a chatty command fills the 64KB pipe buffer and deadlocks if only one stream is read).
- Enforce the timeout with `waitForTimeout(timeoutMs, TimeUnit.MILLISECONDS)`; on timeout call
  `destroy()` and return the same `command_timeout` envelope shape as `ShizukuCommandRunner`
  (including partial output).
- On success return the same shape as `ShizukuCommandRunner`: `success`, `exit_code`, `stdout`, `stderr`.
- Do all blocking work on `Dispatchers.IO` inside `runInterruptible`.
- Keep the reflection lookup and the failure mapping in small, separately testable pure functions
  where possible.

### 2. Wire the fallback into `ShizukuManager.exec`

- Default path stays the user service (`bindUserService`); the direct transport is a FALLBACK.
- Fall back ONLY on bind-level failures: `BindResult.Failure.Timeout` and `BindResult.Failure.BindingDied`.
- NEVER fall back on a call-level failure (`shizuku_call_failed`): there the command may already have
  run, and re-running it could repeat side effects.
- When the fallback succeeds, return its JSON plus a `transport` field (`"server_process"`) so the
  agent and the user can see which path ran. The user-service path should report `"user_service"`.
- When the fallback also fails, return the bind-failure envelope, but make the `recovery` text say
  that the direct server-process fallback was attempted and failed, and name the next real step.
- Extract the fallback decision into a pure, unit-testable function.

### 3. R8 keep rules

Add rules (in the app's release/proguard config — find it, it was not under `app/build.gradle.kts`:
check `build-logic/` and the release build type) so the reflective call survives shrinking:

```
-keepclassmembers class rikka.shizuku.Shizuku {
    private static rikka.shizuku.ShizukuRemoteProcess newProcess(java.lang.String[], java.lang.String[], java.lang.String[]);
}
-keep class rikka.shizuku.ShizukuRemoteProcess { *; }
```

### 4. Tests (plain JUnit, `app/src/test/java/me/rerere/rikkahub/shizuku/`)

- `ShizukuFallbackPolicyTest.kt`: the fallback decision (bind timeout / binding died → fallback;
  bind threw → no fallback; call failure → no fallback) and the response-envelope shapes.
- Extend/add a test that the direct transport's success envelope has the same field names as the
  user-service one (`success`, `exit_code`, `stdout`, `stderr`) so tool callers cannot tell them apart
  except by the `transport` field.

## Constraints

- Kotlin style of the repo: KDoc on non-obvious public/internal API, `internal` visibility for
  app-internal helpers, 4-space indent, no wildcard imports.
- Quality gates: functions ≤ 50 lines, files ≤ 300 lines, ≤ 3 params per function, error handling at
  every I/O boundary, no hardcoded config (timeouts/limits as named constants), no duplicated logic
  ≥ 5 lines (reuse `BoundedOutputStream` and the existing envelope shapes).
- No blind `catch (e: Exception) {}` swallows: every caught failure must be logged (`android.util.Log`)
  and turned into a structured error.
- Do not change the existing user-service behaviour for devices where it works.
- Do not add new third-party dependencies. Do not touch the api version in `libs.versions.toml`.
- Do NOT run `git commit` / `git push` — the orchestrator commits.
- Do not modify files outside the paths listed above (file ownership: you own the shizuku package,
  the shizuku tests, and the proguard file).

## Verification (be honest about what you could run)

This sandbox has **no Android SDK** (`ANDROID_HOME` unset, no `sdkmanager`), so the Gradle build and
`app:testDebugUnitTest` cannot run here. Try in this order and record what actually happened:
1. If a Kotlin compiler + `android.jar` can be obtained cheaply, type-check the new file. Otherwise skip.
2. Run any pure-JVM test you can execute directly (the new policy test is pure Kotlin/JSON — if you
   can compile it standalone with kotlinc, do so).
3. Always leave the exact commands the user must run on a machine with the Android SDK in the Result:
   `./gradlew :app:testDebugUnitTest --tests 'me.rerere.rikkahub.shizuku.*'`
   `./gradlew :app:assembleRelease` (or debug) + install on the device.

## Done criteria

- [ ] `ShizukuServerProcess.kt` exists, implements the direct transport, and degrades to a structured
      error instead of throwing.
- [ ] `ShizukuManager.exec` falls back on bind timeout / binding died only, and reports `transport`.
- [ ] Recovery text for a failed bind mentions the attempted direct fallback.
- [ ] R8 keep rules added for `newProcess` + `ShizukuRemoteProcess`.
- [ ] New unit tests cover the fallback policy and envelope shape.
- [ ] Result states exactly what was verified and what still needs a device/SDK build.

## Result format

End your reply with a `## Result` section containing: files changed (paths), what each change does,
verification actually performed (commands + outcomes), what remains unverified, and any risk you saw.
