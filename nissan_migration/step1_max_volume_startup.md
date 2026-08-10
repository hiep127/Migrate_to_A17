# Step 1 — `CoreAudioVolumeGroupHelper.java` + MaxVolumeStartup

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

## What it is

A whole new Nissan-only file, absent from AOSP entirely — confirmed absent from the A17 baseline. Provides
`getDefaultVolumeForAttributes(attrs, maxGain, minGain, volumeGroup)`, a per-`AudioAttributes`-usage-category
default-volume lookup backed by `SystemProperties`, replacing AOSP's flat "1/3 of range" default for
core-audio volume groups.

`persist.vendor.max_vol_startup` is what actually implements "MaxVolumeStartup" — it's a **factory
first-boot ceiling** on `mDefaultGainIndex`, consulted only when there's no valid persisted user value
(`isValidGainIndexLocked(mStoredGainIndex)` is false). It is **not** a per-ignition re-clamp of the user's
saved volume — the persisted Media volume already lives in `Settings.Global` (see
[Step 2](step2_settings_global_persistence.md)) and survives reboots untouched.

## Source (A14 fork, `CoreAudioVolumeGroupHelper.java`)

```java
package com.android.car.audio;
final class CoreAudioVolumeGroupHelper {
    // system-property keys — mostly reused stock Android names + 3 new vendor ones
    DEFAULT_CALL                 = "ro.config.vc_call_vol_default"
    DEFAULT_MEDIA                = "ro.config.media_vol_default"
    DEFAULT_ALARM                = "ro.config.alarm_vol_default"
    DEFAULT_SYSTEM               = "ro.config.system_vol_default"
    DEFAULT_NOTIFICATION         = "ro.config.notification_vol_default"
    DEFAULT_TOUCHSCREEN_CLICK    = "persist.vendor.touchscreen_click_vol_default"   // new
    DEFAULT_ASSISTANT            = "persist.vendor.assistant_vol_default"          // new
    DEFAULT_MAX_VOLUME_START_UP  = "persist.vendor.max_vol_startup"                // new — MaxVolumeStartup

    static int getDefaultVolumeForAttributes(AudioAttributes attrs, int maxGain, int minGain,
            CoreAudioVolumeGroup volumeGroup)
}
```

`getDefaultVolumeForAttributes` is a `switch` on `attrs.getUsage()` mapping usage buckets to a
`SystemProperties` key, validated against `volumeGroup.isValidGainIndexLocked(...)` (a package-visible
method already exposed on `CoreAudioVolumeGroup` for exactly this purpose), falling through to the stock
formula `(maxGain - minGain) / 3 + minGain` if nothing valid is found:

- Media / Game / Unknown → `getMediaDefaultVolume()`:
  ```java
  int maxVolumeStartUp = SystemProperties.getInt("persist.vendor.max_vol_startup", -1);
  int defaultVolume    = SystemProperties.getInt("ro.config.media_vol_default", -1);
  return (defaultVolume != -1 && maxVolumeStartUp != -1)
          ? Math.min(defaultVolume, maxVolumeStartUp)
          : (maxVolumeStartUp != -1 ? maxVolumeStartUp : defaultVolume);
  ```
- Assistance sonification → touchscreen-click prop, falling back to system prop.
- Voice communication / signalling → call prop.
- Navigation guidance / Safety / Vehicle status / Assistant / Announcement → assistant prop.
- Alarm / Ringtone → alarm prop.
- Notification / Notification event → notification prop.

## Files to change in `a17full`

- **Add** `CoreAudioVolumeGroupHelper.java` to
  `packages/services/Car/service/src/com/android/car/audio/`, ported as-is from the Step-0-resynced source.
- **Modify** `CoreAudioVolumeGroup.java` constructor. A17's shape is different from A14's — it now takes
  `AudioManagerWrapper` / `SparseArray<CarAudioDeviceInfo>` / `CarActivationVolumeConfig`. Wire the helper
  call into the new constructor at the same spot AOSP's stock fallback formula
  (`(mMaxGainIndex - mMinGainIndex) / 3 + mMinGainIndex`) currently lives — keep that formula as the
  helper's own tail case, matching the A14 design (don't duplicate it in both places).

## Gate

Only active when `audioUseCoreVolume=true`. Confirm the `aivi2_n_full`/`aivi2_n_da2` device overlays set
this flag consistent with product intent — the `aasp`/emulator overlay already does, per the A14 fork
(`device/alliance/aasp/emulator/rro_overlay/CarServiceOverlay/res/values/config.xml`). The build-time
`persist.vendor.max_vol_startup=20` property itself comes from
`device/alliance/aasp/audio/aasp.mk` (`PRODUCT_PRODUCT_PROPERTIES`) in the A14 tree — confirm the
equivalent Nissan device `.mk` file still sets it for A17 targets.

## Open item, not blocking

AOSP's own `audioUseMinMaxActivationVolume` feature does a similar boot/source/playback-change volume
clamp (backed by the new `CarActivationVolumeConfig` class). `a17full`'s config already defaults it
`false`, so there's no active conflict today — just don't enable it later without re-checking against
MaxVolumeStartup, to avoid double-clamping boot volume.

## Verify

- Fresh data wipe; confirm first-boot MEDIA volume is clamped to `persist.vendor.max_vol_startup`'s
  configured value, not the stock 1/3-of-range default.
- Confirm devices/configs not using core audio volume (`audioUseCoreVolume=false`) are unaffected — the
  non-core `CarAudioVolumeGroup` class reads its default gain straight from XML `android:defaultGain` and
  is untouched by this change.
- Confirm a second boot after a user has already changed the volume does **not** re-clamp it (the ceiling
  only applies when there's no valid persisted value).
