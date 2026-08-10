# Step 4 — `ZoneAudioPlaybackCallback` hardening (verify-then-port)

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

## What it is

The two-tier `CarAudioPlaybackCallback` / `ZoneAudioPlaybackCallback` split itself is **stock upstream
AOSP** (introduced by commit `cde9fe27c87`, "Refactored CarAudioPlaybackCallback to
ZoneAudioPlaybackCallback" — not Nissan-authored), already present unmodified in `a17full`.

Only the **accumulated bugfixes on top of this stock class** are Nissan/Alliance-authored:

1. Null-`AudioDeviceInfo` guard (multiple fix commits: `6fa17fc0019`, `eacce20d75b`, `42aa5741ffc`,
   `78ce77e3776`, `3d765842ca6`, `d67091df5e0`).
2. Shutdown-state guard on `onPlaybackConfigChanged` (commits `f93f0146771`/`7e3324d5771`, "on
   playbackconfigchanged triggered when power state is on shutdown").
3. **ANR/main-thread fix** — see below, real commit content confirmed directly.

## The ANR fix — confirmed real, and precisely sourced

`git show 429afe45ba0` (`a14full`'s local Car repo), read in full — this is a genuinely important fix:

```
commit 429afe45ba06f8be2dea33d3a1cbc74de74a8cf9
Author: Bassem Boubaker <bassem.boubaker-extern@renault.com>
Date:   Wed Jul 8 12:03:13 2026 +0200

    [AUDIO][AOSP_IMPR] fix: main thread ANR/reboot on native maps guidance after sleep-wake

    [ISSUE Tracker] CCSEXT-241726, CCSEXT-242114
    [DESC] Watchdog killed com.android.car (triggering a system reboot and loss of
    active Google Maps guidance) because CarAudioPlaybackCallback was registered
    with a null Handler, so AudioManager delivered onPlaybackConfigChanged() on
    the main thread. That callback path can call into AudioManager
    (getDeviceForPortId -> listAudioDevicePorts -> updateAudioPortCache), making a
    synchronous native IPC to audioserver that can take several seconds under
    load (e.g. stopping projection while starting native maps), stalling the
    main thread long enough for the car watchdog daemon to kill the process.

    Thread 1 (main thread)
            at android.media.AudioManager.updateAudioPortCache(AudioManager.java:7445)
            - locked <0x0c8e6714> (a java.lang.Object)
            at android.media.AudioManager.listAudioDevicePorts(AudioManager.java:7255)
            at com.android.car.audio.CarAudioZone.findActiveAudioAttributesFromPlaybackConfigurations
            at com.android.car.audio.ZoneAudioPlaybackCallback.startTimersForContextThatBecameInactiveLocked
            at com.android.car.audio.ZoneAudioPlaybackCallback.onPlaybackConfigChanged
            - locked <0x09822bbd> (a java.lang.Object)
            at com.android.car.audio.CarAudioPlaybackCallback.onPlaybackConfigChanged

    fix: register CarAudioPlaybackCallback on CarAudioService's dedicated worker
    Handler instead of the main Looper, and narrow ZoneAudioPlaybackCallback's
    lock scope so the AudioManager lookup is never made while mLock is held.
```

**This is a real bug with a real symptom you'd recognize in the field: a full system reboot, triggered by
the watchdog killing `com.android.car` while Google Maps native guidance is active, typically right after a
sleep-wake cycle or while switching from projection to native navigation.** Worth flagging loudly to
whoever triages field reboots — this exact signature (watchdog kill, main-thread stack trace through
`AudioManager.listAudioDevicePorts`) is diagnostic.

The actual patch (both files, complete):

```diff
--- a/service/src/com/android/car/audio/CarAudioService.java
+++ b/service/src/com/android/car/audio/CarAudioService.java
@@ -1715,7 +1715,7 @@ private void setUserMuteOnAudioOffMode(boolean mute, int flags) {
     private void setupAudioConfigurationCallbackLocked() {
         mCarAudioPlaybackCallback =
                 new CarAudioPlaybackCallback(mCarAudioZones, mClock, mKeyEventTimeoutMs);
-        mAudioManager.registerAudioPlaybackCallback(mCarAudioPlaybackCallback, null);
+        mAudioManager.registerAudioPlaybackCallback(mCarAudioPlaybackCallback, mHandler);
     }

--- a/service/src/com/android/car/audio/ZoneAudioPlaybackCallback.java
+++ b/service/src/com/android/car/audio/ZoneAudioPlaybackCallback.java
@@ -60,15 +60,15 @@ final class ZoneAudioPlaybackCallback {
         ArrayMap<String, AudioPlaybackConfiguration> newActiveConfigs =
                 filterNewActiveConfiguration(configurations);

+        List<AudioPlaybackConfiguration> newlyInactiveConfigurations;
         synchronized (mLock) {
-            List<AudioPlaybackConfiguration> newlyInactiveConfigurations =
-                    getNewlyInactiveConfigurationsLocked(newActiveConfigs);
+            newlyInactiveConfigurations = getNewlyInactiveConfigurationsLocked(newActiveConfigs);

             mLastActiveConfigs.clear();
             mLastActiveConfigs.putAll(newActiveConfigs);
-
-            startTimersForContextThatBecameInactiveLocked(newlyInactiveConfigurations);
         }
+
+        startTimersForContextThatBecameInactive(newlyInactiveConfigurations);
     }

-    @GuardedBy("mLock")
-    private void startTimersForContextThatBecameInactiveLocked(
+    private void startTimersForContextThatBecameInactive(
             List<AudioPlaybackConfiguration> inactiveConfigs) {
         List<AudioAttributes> activeAttributes = mCarAudioZone
                 .findActiveAudioAttributesFromPlaybackConfigurations(inactiveConfigs);

-        for (int index = 0; index < activeAttributes.size(); index++) {
-            mAudioAttributesStartTime.put(activeAttributes.get(index), mClock.uptimeMillis());
+        synchronized (mLock) {
+            for (int index = 0; index < activeAttributes.size(); index++) {
+                mAudioAttributesStartTime.put(activeAttributes.get(index), mClock.uptimeMillis());
+            }
         }
     }
```

The shape is exactly what you'd want: `getNewlyInactiveConfigurationsLocked` and the
`mLastActiveConfigs` mutation stay under `mLock` (cheap, in-memory); the result is captured into a local
variable *before* releasing the lock; the `AudioManager`-touching call
(`findActiveAudioAttributesFromPlaybackConfigurations`) then runs **fully unlocked**; only the final
`mAudioAttributesStartTime.put()` loop re-acquires `mLock` briefly to record the results.

## Important: this fix is not on `nissan_u_ccs2_release` — it's a genuine cross-branch gap

`git merge-base --is-ancestor 429afe45ba0 HEAD` on the local `a14full` checkout of
`nissan_u_ccs2_release` returns **false**. `git branch -a --contains 429afe45ba0` shows it only on a
**sibling branch family**: `alliance_u_26w30_260720`/`260722`/`260724` and `alliance_u_release`. This is
not a reconstruction artifact or a future/unmerged iteration of Nissan's own line — it's a real fix that
landed on the shared Alliance branch and **never got merged into the Nissan branch at all**, in A14 or A17.
See [Step 0](step0_resync_source.md) for the full finding and the follow-up question (whether
`nissan_u_ccs2_release` should be picking up `alliance_u_release` fixes automatically).

## Confirmed state of `a17full` — has the same bug, unfixed

- `CarAudioService.java:2479` — `setupAudioConfigurationCallbackLocked()` still calls
  `mAudioManagerWrapper.registerAudioPlaybackCallback(mCarAudioPlaybackCallback, null)` — the exact
  pre-fix `null` Handler.
- `ZoneAudioPlaybackCallback.java:68-93` — `onPlaybackConfigChanged()` still calls the lock-suffixed
  `startTimersForContextThatBecameInactiveLocked(...)` (line 86) **inside** `synchronized (mLock)`
  (lines 73-87), and that method (`:135-144`) calls
  `mCarAudioZone.findActiveAudioAttributesFromPlaybackConfigurations(inactiveConfigs)` — the synchronous
  native-IPC call — while still holding `mLock`. Same exposure as A14.

**Fix 1 (null-`AudioDeviceInfo` guard) — likely not reproducible as-is, confirm during testing:**
`getAudioDeviceAddressFromConfig()` (`ZoneAudioPlaybackCallback.java:203-211`) already re-reads
`devices.get(c)` once and derives both the validity check and `.getAddress()` from the same loop variable,
rather than caching two separate `AudioDeviceInfo` references. The original NPE scenario (mismatched
instances between a check and a put) doesn't obviously reproduce against this structure. Don't blindly port
the old guard — re-test the original NPE repro steps against current code first.

**Fix 2 (shutdown-state guard) — not yet checked.** Needs a look at `onPlaybackConfigChanged` call sites
during device shutdown/power-state transition once [Step 0](step0_resync_source.md) locates the exact
original repro/ticket to test against.

## Files to change in `a17full`

- `CarAudioService.java` — `setupAudioConfigurationCallbackLocked()`: change
  `registerAudioPlaybackCallback(mCarAudioPlaybackCallback, null)` to
  `registerAudioPlaybackCallback(mCarAudioPlaybackCallback, mHandler)` — `a17full` already has an
  equivalent `mHandler` bound to its `mHandlerThread` (confirmed present, used elsewhere in
  `CarAudioService.java` for mirroring/zone-config-switch work), so this is a one-line change, not a new
  thread.
- `ZoneAudioPlaybackCallback.java` — apply the exact restructuring above: capture
  `newlyInactiveConfigurations` into a local variable inside the lock, release the lock, call the renamed
  `startTimersForContextThatBecameInactive` (no `Locked` suffix, no `@GuardedBy` annotation) unlocked, and
  re-acquire `mLock` only around the final `mAudioAttributesStartTime.put()` loop.
- Fixes 1 and 2: port only if testing in this step confirms they're still reproducible against current code
  — do not port speculatively.

## Risk

Low mechanically (the diff is small and precise), but this is a real concurrency correctness fix — test
carefully. The specific risk the original fix already accounted for: don't let
`mLastActiveConfigs`/`mZoneConfigurations` (still locked) get out of sync with the now-unlocked
`newlyInactiveConfigurations` snapshot; the upstream patch handles this by snapshotting before unlocking,
preserve that ordering exactly.

## Verify

- Reproduce the original trigger as closely as possible: active native Maps guidance, sleep-wake cycle or
  a projection→native-navigation switch, under load. Confirm no watchdog kill / reboot.
- Trigger a playback-config-changed burst around device add/remove and around shutdown generally; confirm
  no ANR under load, and confirm `onPlaybackConfigChanged`/`startTimersForContextThatBecameInactive` are no
  longer observed running with `mLock` held during the `AudioManager` call (e.g. via a targeted logcat/trace
  check during a manual test).
- Attempt to reproduce the original null-`AudioDeviceInfo` NPE (rapid device add/remove during active
  playback) against the current code; only port the old guard if it still reproduces.
- Attempt to reproduce the shutdown-state issue (trigger playback-config-changed during a simulated
  power-state-on-shutdown transition); only port the guard if it still reproduces.
- Confirm no regression to volume-key routing / "still active" timeout behavior after the lock-scope change.
