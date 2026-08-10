# Step 0 — Re-sync source of truth

[← back to index](00_README.md)

## Goal

Confirm the `a14full` checkout is a valid porting source, and pull in the one piece that isn't reachable
from it, before writing any A17 code.

## State of the `a14full` checkout

`a14full` (`nissan_u_ccs2_release`, HEAD `dbe12bde8d8`) is a valid, clean, up-to-date porting source. All of
the following are present, committed, and confirmed on disk (no local uncommitted edits):

- `CoreAudioVolumeGroupHelper.java` ([Step 1](step1_max_volume_startup.md))
- `CarAudioSettings.java`'s `GROUP_ID_MEDIA`/`RING`/`CALL` → `Settings.Global` logic and the
  `ro.config.isPersistVolumeGroupMute` sysprop override ([Step 2](step2_settings_global_persistence.md))
- `CarAudioGainCallbackHandler.java` ([Step 3](step3_oem_gain_callback_wiring.md))
- Audio Off's mechanism — `CarAudioService.handleMuteChanged()`'s media-group branch,
  `CarZonesAudioFocus.applyUserMuteAudioFocusPolicy()`, `CarAudioFocus`'s `mUserMuted`/
  `isEntertainmentSource()`/restoration-guard logic ([Step 6](step6_audio_off_mode.md))
- The BT voice-call focus block ([Step 5](step5_bt_voice_call_focus_block.md)), with full original comments

`git diff HEAD remotes/m/nissan_u_ccs2_release -- service/src/com/android/car/audio/` confirms local `HEAD`
(2026-08-03) is newer than `remotes/m/nissan_u_ccs2_release` (2026-04-29) — the remote ref is missing
content HEAD already has, not the other way around.

## The one real gap: Step 4's ANR fix is on a different branch entirely

The [Step 4](step4_zone_playback_callback_hardening.md) ANR fix, commit `429afe45ba0`, is **not** reachable
from `nissan_u_ccs2_release` — `git merge-base --is-ancestor 429afe45ba0 HEAD` returns false.
`git branch -a --contains 429afe45ba0` shows it only on a sibling branch family: `alliance_u_26w30_260720`/
`260722`/`260724` and `alliance_u_release`. Confirmed directly: `a14full`'s `CarAudioService.java:1711`
still has the pre-fix `registerAudioPlaybackCallback(mCarAudioPlaybackCallback, null)`, and
`ZoneAudioPlaybackCallback.java` still calls the lock-suffixed
`startTimersForContextThatBecameInactiveLocked` under `mLock`.

This is a real, unmerged, cross-branch fix — present on neither `nissan_u_ccs2_release` (A14) nor `a17full`
(A17). It needs to be pulled from `alliance_u_release` specifically, not reconstructed.

## Actions

1. Pull the ANR fix from `alliance_u_release` (or whichever `alliance_u_26w30_*` snapshot is authoritative):
   `git show 429afe45ba0 -- service/src/com/android/car/audio/ZoneAudioPlaybackCallback.java service/src/com/android/car/audio/CarAudioService.java`
   against that branch, then port forward to `a17full`'s current method names/structure (see
   [Step 4](step4_zone_playback_callback_hardening.md) for the full patch content).
2. Confirm with whoever owns Gerrit whether `nissan_u_ccs2_release` is meant to pick up `alliance_u_release`
   fixes automatically (branch merge policy), or whether this is a genuine gap Nissan needs to cherry-pick
   manually — this determines whether Step 4's finding is a one-off or a sign that other `alliance_u_*`-only
   fixes are also missing from the Nissan line and worth a broader audit later.

## Verify

- `git merge-base --is-ancestor 429afe45ba0 HEAD` on the authoritative `nissan_u_ccs2_release` ref returns
  true once the ANR fix is merged/cherry-picked, or the fix has been manually ported to `a17full` per
  [Step 4](step4_zone_playback_callback_hardening.md).
- Spot-check a few more `car/audio/` paths from the `alliance_u_*` branch family against
  `nissan_u_ccs2_release` to confirm no other fixes are missing, if time allows.

## Better source for Steps 8-10

[`../diffs_nissan_vs_google/packages_services_Car.diff`](../diffs_nissan_vs_google/packages_services_Car.diff)
is a real `git diff nissan_u_ccs2_release..android14-release` (both Android 14) — a cleaner signal than
diffing across Android versions (`a14full` vs `a17full`) since it can't confuse "AOSP changed this between
versions" with "Nissan changed this." It surfaced [Step 8](step8_gain_context_enrichment.md),
[Step 9](step9_suspend_wake_volume_limit.md), and [Step 10](step10_forced_mute_volume_blocking.md) — three
Nissan features the `a14full`/`a17full` comparison missed. `frameworks_base.diff` and `frameworks_av.diff`
in the same folder are unreviewed as of this writing and may contain platform-level Nissan customizations
outside `car/audio` — check that folder's `README.md` for scope before reading them.
