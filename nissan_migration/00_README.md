# Migrate Nissan Car-Audio Behavior onto A17

## Why this folder exists

Verification work (see
[`../master/FINAL_Conclusion_A17full_Verification_And_Remaining_Work.md`](../master/FINAL_Conclusion_A17full_Verification_And_Remaining_Work.md))
established that `a17full`'s `packages/services/Car/service/src/com/android/car/audio/` is a clean AOSP A17
tree — the Google-side structural migration (13 new files, `CarAudioZonesHelper` as interface, 4-way
`AudioPolicy` split, device-based `CarVolumeGroup`, IAudioControl AIDL v6, etc.) is already complete, but
**zero Nissan behavioral code exists anywhere in it** (0 Nissan/LGE authors across 661 commits). Meanwhile
the Nissan device overlays (`device/nissan/.../CarServiceOverlayNissanFull/res/values/config.xml`) already
flip `audioUseDynamicRouting`/`audioUseCarVolumeGroupMuting` to `true` — i.e. the config layer is already
pointed at Nissan behavior that the Java layer doesn't implement yet.

This folder is the re-implementation project that closes that gap.

## Sources

Steps 1-7 are sourced from direct, on-disk reading of the `a14full` checkout (`nissan_u_ccs2_release`),
cross-diffed against `a17full`'s stock baseline to confirm what's genuinely Nissan vs. already-stock AOSP.

Steps 8-10 are sourced from
[`../diffs_nissan_vs_google/packages_services_Car.diff`](../diffs_nissan_vs_google/packages_services_Car.diff),
a real `git diff` between Nissan's `nissan_u_ccs2_release` branch and Google's `android14-release` tag —
both on Android 14, so it isolates pure Nissan customization with zero AOSP-version noise. If more budget
exists, re-running Steps 1-7 against this same diff (instead of the `a14full`/`a17full` comparison) is worth
doing — it's a strictly better signal and is how Steps 8-10 were found in the first place.

One cross-branch gap: the [Step 4](step4_zone_playback_callback_hardening.md) ANR fix exists only on the
sibling `alliance_u_release` branch family, never merged into the Nissan line — see
[Step 0](step0_resync_source.md).

## Steps

| Step | File | What it covers | Risk |
|---|---|---|---|
| 0 | [step0_resync_source.md](step0_resync_source.md) | Confirm source-of-truth state; pull the one real cross-branch gap found (Step 4's ANR fix) | — (prerequisite) |
| 1 | [step1_max_volume_startup.md](step1_max_volume_startup.md) | `CoreAudioVolumeGroupHelper.java` + MaxVolumeStartup boot volume ceiling | Low |
| 2 | [step2_settings_global_persistence.md](step2_settings_global_persistence.md) | `Settings.Global` persistence for MEDIA/RING/CALL (fixes guest volume bar + CP-song max-volume bugs) | Low |
| 3 | [step3_oem_gain_callback_wiring.md](step3_oem_gain_callback_wiring.md) | OEM gain-callback wiring to the Alliance plugin | Low |
| 4 | [step4_zone_playback_callback_hardening.md](step4_zone_playback_callback_hardening.md) | `ZoneAudioPlaybackCallback` ANR fix (confirmed real, cross-branch gap) + null/shutdown guards (verify-then-port) | Low |
| 5 | [step5_bt_voice_call_focus_block.md](step5_bt_voice_call_focus_block.md) | Block duplicate BT voice-call focus requests | Low |
| 6 | [step6_audio_off_mode.md](step6_audio_off_mode.md) | Audio Off mode + master-mute callback conflation | High — do late |
| 7 | [step7_regression_pass.md](step7_regression_pass.md) | Full build/boot regression across both device targets | — (final gate) |
| 8 | [step8_gain_context_enrichment.md](step8_gain_context_enrichment.md) | NAV_DUCKING exposure to OEM plugin + shared-device-port gain-active disambiguation | Low-Med |
| 9 | [step9_suspend_wake_volume_limit.md](step9_suspend_wake_volume_limit.md) | Suspend-exit volume-limit scoped to media groups + power-state limit reset on wake | Low-Med |
| 10 | [step10_forced_mute_volume_blocking.md](step10_forced_mute_volume_blocking.md) | Block explicit volume-set requests during forced-mute/cyber HAL conditions (corrects an earlier wrong "nothing to port" call) | Low |

Steps 8-10 touch the same `CarVolumeGroup.onAudioGainChanged()`/`CarAudioService` gain-callback code paths
as each other and as parts of Step 6 — implement and test them together rather than in strict isolation,
and do all of them (plus Step 6) before the final [Step 7](step7_regression_pass.md) regression pass.

## Out of scope — confirmed stock AOSP already

Do **not** spend effort on these; re-verify only if something breaks during Step 7:

- **Occupant-zone filter / mirror-request cleanup on user reassignment** — was a real Nissan A14 feature
  (`CarAudioService.removePrimaryZoneRequestForOccupantLocked()` + `removeAudioMirrorForZoneId()`, called
  from `updateUserForOccupantZoneLocked()`; `MediaRequestHandler.getAssignedRequestIdForOccupantZoneId()`),
  but this logic is **already present** in `a17full` today, wired at `CarAudioService.java:4024-4026` and
  `MediaRequestHandler.java:244` — AOSP independently adopted equivalent logic by A17. No work needed.
- **Mirror handling patches (routing/HAL-command logic itself)** — unmodified stock AOSP.
- **Restore volume on unmute** — `CarVolumeGroup`/`CoreAudioVolumeGroup` mute/unmute design never overwrites
  the underlying gain index; restoration is inherent to stock AOSP, byte-identical in both trees.

Cyber-mute's HAL-reason-handling primitive (`Reasons.FORCED_MASTER_MUTE`/`TCU_MUTE`/`REMOTE_MUTE` via
`CarAudioGainMonitor.shouldBlockVolumeRequest()`) is also stock AOSP already in `a17full` — but wiring that
primitive into `CarAudioService.setGroupVolume()` is **not**, and is real work: see
[Step 10](step10_forced_mute_volume_blocking.md).

See [unsure_items.md](unsure_items.md) for items investigated but not confirmed enough to include as work
items: the "Siri/E-call volume" fix, `canSwapCallOrRingerClientRequest()`'s second call site, and
`CarAudioContextInfo.equals()`/`hashCode()`.
