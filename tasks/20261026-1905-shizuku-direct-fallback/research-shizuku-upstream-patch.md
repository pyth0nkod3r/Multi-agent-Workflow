# Research — Shizuku upstream patch for the MediaTek/HyperOS user-service NPE

Unit: spec-02 · Owner: researcher sub-agent (bg run da4c96da, SUCCEEDED) · Date: 2026-10-26

## Confirmed mechanism (primary sources)

- AOSP `android.app.Application.getProcessName()` →
  `ActivityThread.currentProcessName()` →
  `am.mBoundApplication != null ? am.mBoundApplication.processName : null`
  (frameworks/base, consistent Android 9 → 16).
- Shizuku's `UserService.create()` runs in a process started via `ActivityThread.systemMain()`,
  where `mBoundApplication` is intentionally null → `getProcessName()` returns null.
- MediaTek's patch inside `LoadedApk.makeApplicationInner` (quoted by Shizuku collaborator
  vvb2060, https://github.com/RikkaApps/Shizuku/issues/1171#issuecomment-2952634428):

  ```java
  if (!this.mApplicationInfo.isSystemApp() && !this.mApplicationInfo.isInstantApp() && !this.mPackageName.equals("android")) {
      String procName = Application.getProcessName();
      if (procName.equals(this.mPackageName) && appContext != null) {
          ResOptExtFactory.getInstance().makeResOptExt().preloadRes(appContext, getResources());
      }
  }
  ```

  → `procName` is null → NPE → `UserService.create()` catches Throwable → returns null →
  `ServiceStarter.main()` exits → the binder is never sent → client sees `phase: bind_timeout`.
- Same mechanism in Shizuku#1198 (open, unfixed), #1171, and independently reproduced in the
  community forks Nightzuku (PR #1) and shevery-fork — the `AppBindData` pre-population fix is
  already proven in the wild on MediaTek devices.

## The patch (target: RikkaApps/Shizuku-API → server-shared/src/main/java/rikka/shizuku/server/UserService.java)

```diff
@@ -72,16 +72,49 @@ public class UserService {
             Field mPackageInfo = context.getClass().getDeclaredField("mPackageInfo");
             mPackageInfo.setAccessible(true);
             Object loadedApk = mPackageInfo.get(context);
-            Method makeApplication = loadedApk.getClass().getDeclaredMethod("makeApplication", boolean.class, Instrumentation.class);
-            Application application = (Application) makeApplication.invoke(loadedApk, true, null);
-            Field mInitialApplication = activityThread.getClass().getDeclaredField("mInitialApplication");
-            mInitialApplication.setAccessible(true);
-            mInitialApplication.set(activityThread, application);
 
-            ClassLoader classLoader = application.getClassLoader();
+            // OEM frameworks (MediaTek/HyperOS, Transsion) call Application.getProcessName()
+            // inside LoadedApk#makeApplicationInner and expect a non-null String.
+            // Pre-populate ActivityThread#mBoundApplication so getProcessName() returns
+            // the actual process name rather than null (which triggers an NPE).
+            String processName = name != null ? name : pkg + ":user_service";
+            try {
+                Class<?> appBindDataClass = Class.forName("android.app.ActivityThread$AppBindData");
+                Constructor<?> ctor = appBindDataClass.getDeclaredConstructor();
+                ctor.setAccessible(true);
+                Object appBindData = ctor.newInstance();
+                setAppBindDataField(appBindDataClass, appBindData, "processName", processName);
+                setAppBindDataField(appBindDataClass, appBindData, "appInfo", context.getApplicationInfo());
+                Field mBoundApplication = activityThread.getClass().getDeclaredField("mBoundApplication");
+                mBoundApplication.setAccessible(true);
+                mBoundApplication.set(activityThread, appBindData);
+            } catch (Throwable ignored) {
+            }
+
+            Application application = null;
+            try {
+                Method makeApplication = loadedApk.getClass().getDeclaredMethod("makeApplication", boolean.class, Instrumentation.class);
+                application = (Application) makeApplication.invoke(loadedApk, true, null);
+                Field mInitialApplication = activityThread.getClass().getDeclaredField("mInitialApplication");
+                mInitialApplication.setAccessible(true);
+                mInitialApplication.set(activityThread, application);
+            } catch (Throwable tr) {
+                Log.w(TAG, String.format("makeApplication failed for %s/%s, falling back to Context-only user service", pkg, cls), tr);
+                application = null;
+            }
+
+            ClassLoader classLoader = application != null ? application.getClassLoader() : context.getClassLoader();
             Class<?> serviceClass = classLoader.loadClass(cls);
@@
             if (constructorWithContext != null) {
-                service = (IBinder) constructorWithContext.newInstance(application);
+                service = (IBinder) constructorWithContext.newInstance(application != null ? application : context);
             } else {
                 service = (IBinder) serviceClass.newInstance();
             }
@@
+    private static void setAppBindDataField(Class<?> clazz, Object target, String fieldName, Object value) {
+        if (value == null) {
+            return;
+        }
+        try {
+            Field field = clazz.getDeclaredField(fieldName);
+            field.setAccessible(true);
+            field.set(target, value);
+        } catch (Throwable ignored) {
+        }
+    }
 }
```

### Refinements before it is posted upstream

1. The two empty `catch (Throwable ignored) {}` blocks should log at warning level (upstream style
   accepts a narrow ignore, but a silent swallow hides a failed workaround). Log once, then continue.
2. `processName` should be the *user-service* process name (`<pkg>:user_service`), never the package
   name — MTK's branch compares the two, and matching would enter the resource-preload path.
3. `appInfo` must be non-null: AOSP `LoadedApk.createOrUpdateClassLoaderLocked` calls
   `ActivityThread.currentPackageName()`, which dereferences `mBoundApplication.appInfo.packageName`
   (secondary NPE documented in Nightzuku#12).
4. Upstream flow: land in `RikkaApps/Shizuku-API` (master), then bump the `api` submodule in
   `RikkaApps/Shizuku` (PR #80 introduced `makeApplication()`; PR #547 is the Context-only fallback).

## Verification recipe (Redmi 2409BRN2CG / mt6768 / HyperOS V816 / Android 16)

Pre-patch fingerprint:

```bash
adb logcat -d -s ShizukuServiceStarter:V
# W ShizukuServiceStarter: unable to start service <pkg>/<cls>...
# Caused by: java.lang.NullPointerException: ... on a null object reference
#   at android.app.LoadedApk.makeApplicationInner(LoadedApk.java:1526)
```

Build + deploy the patched manager:

```bash
git clone --recursive https://github.com/RikkaApps/Shizuku.git
cd Shizuku && git submodule update --init --recursive
# apply the patch to api/server-shared/src/main/java/rikka/shizuku/server/UserService.java
./gradlew :manager:assembleDebug
adb install -r manager/build/outputs/apk/debug/manager-debug.apk
adb shell sh /sdcard/Android/data/moe.shizuku.privileged.api/start.sh
```

Post-patch proof (user service now starts and hands the binder back):

```
I ShizukuServiceStarter: starting service <pkg>/<cls>...
I ShizukuServiceStarter: send binder to moe.shizuku.privileged.api in user 0
```

After that, the *unmodified* rikkahub-agent build works too — the standard `bindUserService`
transport starts working, so the client-side fallback is no longer required on that device.

## Verdict

- Upstream Shizuku fix: **viable and high confidence** — portable, no vendor code, already proven in
  independent forks. Needs a custom Shizuku build for the user today, or an upstream release later.
- Client-side fallback (spec-01, rikkahub-agent): still the **primary immediate remedy** — end users
  run stock Shizuku 13.6.0 and cannot be assumed to have a patched build.
