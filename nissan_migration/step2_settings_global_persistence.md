# Step 2 — `Settings.Global` persistence for MEDIA/RING/CALL

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

## What it is

One well-defined patch that fixes two separately-reported bugs at once:

- **CP-song max volume** (tickets `REAVN-156016`, commits `1b05ef870a1`/`788a11a61f2`/`5417577f32e`,
  "CP song is outputted in Max volume during profile switch"). "CP" = CarPlay per the commit description.
- **Guest-profile volume bar** (ticket `REAVN-158472`, commits `bfac1980504`/`69da031feb1`, "Phone volume
  bar appears when we switch to the guest profile").

Root cause both times: `getStoredVolumeGainIndexForUser`/`storeVolumeGainIndexForUser` persist per-user via
`Settings.System`, keyed by `userId`. During a user-switch transition, the framework briefly reads the new
(fresh/ephemeral) user's unset default (`-1`) before the real value is re-applied. For MEDIA this manifests
as a moment of max-volume playback across the switch; for RING/CALL on a guest profile it manifests as a
phantom phone/ring volume bar popping up (the framework treats the unset default as "louder than expected").

The CP-song fix (MEDIA only) landed first and is the foundational patch; the guest fix later extended the
exact same mechanism to RING and CALL.

## Source pattern (A14 fork, `CarAudioSettings.java`)

```java
GROUP_ID_MEDIA = CoreAudioHelper.getVolumeGroupIdForAudioAttributes(mediaAttributes) - 1;
GROUP_ID_RING  = CoreAudioHelper.getVolumeGroupIdForAudioAttributes(ringAttributes) - 1;
GROUP_ID_CALL  = CoreAudioHelper.getVolumeGroupIdForAudioAttributes(callAttributes) - 1;

// in getStoredVolumeGainIndexForUser / storeVolumeGainIndexForUser /
// storeVolumeGroupMuteForUser / getVolumeGroupMuteForUser:
if ((groupId == GROUP_ID_MEDIA) || (groupId == GROUP_ID_RING) || (groupId == GROUP_ID_CALL)) {
    // Settings.Global.getInt/putInt(...) instead of per-user Settings.System
}
```

No RRO booleans, no new Settings keys — reuses the existing `VOLUME_SETTINGS_KEY_FOR_GROUP_PREFIX` /
`VOLUME_SETTINGS_KEY_FOR_GROUP_MUTE_PREFIX` key scheme, just routes MEDIA/RING/CALL through
`Settings.Global` (device-wide) instead of per-user `Settings.System`.

## Files to change in `a17full`

`packages/services/Car/service/src/com/android/car/audio/CarAudioSettings.java` (current file is small —
read it in full first). The four methods to modify, confirmed present in A17 today, all currently routed
through the private `getIntForUser`/`putIntForUser` helpers (→ `Settings.System.getInt/putInt` via
`getContentResolverForUser(userId)`):

- `getStoredVolumeGainIndexForUser(userId, zoneId, configId, groupId)`
- `storeVolumeGainIndexForUser(userId, zoneId, configId, groupId, gainIndex)`
- `storeVolumeGroupMuteForUser(userId, zoneId, configId, groupId, isMuted)`
- `getVolumeGroupMuteForUser(userId, zoneId, configId, groupId)`

Changes:

1. Resolve `GROUP_ID_MEDIA`/`GROUP_ID_RING`/`GROUP_ID_CALL` once (constructor or static init) via
   `CoreAudioHelper.getVolumeGroupIdForAudioAttributes(...)`.
2. In each of the 4 methods, branch: if `groupId` matches one of the three, read/write through
   `Settings.Global.getInt(mContext.getContentResolver(), key, default)` /
   `Settings.Global.putInt(mContext.getContentResolver(), key, value)` instead of the per-user path.
   Otherwise, fall through to the existing stock behavior unchanged.
3. Add a sysprop override to `isPersistVolumeGroupMuteEnabled(userId)` — today it only reads
   `CarSettings.Secure.KEY_AUDIO_PERSIST_VOLUME_GROUP_MUTE_STATES` via `getSecureIntForUser`. Add:
   ```java
   int isPersistVolumeGroupMute = SystemProperties.getInt("ro.config.isPersistVolumeGroupMute", -1);
   if (isPersistVolumeGroupMute != -1) return isPersistVolumeGroupMute != 0;
   return getSecureIntForUser(CarSettings.Secure.KEY_AUDIO_PERSIST_VOLUME_GROUP_MUTE_STATES, 0, userId) == 1;
   ```
   This sysprop is what feeds [Step 6](step6_audio_off_mode.md)'s reboot-persistence behavior, since
   Audio Off is implemented purely as MEDIA-group mute.

## Risk

Low — additive branches in 4 already-small methods, no signature changes, no interaction with Google's new
fade/activation-volume code (that code touches `CarVolumeGroup`/`CoreAudioVolumeGroup`, not
`CarAudioSettings`).

## Verify

- Switch to a guest profile; confirm no phantom ring/phone volume bar appears.
- Switch users mid-playback; confirm MEDIA volume doesn't spike to max during the transition.
- Confirm groups **other than** MEDIA/RING/CALL still persist per-user as before — no regression to
  per-user volume for navigation, notification, etc.
- Confirm `ro.config.isPersistVolumeGroupMute` correctly overrides the stock Secure-setting path when set,
  and falls back to stock behavior when unset (`-1`).
