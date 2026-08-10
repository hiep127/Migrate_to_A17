# Step 8 — Gain-callback context enrichment: NAV_DUCKING exposure + shared-port active-group disambiguation

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

Source: [`packages_services_Car.diff`](../diffs_nissan_vs_google/packages_services_Car.diff) (Nissan
`nissan_u_ccs2_release` vs Google `android14-release`). Confirmed missing from `a17full` by direct grep —
zero hits for `isNavDuckingActive`, `isVolumeGroupActive`, `setNavDucking`, `isGroupActive`.

## What it is

Two related additions, both threaded through the same gain-callback dispatch chain
(`CarAudioGainMonitor` → `CarAudioZone`/`CarAudioZoneConfig` → `CarVolumeGroup`), both exposed outward via
`CarVolumeInfoWrapper`/`CarAudioService`.

### 8a. NAV_DUCKING raw-HAL-state exposure to the OEM plugin

A read-only chain exposing whether the AudioControl HAL currently reports an active `NAV_DUCKING` reason
for a zone — distinct from the *software* attenuation index (which gets reset by user volume key presses).
Confirmed real doc comment from `CarAudioZone.java`:

```java
/**
 * Returns true if the current zone configuration has an active
 * NAV_DUCKING reason from the Audio HAL. This reflects raw HAL gain reason state and
 * is not cleared by user volume key presses (unlike the software attenuation index).
 * Also handles the corner case where the HAL reports NAV_DUCKING with an empty gains list.
 * The zone config tracks halReasons unconditionally on every HAL callback.
 */
boolean isNavDuckingActive() {
    return getCurrentCarAudioZoneConfig().isNavDuckingActive();
}
```

Chain, confirmed via diff:

- `CarAudioZoneConfig.java`: new field
  `private List<Integer> mLastHalReasons = new ArrayList<>();`, updated unconditionally at the top of
  `onAudioGainChanged()` (`mLastHalReasons = new ArrayList<>(halReasons);`); new method
  `isNavDuckingActive()` returning `mLastHalReasons.contains(Reasons.NAV_DUCKING)`.
- `CarAudioZone.java`: `isNavDuckingActive()` delegates to the current zone config (shown above).
- `CarAudioService.java`: `isNavDuckingActiveForZone(int zoneId)` delegates to the zone, under
  `mImplLock`.
- `CarVolumeInfoWrapper.java`: `isNavDuckingActiveForZone(int zoneId)` delegates to `CarAudioService`.
- `CarAudioPolicyVolumeCallback.java`: reads it and threads it into the OEM volume-request builder:
  ```java
  boolean navDucking = mCarVolumeInfo.isNavDuckingActiveForZone(zoneId);
  return new OemCarAudioVolumeRequest.Builder(zoneId)
          .setCarVolumeGroupInfos(infos)
          .setActivePlaybackAttributes(activeAudioAttributes)
          .setCallState(mCarVolumeInfo.getCallStateForZone(zoneId))
          .setNavDucking(navDucking)
          .build();
  ```
- `car-lib/src/android/car/oem/OemCarAudioVolumeRequest.java`: full public API addition —
  `mNavDucking` field, `isNavDucking()` getter, `Builder.setNavDucking(boolean)`, plus the matching
  `equals()`/`hashCode()`/`toString()`/`Parcelable` plumbing. **This is a public car-lib API surface change**
  the Alliance OEM plugin can consume from every `OemCarAudioVolumeRequest` it receives — check whether the
  plugin (`vendor/alliance/services/car/plugins/audio/`) actually reads `isNavDucking()` today; if it does,
  this is a functional gap for the plugin on A17, not just an internal nicety.

### 8b. Shared-device-port gain-active disambiguation

Fixes a real correctness bug: several volume groups can share one physical device port (the example in the
Nissan comment: traffic announcement and media both routed to `BUS00_MEDIA`). A HAL gain-change report for
that port previously risked being adopted by whichever group's code ran last, even if that group isn't the
one actually rendering. Confirmed comment from `CarAudioZoneConfig.java`:

```java
/**
 * @param volumeInfo used to resolve whether a volume group sharing a device port with
 *                   another group is the one actually active on it (see the isGroupActive
 *                   guard inside {@link CarVolumeGroup#onAudioGainChanged}). {@code null} only
 *                   ever occurs in tests that call this method directly without going through
 *                   the real dispatcher -- production callers ({@link CarAudioGainMonitor},
 *                   {@link CarAudioGainCallbackHandler}) always pass a non-null instance,
 *                   enforced by {@code Objects.requireNonNull} in their constructors.
 */
```

- `CarAudioService.java`: `isVolumeGroupActive(int zoneId, int groupId)` — reads **cached playback
  activity** (not a live query), explicitly documented as safe to call from the audio gain callback path
  (avoiding re-entrant/blocking calls into `AudioManager` from inside a HAL callback).
- `CarVolumeInfoWrapper.java`: `isVolumeGroupActive(zoneId, groupId)` delegates to `CarAudioService`.
- `CarAudioZoneConfig.onAudioGainChanged()`: for each `CarVolumeGroup` whose addresses contain the reported
  device address, computes `isGroupActive = volumeInfo == null || volumeInfo.isVolumeGroupActive(getZoneId(), groupId)`
  and passes it into `CarVolumeGroup.onAudioGainChanged(halReasons, gainInfo, isGroupActive)`.
- `CarVolumeGroup.onAudioGainChanged()`: gains a 3rd `boolean isGroupActive` parameter; the index-adoption
  line becomes:
  ```java
  if (CarAudioGainMonitor.shouldUpdateVolumeIndex(halReasons)
          && isGroupActive
          && (halIndex != getRestrictedGainForIndexLocked(mCurrentGainIndex))) {
      mCurrentGainIndex = halIndex;
      eventType |= EVENT_TYPE_VOLUME_GAIN_INDEX_CHANGED;
  }
  ```
  i.e. a shared-port gain report only updates the group's stored index if that group is confirmed active —
  otherwise the inactive group's cached index is left untouched.
- `CarAudioGainMonitor.java`: the call into `CarAudioZone.onAudioGainChanged` gains the
  `mCarVolumeInfoWrapper` parameter so it can be threaded down: `carAudioZone.onAudioGainChanged(reasons,
  gainsByZones.valueAt(i), mCarVolumeInfoWrapper)`.

## Files to change in `a17full`

- `CarAudioZoneConfig.java`: add `mLastHalReasons` field + `isNavDuckingActive()`; change
  `onAudioGainChanged()` signature to accept `@Nullable CarVolumeInfoWrapper volumeInfo`, compute
  `isGroupActive` per group, pass it through.
- `CarAudioZone.java`: add `isNavDuckingActive()`; update `onAudioGainChanged()` call site to pass
  `volumeInfo` through to the zone config.
- `CarVolumeGroup.java`: `onAudioGainChanged()` gains the `isGroupActive` parameter; gate the index-adoption
  branch on it, matching the diff above exactly.
- `CarAudioGainMonitor.java`: thread `mCarVolumeInfoWrapper` through the `onAudioGainChanged` call.
- `CarAudioService.java`: add `isVolumeGroupActive(zoneId, groupId)` and `isNavDuckingActiveForZone(zoneId)`.
- `CarVolumeInfoWrapper.java`: add the matching pass-through methods.
- `CarAudioPolicyVolumeCallback.java`: read `isNavDuckingActiveForZone()` and call
  `OemCarAudioVolumeRequest.Builder.setNavDucking()`.
- `car-lib/src/android/car/oem/OemCarAudioVolumeRequest.java`: add the `mNavDucking` field, getter,
  builder method, and equality/parcel plumbing — this is a car-lib **API surface** change, so check whether
  it needs an API-lint/current.txt update in this codebase's build process.

Note: `a17full`'s current `CarAudioGainCallbackHandler.java` ([Step 3](step3_oem_gain_callback_wiring.md))
also calls into this same `onAudioGainChanged` chain via `CarAudioZone.onAudioGainChanged` — when
implementing Step 3, make sure the OEM gain path also threads `CarVolumeInfoWrapper` through consistently
with the HAL-direct path documented here, or 8b's disambiguation only works for one of the two entry points.

## Risk

Low-medium. 8a is purely additive (new getter chain + new builder field, default `false`/no-op if unused).
8b changes real gain-application logic in `CarVolumeGroup.onAudioGainChanged()` — a method already touched
by [Step 6](step6_audio_off_mode.md)'s indirect callers and general volume/mute flow; test carefully for any
zone/config that doesn't actually have shared device ports (should be a pure no-op there, since
`isGroupActive` is `true` whenever `volumeInfo` doesn't say otherwise).

## Verify

- Configure (or find) a device port shared by two volume groups (matching the fork's traffic-announcement +
  media example); trigger a gain change on that port; confirm only the actually-active group's stored index
  updates, not the inactive one's.
- Confirm `CarAudioPolicyVolumeCallback` correctly reports `navDucking=true` while NAV_DUCKING is HAL-active,
  and that it reflects raw HAL state (i.e. doesn't get cleared just because the user pressed a volume key).
- If the Alliance OEM plugin consumes `OemCarAudioVolumeRequest.isNavDucking()`, confirm its behavior with
  the flag now populated (previously it would have always seen the Java default, `false`, since A17 never
  set it).
