# Step 9 — Suspend-exit volume-limit scoping + power-state limit reset

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

> **Provenance note:** same as [Step 8](step8_gain_context_enrichment.md) — found via the authoritative
> `packages_services_Car.diff` (Nissan `nissan_u_ccs2_release` vs Google `android14-release`), not the
> earlier A14-fork research passes. Confirmed missing from `a17full`: zero hits for
> `CarPowerManagementService`, `subscribePowerManagementService`, `isSuspendReason` anywhere in
> `car/audio/`. `a17full` does have a `CarAudioPowerListener.java`, but it's an unrelated, pre-existing
> stock AOSP mechanism — it listens to `ICarPowerPolicyListener`/`CarPowerPolicy` (whether the AUDIO power
> *component* is enabled), not `ICarPowerStateListener`/power *state* transitions
> (`STATE_SUSPEND_EXIT`/`STATE_HIBERNATION_EXIT`). No collision; this is a genuinely separate, additional
> mechanism to add.

## What it is

Two related pieces addressing the same underlying scenario — media volume potentially being too loud
immediately after the car wakes from suspend/hibernation:

### 9a. `SUSPEND_EXIT_VOL_LIMITATION` scoped to media groups only, in `CarVolumeGroup.onAudioGainChanged()`

The AudioControl HAL reports `Reasons.SUSPEND_EXIT_VOL_LIMITATION` on wake, targeting a specific bus
(confirmed comment: "HAL sets BUS00_MEDIA only"). Nissan's fix prevents that HAL-reported limitation from
being wrongly applied to **non-media** volume groups that happen to share the same bus address:

```java
boolean hasMediaUsage = getAllSupportedUsagesForAddress(gain.getDeviceAddress())
        .contains(AudioAttributes.USAGE_MEDIA);
...
boolean shouldLimit = CarAudioGainMonitor.shouldLimitVolume(halReasons);
// SUSPEND_EXIT_VOL_LIMITATION targets only media groups (HAL sets BUS00_MEDIA only).
// Non-media groups sharing the same bus address must not inherit the media limit.
if (shouldLimit && CarAudioGainMonitor.isSuspendReason(halReasons) && !hasMediaUsage) {
    shouldLimit = false;
}
```

And a companion explicit clamp — if the suspend-exit HAL index is lower than the group's current cached
index, snap down to it (don't let stale pre-suspend volume linger above the HAL-reported safe ceiling):

```java
if (CarAudioGainMonitor.isSuspendReason(halReasons)
        && hasMediaUsage
        && (mCurrentGainIndex > halIndex)) {
    mCurrentGainIndex = halIndex;
}
```

`CarAudioGainMonitor.isSuspendReason(List<Integer> reasons)` (a small Nissan-only helper,
`reasons.contains(Reasons.SUSPEND_EXIT_VOL_LIMITATION)`) is a prerequisite for this — it exists in `a14full`
but was already independently removed/folded into a lookup table in AOSP's own `CarAudioGainMonitor.java`
evolution by A17 (confirmed during the earlier verification pass — this is not itself a functional gap,
just a helper Nissan's code happens to call by name; re-add it as a small private/package-visible static
method if it doesn't already exist under a different name in `a17full`'s current `CarAudioGainMonitor.java`).

### 9b. `CarAudioService` power-state listener — reset all volume-group limits on suspend/hibernation exit

A broader safety net, independent of any single HAL gain callback — registers directly with
`CarPowerManagementService` for raw power-state transitions and resets **every** primary-zone volume
group's limit state on exit from suspend or hibernation:

```java
private final ICarPowerStateListener mCarPowerStateListener =
    new ICarPowerStateListener.Stub() {
        @Override
        public void onStateChanged(int state, long expirationTimeMs) {
            CarPowerManagementService powerService =
                    CarLocalServices.getService(CarPowerManagementService.class);
            if (powerService == null) {
                Slogf.w(TAG, "Cannot get CarPowerManagementService");
                return;
            }
            Slogf.d(TAG, "onStateChanged " + powerService.getPowerState());
            handlePowerState(powerService.getPowerState());
        }
    };

private void subscribePowerManagementService() {
    Slogf.i(TAG, "subscribePowerManagementService");
    CarPowerManagementService powerService =
            CarLocalServices.getService(CarPowerManagementService.class);
    if (powerService == null) {
        Slogf.w(TAG, "Cannot get CarPowerManagementService");
        return;
    }
    powerService.registerListener(mCarPowerStateListener);
}

private void handlePowerState(int powerState) {
    CarVolumeGroup[] groups = getCarAudioZoneLocked(PRIMARY_AUDIO_ZONE).getCurrentVolumeGroups();
    switch (powerState) {
        case CarPowerManager.STATE_SUSPEND_EXIT:
        case CarPowerManager.STATE_HIBERNATION_EXIT:
            Slogf.i(TAG, "Exit suspend state, reset limitLock by suspend state");
            for (int i = 0; i < groups.length; i++) {
                groups[i].resetLimitLocked();
            }
            break;
        default:
            break;
    }
}
```

`subscribePowerManagementService()` is called once, alongside `setupOemAudioGainListener()`
([Step 3](step3_oem_gain_callback_wiring.md)) in `init()`. `CarVolumeGroup.resetLimitLocked()` itself
**already exists in `a17full`** as a general-purpose stock primitive (confirmed by grep) — only the
suspend/hibernation-exit trigger wiring above is missing, not the underlying reset mechanism.

## Files to change in `a17full`

- `CarAudioGainMonitor.java`: confirm whether an `isSuspendReason`-equivalent check exists (it may be
  folded into the current reason-lookup table under a different shape); add it back as a small helper if not,
  matching the exact semantics (`reasons.contains(Reasons.SUSPEND_EXIT_VOL_LIMITATION)`).
- `CarVolumeGroup.java`: in `onAudioGainChanged()`, add the `hasMediaUsage` check and both the `shouldLimit`
  scoping guard and the explicit `mCurrentGainIndex` clamp, exactly as shown above. This sits right next to
  [Step 8](step8_gain_context_enrichment.md)'s `isGroupActive` change in the same method — implement both
  together to avoid re-touching this method twice.
- `CarAudioService.java`: add `mCarPowerStateListener`, `subscribePowerManagementService()`,
  `handlePowerState()`; call `subscribePowerManagementService()` from `init()` alongside
  `setupOemAudioGainListener()` ([Step 3](step3_oem_gain_callback_wiring.md)).

## Risk

Low-medium. 9a touches the same `CarVolumeGroup.onAudioGainChanged()` method as
[Step 8](step8_gain_context_enrichment.md) — implement both in the same pass to avoid merge friction. 9b is
additive (a new listener registration + a reset call) with a well-defined, narrow trigger (only fires on
the two named power states).

## Verify

- Put the device into suspend, wake it, and confirm: media volume on the shared bus doesn't get stuck at an
  overly-limited or overly-loud level; non-media groups sharing the same bus address are unaffected by the
  media-only limitation.
- Confirm `handlePowerState` fires and `resetLimitLocked()` runs on all primary-zone groups specifically on
  `STATE_SUSPEND_EXIT`/`STATE_HIBERNATION_EXIT` — not on other power-state transitions (e.g. shutdown-prepare,
  on).
- Confirm no crash/NPE if `CarPowerManagementService` isn't available yet at the time
  `subscribePowerManagementService()` runs during `init()`.
