# Step 6 — Audio Off mode + master-mute callback conflation

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md), [Step 2](step2_settings_global_persistence.md)

**Do this step last**, after Steps 1-5 are stable. This is the largest, most structurally entangled item in
the whole migration.

## What it is

"Audio Off" is implemented as a **MEDIA-volume-group-mute proxy** — there is no independent mode flag,
state machine, or dedicated Settings observer. Some other component (Nissan HMI app or a physical switch)
calls the **ordinary public `CarAudioManager.setVolumeGroupMute(zoneId, groupId, mute, flags)` API** against
the primary zone's MEDIA group, exactly like a user manually muting media volume would. `CarAudioService`'s
existing `handleMuteChanged()` special-cases that specific group to additionally block/unblock entertainment
focus, which is what turns a plain group mute into "Audio Off" behavior.

## Confirmed on-disk in `a14full` (`nissan_u_ccs2_release`, clean working tree, no local edits)

### `CarAudioService.handleMuteChanged()` — `CarAudioService.java:723-738`, read directly

```java
private void handleMuteChanged(int zoneId, int groupId, int flags) {
    if (!mUseCarVolumeGroupMuting) {
        return;
    }

    // Handle AudioOFF change state
    if(groupId == getVolumeGroupIdForUsage(PRIMARY_AUDIO_ZONE,USAGE_MEDIA))
    {
        CarVolumeGroup group_media = getCarVolumeGroupLocked(zoneId, groupId);
        mFocusHandler.applyUserMuteAudioFocusPolicy(zoneId, group_media.isMuted());
    }

    callbackGroupMuteChanged(zoneId, groupId, flags);
    callbackMasterMuteChange(zoneId, flags);
    mCarVolumeGroupMuting.carMuteChanged();
}
```

There is **no separate trigger file**. `handleMuteChanged` is called from the existing mute/volume-change
call sites already in `CarAudioService.java` (confirmed at lines 712, 2533, 3174, 3201 — the same places
A17 stock already calls it from). Whatever mutes the MEDIA group — an app calling
`CarAudioManager.setVolumeGroupMute()`, a hardware key, an HMI control — flows through the exact same path
as any other group mute; `handleMuteChanged`'s media-group branch is the entire "Audio Off" hook.

Confirmed current **A17-stock** form (`a17full`'s `CarAudioService.java:1284-1290`, for comparison):

```java
private void handleMuteChanged(int zoneId, int groupId, int flags) {
    if (!useCarVolumeGroupMuting()) {
        return;
    }
    callbackGroupMuteChanged(zoneId, groupId, flags);
    mCarVolumeGroupMuting.carMuteChanged();
}
```

`callbackGroupMuteChanged()` only calls `onGroupMuteChange(...)`; `callbackMasterMuteChange()` is called
**only** from `setMasterMute()` today. The Nissan delta is exactly the two things visible in the diff above:
the media-group `applyUserMuteAudioFocusPolicy` call, and the extra `callbackMasterMuteChange(zoneId,
flags)` call (which is what conflates group-mute with master-mute for HMI/callback purposes).

### Fan-out: `CarZonesAudioFocus.applyUserMuteAudioFocusPolicy()` — `CarZonesAudioFocus.java:389-394`, read directly

```java
void applyUserMuteAudioFocusPolicy(int zoneId, boolean muted) {
    CarAudioFocus focus = mFocusZones.get(zoneId);
    if (focus != null) {
        focus.applyUserMuteAudioFocusPolicy(muted);
    }
}
```

### Enforcement: `CarAudioFocus.java` — read directly, in full

- `private boolean mUserMuted = false;` (line 108) — a single per-zone boolean, **not** per-`FocusEntry`
  state.
- `applyUserMuteAudioFocusPolicy(boolean userMuted)` (lines 1147-1158):
  ```java
  void applyUserMuteAudioFocusPolicy(boolean userMuted) {
      Slogf.i(TAG, "applyUserMuteAudioFocusPolicy muted=" + userMuted);
      synchronized (mLock) {
          mUserMuted = userMuted;
          if (userMuted) {
              return;
          }
          Slogf.i(TAG, "applyUserMuteAudioFocusPolicy unmute & unblocking user mutable losers");
          removeBlockerAndRestoreUnblockedFocusLosersLocked(null);
      }
  }
  ```
  Turning **on** just sets the flag and returns (existing entertainment focus holders keep playing — this
  only affects *future* regain). Turning **off** re-runs the same blocker-restoration sweep used for normal
  focus-entry cleanup, passing `null` as the "dead entry" (no entry actually died; this call just wants to
  re-evaluate everyone).
- `isEntertainmentSource(FocusEntry entry)` (lines 790-807) — null-safe; `true` iff
  `AudioAttributes.getUsage()` is `USAGE_MEDIA` or `USAGE_GAME`.
- `removeBlockerAndRestoreUnblockedFocusLosersLocked(FocusEntry deadEntry)` (lines 812-844) — the
  restoration guard, with the `deadEntry != null` null-check the general cleanup path needs:
  ```java
  if (deadEntry != null) {
      entry.removeBlocker(deadEntry);
  }
  if (entry.isUnblocked() && !(mUserMuted && isEntertainmentSource(entry))) {
      // ... restore: it.remove(); entry.setDucked(false); mFocusHolders.put(...);
      // dispatchFocusGainedLocked(entry.getAudioFocusInfo());
  }
  ```

### `FocusInteraction.AUDIO_OFF_MODE_ENABLED` — confirmed dead code

`FocusInteraction.java:78` declares `private static final String AUDIO_OFF_MODE_ENABLED =
"CAR_AUDIO_OFF_MODE_ENABLED";` — confirmed on disk, and confirmed **never referenced anywhere else** in the
package (no observer, no read). Leftover from an earlier iteration. **Do not port.**

### Persistence (depends on Step 2)

Reboot persistence of the MEDIA-group mute (and thus of Audio Off) goes through
[Step 2](step2_settings_global_persistence.md)'s `ro.config.isPersistVolumeGroupMute` sysprop override on
`CarAudioSettings.isPersistVolumeGroupMuteEnabled()` — also confirmed present verbatim in the current
`a14full` `CarAudioSettings.java`.

If an external trigger component exists (Nissan HMI app or a physical switch), it lives outside
`car/audio/` and simply calls the standard `setVolumeGroupMute` API — there's nothing to port on the
`car/audio` side for the trigger itself. There is no per-`FocusEntry` mute tracking; the mechanism uses a
single `mUserMuted` boolean on `CarAudioFocus`, checked inline in the restoration guard — do not add mute
state to `FocusEntry.java`.

## New integration risk not present in A14

A17's `CarAudioFocus` already has Google's own fade-manager dispatch —
`sendFocusLossLocked(loser, lossType, winner, shouldFade, transientFadeManagerConfig)`, real AOSP A15+ code,
currently **inert** since `audioUseFadeManagerConfiguration=false` in `a17full`'s config. The Audio-Off
restoration guard above must compose with this dispatch path, not fight it, if fade is ever turned on later.
Since the verification pass recommends keeping fade off, this is a **documented risk, not a blocker** for
this step — but flag it explicitly in code review so nobody enables fade later without re-testing Audio Off.

## Files to change in `a17full`

- **Modify** `CarAudioService.java`: `handleMuteChanged()` — add the media-group check
  (`applyUserMuteAudioFocusPolicy` call) and the extra `callbackMasterMuteChange(zoneId, flags)` call, per
  the confirmed A14 diff above. No new file needed.
- **Modify** `CarZonesAudioFocus.java`: add `applyUserMuteAudioFocusPolicy(zoneId, muted)` — verbatim, as
  shown above.
- **Modify** `CarAudioFocus.java`: add `mUserMuted` field, `isEntertainmentSource()`,
  `applyUserMuteAudioFocusPolicy()` — verbatim, as shown above; modify the restoration guard in
  `removeBlockerAndRestoreUnblockedFocusLosersLocked()` to add the `!(mUserMuted &&
  isEntertainmentSource(entry))` clause.
- **No change** to `FocusEntry.java` or `FocusInteraction.java` needed for this mechanism.

(Method/field names to double check against current A17 — `getVolumeGroupIdForUsage`,
`getCarVolumeGroupLocked` — confirm exact signatures in `a17full`'s `CarAudioService.java` rather than
assuming the A14 names carried over unchanged, since some helper names may differ after the A17 refactor.)

## Verify

- Have some caller (a test app, or `adb shell` via `ICarAudio`/`car_service` shell command if one exists —
  confirm the real trigger mechanism via [Step 0](step0_resync_source.md)) call
  `CarAudioManager.setVolumeGroupMute()` on the primary zone's MEDIA group; confirm it mutes.
- With that mute active, start a new entertainment source that then loses focus and would normally regain
  it once unblocked; confirm it does **not** auto-resume while still muted.
- Unmute the MEDIA group; confirm previously-blocked entertainment focus is restored
  (`removeBlockerAndRestoreUnblockedFocusLosersLocked(null)` fires correctly).
- Confirm `ICarVolumeCallback.onMasterMuteChanged` fires on MEDIA-group mute specifically, not just on a
  true master mute via `setMasterMute()`.
- Reboot with the MEDIA group muted; confirm it persists per Step 2's sysprop/secure-setting logic.
- Confirm non-entertainment sources (navigation, phone, notifications) are **unaffected** — the guard is
  scoped to `USAGE_MEDIA`/`USAGE_GAME` only.
