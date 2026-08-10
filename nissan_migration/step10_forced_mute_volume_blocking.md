# Step 10 — Block volume-set requests during forced-mute/cyber conditions

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

## What it is

Source: [`packages_services_Car.diff`](../diffs_nissan_vs_google/packages_services_Car.diff). The
HAL-reason-handling primitive (`Reasons.FORCED_MASTER_MUTE`/`TCU_MUTE`/`REMOTE_MUTE` via
`CarAudioGainMonitor.shouldBlockVolumeRequest()`, `CarVolumeGroup.onAudioGainChanged()` →
`setBlockedLocked()`) is stock AOSP, already present in `a17full` — no work needed there. But Nissan also
wires that same `shouldBlockVolumeRequest()` check into `CarAudioService.setGroupVolume()`, so an incoming
**explicit volume-set request** (a user turning a physical knob, an app calling
`CarAudioManager.setGroupVolume()`) is rejected outright while a blocking HAL reason (forced master mute /
TCU mute / remote mute — "cyber attack" scenarios per the AIDL doc) is active, not just gain values reported
*by* the HAL. This wiring is confirmed missing from `a17full`: zero hits for
`mReasons`/`shouldBlockVolumeRequest` in `CarAudioService.java` today.

## Confirmed via diff

`CarAudioService.java` gains a `private List<Integer> mReasons = new ArrayList<>();` field, kept in sync
with the **latest HAL gain-change reasons** from both existing entry points:

```java
private final HalAudioGainCallback mHalAudioGainCallback =
        new HalAudioGainCallback() {
            @Override
            public void onAudioDeviceGainsChanged(
                    List<Integer> halReasons, List<CarAudioGainConfigInfo> gains) {
                Slogf.i(TAG, "onAudioDeviceGainsChanged halReasons: %s, gains: %s", halReasons, gains);
                synchronized (mImplLock) {
                    mReasons = new ArrayList<>(halReasons);
                    handleAudioDeviceGainsChangedLocked(halReasons, gains);
                }
            }
        };

/**
 * Audio Gain Changes Reported by OEM Callback
 */
void onAudioGainChanged(List<Integer> halReasons, List<CarAudioGainConfigInfo> gains) {
    Slogf.i(TAG, "onAudioGainChanged halReasons: %s, gains: %s", halReasons, gains);
    synchronized (mImplLock) {
        mReasons = new ArrayList<>(halReasons);
        handleonAudioGainChangedLocked(halReasons, gains);
    }
}
```

(The `onAudioGainChanged` entry point above is the OEM-path callback that
[Step 3](step3_oem_gain_callback_wiring.md)'s `CarAudioGainCallbackHandler` calls into — both the
HAL-direct and OEM gain paths update the same `mReasons` field.)

Then `setGroupVolume()` checks it before applying any explicit volume-set request:

```java
public void setGroupVolume(int zoneId, int groupId, int index, int flags) {
    enforcePermission(Car.PERMISSION_CAR_CONTROL_AUDIO_VOLUME);
    callbackGroupVolumeChange(zoneId, groupId, flags);
    ...
    synchronized (mImplLock) {
        boolean shouldBlock = CarAudioGainMonitor.shouldBlockVolumeRequest(mReasons);
        if (shouldBlock) {
            Slogf.d(TAG, "setGroupVolume, volume change is blocked for groupId: %d,", groupId);
            return;
        }
        CarVolumeGroup group = getCarVolumeGroupLocked(zoneId, groupId);
        wasMute = group.isMuted();
        group.setCurrentGainIndex(index);
        ...
    }
}
```

`CarAudioGainMonitor.shouldBlockVolumeRequest(List<Integer> reasons)` already exists, unmodified, in
`a17full` today (confirmed in the earlier verification pass):
```java
static boolean shouldBlockVolumeRequest(List<Integer> reasons) {
    return reasons.contains(Reasons.FORCED_MASTER_MUTE) || reasons.contains(Reasons.TCU_MUTE)
            || reasons.contains(Reasons.REMOTE_MUTE);
}
```
This is the "cyber attack" scenario the `Reasons.aidl` doc comment references — so the primitive was always
there; only the wiring from `setGroupVolume()` into it, and the `mReasons` bookkeeping that feeds it, were
missing.

## Files to change in `a17full`

- `CarAudioService.java`:
  - Add `private List<Integer> mReasons = new ArrayList<>();`.
  - Update `mHalAudioGainCallback.onAudioDeviceGainsChanged()` to populate `mReasons` before dispatching,
    as shown above.
  - Add the `onAudioGainChanged(List<Integer> halReasons, List<CarAudioGainConfigInfo> gains)` entry point
    (if not already added as part of [Step 3](step3_oem_gain_callback_wiring.md)) and populate `mReasons`
    there too.
  - In `setGroupVolume()`, add the `shouldBlock` check and early return, exactly as shown above — insert it
    right before the existing `CarVolumeGroup group = getCarVolumeGroupLocked(...)` line.

## Risk

Low — small, additive, reuses an existing stock primitive (`shouldBlockVolumeRequest`). The only real risk
is scope: this blocks **all** explicit volume-set requests while any of the three listed HAL reasons is
active, including legitimate ones from the OEM plugin or system UI — confirm that's the intended product
behavior (it matches the "cyber attack" defensive intent documented in the AIDL `Reasons.aidl` comment: the
system should refuse to let anything un-mute/change volume while that condition is asserted by the HAL).

## Verify

- Simulate (or trigger via the Alliance vendor HAL, if there's a test hook) a `FORCED_MASTER_MUTE` /
  `TCU_MUTE` / `REMOTE_MUTE` HAL reason; confirm a subsequent `CarAudioManager.setGroupVolume()` call is
  silently rejected (no gain-index change, no crash) while the reason is active.
- Confirm volume-set requests succeed normally again once the HAL clears the reason.
- Confirm this doesn't interfere with [Step 8](step8_gain_context_enrichment.md)'s `isGroupActive`/
  NAV_DUCKING logic or [Step 9](step9_suspend_wake_volume_limit.md)'s suspend-exit limiting — all three
  touch gain-callback-adjacent code paths in the same file/class family, so test them together, not just
  individually.
