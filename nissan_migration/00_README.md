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

## Two research passes, two levels of confidence

**Steps 1-7** came from three parallel deep-dive research passes over the actual A14 fork, cross-diffed
against the A17 stock baseline in `a17full` to positively confirm what's genuinely Nissan vs. stock AOSP.
That method mixes two kinds of change (AOSP's own A14→A17 evolution, and Nissan's A14 customization) and
relies on knowing the right method/class names to grep for — real, but imperfect.

**Steps 8-10** came from a materially stronger source added later:
[`../diffs_nissan_vs_google/packages_services_Car.diff`](../diffs_nissan_vs_google/packages_services_Car.diff),
a real `git diff` between Nissan's actual `nissan_u_ccs2_release` branch and Google's actual
`android14-release` tag — **both on Android 14**, so it isolates pure Nissan customization with zero
AOSP-version noise. Reviewing that diff surfaced three real, substantial Nissan features Steps 1-7's
research missed entirely (gain-callback context enrichment, suspend-wake volume limiting, and — importantly
— a real gap in what the "cyber-mute" out-of-scope entry below originally claimed). It also **confirmed**
that most of Steps 1-7 were right, and **corrected** the occupant-zone/mirror-handling conclusion below.
Recommendation: if more budget exists, re-run Steps 1-7's research using this same
diff-against-real-Google-tag method instead of the original A14-fork/A17-diff method — it's a strictly
better signal.

**Important caveat surfaced by the original research:** the currently-checked-out `a14full` working tree
(`nissan_u_ccs2_release`, HEAD `dbe12bde8d8`) is largely fine (later re-verified — see
[Step 0](step0_resync_source.md)), but one real gap was found: the
[Step 4](step4_zone_playback_callback_hardening.md) ANR fix exists only on a sibling `alliance_u_release`
branch family, never merged into the Nissan line. [Step 0](step0_resync_source.md) has the full, corrected
account.

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

## Out of scope — confirmed stock AOSP already, or unconfirmed

Do **not** spend effort on these; re-verify only if something breaks during Step 7:

- **Occupant-zone filter / mirror-request cleanup on user reassignment** — **correction**: this *was* a
  real, substantial Nissan A14 feature (`CarAudioService.removePrimaryZoneRequestForOccupantLocked()` +
  `removeAudioMirrorForZoneId()`, called from `updateUserForOccupantZoneLocked()`; `MediaRequestHandler
  .getAssignedRequestIdForOccupantZoneId()`), confirmed via the authoritative Nissan-vs-Google diff — the
  earlier "not found, matches stock" conclusion was based on the A14-fork research not knowing the right
  method names to check. However, direct grep against `a17full` today shows **all of this logic is already
  present**, wired at `CarAudioService.java:4024-4026` and in `MediaRequestHandler.java:244`. Likely
  explanation: AOSP itself adopted equivalent logic by A17, independent of Nissan. Net effect is the same
  as originally concluded — **no work needed** — but for the correct reason.
- **Mirror handling patches (routing/HAL-command logic itself)** — still confirmed unmodified stock AOSP,
  no correction needed here.
- **Cyber-mute** — **partially corrected**. The HAL-reason-handling primitive itself
  (`Reasons.FORCED_MASTER_MUTE`/`TCU_MUTE`/`REMOTE_MUTE` via `CarAudioGainMonitor.shouldBlockVolumeRequest()`)
  is genuinely stock AOSP, unmodified, already in `a17full` — that part of the original conclusion holds.
  But the authoritative diff found a second piece the original research missed: Nissan wires that same check
  into `CarAudioService.setGroupVolume()` to reject explicit volume-set requests during a blocking HAL
  condition, which is **not** present in `a17full`. See [Step 10](step10_forced_mute_volume_blocking.md) —
  this is real work, not out of scope.
- **Restore volume on unmute** — still confirmed: `CarVolumeGroup`/`CoreAudioVolumeGroup` mute/unmute design
  never overwrites the underlying gain index; restoration is inherent to stock AOSP, byte-identical in both
  trees.
- **Siri / E-call volume fix** — still unconfirmed. Exhaustive search (Car project-wide, plus a scan of the
  authoritative diff's `CarAudioFocus.java`/`FocusInteraction.java` sections) found no feature matching this
  description. The closest adjacent code is `canSwapCallOrRingerClientRequest()` in `CarAudioFocus.java`
  (already present in `a17full`, used in the focus-holders matching path; the authoritative diff shows
  Nissan also applies it in the *blocked/pending-request* matching path, one call site A17 may or may not
  already have — low priority, spot-check during [Step 5](step5_bt_voice_call_focus_block.md) or
  [Step 6](step6_audio_off_mode.md) implementation rather than as a standalone step). Recommend getting the
  actual ticket/CR number for "Siri/E-call volume" before assuming further work is needed.
- **`CarAudioContextInfo.equals()`/`hashCode()`** — a small Nissan-only addition (proper value-equality,
  likely needed for `Set`/`Map` usage or test assertions elsewhere). Minor; add only if something that
  depends on it surfaces during implementation of the other steps — not worth a standalone step.
