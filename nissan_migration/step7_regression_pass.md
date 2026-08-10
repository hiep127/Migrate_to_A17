# Step 7 — Full regression pass

[← back to index](00_README.md) · Depends on: Steps 0-6, 8-10 complete

> **Note on step numbering**: this step was originally the final gate after Steps 1-6. Steps 8-10 were
> added later (from the [authoritative Nissan-vs-Google diff](step0_resync_source.md#addendum-a-materially-better-source-became-available))
> and should be implemented **before** this regression pass runs, not after — treat this as the true final
> step regardless of file-name ordering.

## Goal

Confirm all 9 features implemented in Steps 1-6 and 8-10 compose correctly with each other and with the
AOSP A17 baseline, and that the Nissan device config layer now drives real behavior instead of silently
falling back to stock AOSP.

## Build and boot

- Rebuild both Nissan device targets: `aivi2_n_full` and `aasp_n`.
- Boot each; confirm no boot loop, no crash in `com.android.car` / `system_server`.

## System-level checks

- `dumpsys car_service` (audio section): no config-parse errors; zones/groups/policies match what's
  expected from the existing `car_audio_configuration.xml` (still v3, per the verification pass — this
  migration does not require a v4 bump).
- `dumpsys audio`: no unexpected policy registration failures across the 4 split `AudioPolicy` instances
  (`mVolumeControlAudioPolicy`/`mFocusControlAudioPolicy`/`mRoutingAudioPolicy`/
  `mFadeManagerConfigAudioPolicy`).
- Confirm the RRO overlay flags already set to `true` in
  `device/nissan/.../overlay/CarServiceOverlayNissanFull/res/values/config.xml`
  (`audioUseDynamicRouting`, `audioUseCarVolumeGroupMuting`) now drive the newly-added Nissan logic from
  Steps 1-6 and 8-10, instead of the stock AOSP fallback behavior that was silently running before this
  migration.

## Cross-feature composition checks

- MaxVolumeStartup ([Step 1](step1_max_volume_startup.md)) + `Settings.Global` persistence
  ([Step 2](step2_settings_global_persistence.md)): fresh boot clamps MEDIA volume correctly; a later boot
  with a persisted value does not re-clamp.
- Audio Off ([Step 6](step6_audio_off_mode.md)) + master-mute callback: toggling Audio Off does not
  interfere with a genuine `setMasterMute()` call, and vice versa.
- OEM gain callback ([Step 3](step3_oem_gain_callback_wiring.md)) does not double-apply gain changes
  alongside the stock HAL-direct path during normal ignition-cycle gain events.
- `ZoneAudioPlaybackCallback` hardening ([Step 4](step4_zone_playback_callback_hardening.md)) holds up
  under the same playback-config-changed load that Audio Off's focus-loser restoration
  ([Step 6](step6_audio_off_mode.md)) generates (both touch focus/playback state around the same time on a
  mute toggle).
- **Steps 8, 9, and 10 all touch `CarVolumeGroup.onAudioGainChanged()`/`CarAudioService`'s gain-callback
  code paths, plus Step 6's mute path — test them as one cluster, not independently:**
  - Trigger a suspend/wake cycle ([Step 9](step9_suspend_wake_volume_limit.md)) while a shared device port
    exists between a media and non-media group ([Step 8](step8_gain_context_enrichment.md)); confirm the
    suspend-exit limit only applies to the media group and the shared-port active-group disambiguation still
    picks the right group.
  - Trigger a forced-mute/cyber HAL condition ([Step 10](step10_forced_mute_volume_blocking.md)) while
    NAV_DUCKING is also active ([Step 8](step8_gain_context_enrichment.md)); confirm the volume-set block
    and the NAV_DUCKING exposure to the OEM plugin don't interfere with each other.
  - Confirm none of Steps 8-10's changes to `onAudioGainChanged()` broke Audio Off's media-group mute
    detection ([Step 6](step6_audio_off_mode.md)), since both read/react to volume-group state changes.

## Standard regression (unaffected areas — confirm no behavior change)

- Volume keys (up/down/mute) across all zones.
- Focus arbitration: media vs. navigation vs. phone vs. notification, including the BT voice-call guard
  from [Step 5](step5_bt_voice_call_focus_block.md).
- Ducking behavior (HAL-signaled and app-requested), including NAV_DUCKING exposure to the OEM plugin
  ([Step 8](step8_gain_context_enrichment.md)).
- ANC (if applicable on the target hardware) — unrelated to this migration, confirm no regression.
- Guest profile switch and general multi-user switching, beyond just the RING/CALL bar fix from
  [Step 2](step2_settings_global_persistence.md).
- Suspend/hibernation/wake cycles generally ([Step 9](step9_suspend_wake_volume_limit.md)) — not just the
  volume-limit behavior, but that the new `CarPowerManagementService` listener registration doesn't affect
  boot time or other power-state-dependent car service behavior.
- Explicit volume-set requests (`CarAudioManager.setGroupVolume()`) under normal (non-blocked) conditions
  ([Step 10](step10_forced_mute_volume_blocking.md)) — confirm the new `shouldBlock` check adds no
  measurable latency or false-positive blocking.

## Sign-off

Once all of the above pass on both `aivi2_n_full` and `aasp_n`, this migration is complete. Re-check the
[Out of scope](00_README.md#out-of-scope--confirmed-stock-aosp-already-or-unconfirmed) list in the index
only if something breaks in an area that list claims is unaffected stock AOSP behavior — that would mean
the earlier research missed something and needs re-verification.
