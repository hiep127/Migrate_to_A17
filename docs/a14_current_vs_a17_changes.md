# AOSP A14→A17 Changelog vs. `/a14full` (Nissan) — File-by-File Comparison

**What this is:** the `Migrate_to_A17` folder is the **Google AOSP** changelog for
`service/src/com/android/car/audio/` across A14 → A15 → A16 → A17. This document maps each AOSP change
onto the **actual code in `/a14full`** (the Nissan fork) and states, per file, what exists today and
what the migration would have to add/change.

**Baseline finding:** `/a14full` `car/audio` = **stock AOSP A14 + Nissan patches**. None of the AOSP
A15/A16 new files exist; `CarAudioZonesHelper` is still a class (XML v1–v3); `CarAudioService` has a
single `AudioPolicy`; `CarAudioFocus` uses non-fade focus dispatch. So essentially the **entire A15,
A16 and A17 car/audio delta is still to be applied**, on top of existing Nissan customizations.

**Reliability note:** the per-file detail below is taken from `raw/AOSP_Car_Audio_File_Diff.md` +
`raw/AOSP_Car_Audio_Changelog.md`, which are derived from real git patches (patch sizes A14→A15 =
10,942 lines, A15→A16 = 5,754, A16→A17 = 485). The `changelogs/` + `master/` files are release-note
summaries (they say so in their own headers) and cover extra layers (core Audio HAL / CAP engine, app
hardening) that are **outside** the `car/audio` diff scope — treat those as unverified until the real
AOSP diffs are fetched.

Source path in the fork: `/home/worker/a14full/android/packages/services/Car/service/src/com/android/car/audio/`

---

## Legend
- **ABSENT** — AOSP added this file; not in `/a14full` yet → must be added.
- **STOCK A14** — present, matches AOSP A14 baseline → apply AOSP delta as-is.
- **NISSAN-FORKED** — present but Nissan already modified it → AOSP delta must be **merged**, conflict risk.

---

## A14 → A15 (biggest delta — 10,942-line patch)

### New files AOSP added (all ABSENT in `/a14full`)

| AOSP new file | Status in `/a14full` | Purpose to bring in |
|---|---|---|
| `AudioManagerWrapper.java` | ABSENT | Wraps all `AudioManager` calls (testability). **Touches almost every other file** — this is the spine of the A15 refactor. |
| `CarActivationVolumeConfig.java` | ABSENT | Per-group min/max activation volume (%). |
| `CarAudioDeviceCallback.java` | ABSENT | Dynamic device add/remove → re-route. |
| `CarAudioFadeConfigurationHelper.java` | ABSENT | Parses `car_audio_fade_configuration.xml`. |
| `CarAudioParserUtils.java` | ABSENT | XML parse helpers extracted from `CarAudioZonesHelper`. |
| `CarAudioPlaybackMonitor.java` | ABSENT | Tracks active playback → drives activation volume. (Note: fork has a different `CarAudioPlaybackCallback.java`.) |
| `CarAudioProtoUtils.java` | ABSENT | Protobuf dump of fade config. |
| `CarAudioServerStateCallback.java` | ABSENT | Audioserver-crash → reinit. |

### Modified files (present in `/a14full`, currently STOCK A14 unless noted)

| File | `/a14full` today | AOSP A15 change to apply |
|---|---|---|
| `CarAudioService.java` | **single** `mAudioPolicy` (line 331) | Split into 4 policies (`mVolumeControlAudioPolicy`, `mFocusControlAudioPolicy`, `mRoutingAudioPolicy`, `mFadeManagerConfigAudioPolicy`); constructor gains `AudioManagerWrapper` + `audioFadeConfigPath`; new fields for min/max-activation + fade; defer-init if audioserver down. |
| `CarAudioFocus.java` | **NISSAN-FORKED** — `sendFocusLossLocked(loser, lossType)` plain dispatch (line 187) | Add cross-fade variant `sendFocusLossLocked(loser, lossType, winner, shouldFade, transientFadeConfig)` using `dispatchAudioFocusChangeWithFade`; switch usage-based eval; `AudioManager`→`AudioManagerWrapper`. **Merge carefully — Nissan already patched this file.** |
| `CarAudioZonesHelper.java` | `final class`, XML **v1–v3** (lines 61,98–107) | In A16 becomes interface (see below); at A15 it keeps parsing but gains v4 elements. |
| `CarAudioDeviceInfo.java` | STOCK A14 | Major refactor to `AudioDeviceAttributes`; add `isActive()`, dynamic `audioDevicesAdded/Removed()`. |
| `CarAudioDynamicRouting.java` | STOCK A14 | Only route selected+active configs; default config always appended last as fallback. Signature change. |
| `CarAudioContext.java` | STOCK A14 | Remove `getContextsToDuck()`; add `getLegacyContextForUsage()`, context-id in `AudioAttributesWrapper`. |
| `CarAudioVolumeGroup.java` | STOCK A14 | `contextToAddress`→`contextToDeviceInfo`; add `CarActivationVolumeConfig`. |
| `CarAudioZone.java` | STOCK A14 | `getCurrentAudioDevices()` (AudioDeviceAttributes); add `getVolumeGroupForAudioAttributes()`, device add/remove, selected/active state. |
| `CarAudioZoneConfig.java` | STOCK A14 | Add fade-config fields + `isSelected`/`isActive`; `updateVolumeDevices()`. |
| `CarVolumeGroup.java` | STOCK A14 | address-based→device-info mapping; add activation-gain + `updateDevices(boolean)`. |
| `CarVolumeGroupFactory.java` | STOCK A14 | `AudioManager`→`AudioManagerWrapper`; mapping type change. |
| `CarZonesAudioFocus.java` | STOCK A14 | `AudioManagerWrapper` + `CarAudioFeaturesInfo`; `@GuardedBy` tightening. |
| `CoreAudioVolumeGroup.java` | **NISSAN-FORKED** (#222006: `CoreAudioVolumeGroupHelper`, `storeAndPersistGainIndexLocked`, lines 74,136…) | `AudioManagerWrapper`; drop `isPlatformVersionAtLeastU` guards; `isVolumeGroupMuted`/`adjustVolumeGroupVolume`; add `updateDevices(boolean)`. **Merge with the #222006 default-volume/persist patch.** |
| `CoreAudioVolumeGroupCallback.java` | STOCK A14 | `AudioManagerWrapper`. (Executor location oscillates A15→A16→A17 — see below.) |
| `FocusInteraction.java` | STOCK A14 | Drop `CarAudioContext` from ctor; usage-based `evaluateRequest(int usage,…)`. |
| `hal/AudioControlWrapper*.java`, `hal/HalAudio*.java` | STOCK A14, **still in `hal/`** | Interface updates for A15 HAL capabilities. |

---

## A15 → A16 (5,754-line patch — the "config on HAL" transition for zones)

### New files AOSP added (all ABSENT in `/a14full`)

| AOSP new file | Status | Purpose |
|---|---|---|
| `AudioControlZoneConverter.java` | ABSENT | Converts HAL AIDL `AudioZone` → `CarAudioZone`. |
| `AudioControlZoneConverterUtils.java` | ABSENT | Low-level HAL→service conversion helpers (~529 lines). |
| `CarAudioZonesHelperAudioControlHAL.java` | ABSENT | **Loads zone topology from the AudioControl HAL (`getAudioZones()`) — no `car_audio_configuration.xml` needed.** This is the confirmed "audio config on HAL" change. |
| `CarAudioZonesHelperImpl.java` | ABSENT | The old XML parser body, now implementing the interface; adds XML **v4** + `<deviceConfigurations>` + `<activationVolumeConfigs>`. |
| `TEST_MAPPING` | ABSENT | CI test routing. |

### Structural / modified

| File | `/a14full` today | AOSP A16 change |
|---|---|---|
| `CarAudioZonesHelper.java` | `final class` (~XML v1–v3) | **Class → interface** (8 signatures incl. `useCoreAudioRouting()`/`useCoreAudioVolume()`). `CarAudioService.init()` picks HAL impl vs XML impl at runtime via `mAudioControlWrapper.supportsAudioZones()`. |
| `hal/AudioControlWrapper.java` | STOCK A14 | Add `supportsAudioZones()`, `getAudioZones()`, `getAudioDeviceConfiguration()`. |
| `hal/AudioControlWrapperAidl.java` | STOCK A14 (AIDL up to v3 feature-gated) | Implement `getAudioZones()` AIDL call. |
| `hal/AudioControlWrapperV1/V2.java` | STOCK A14 | Stub the new zone API (unsupported). |
| `CarAudioService.java` | single policy, XML-only load | Async init (`waitForInitComplete`); HAL-first zone load w/ XML fallback; policies on main looper; persist-fade-balance feature. |
| `CarAudioUtils.java` | STOCK A14 | Absorb `generateCarAudioDeviceInfos()` etc. from the helper. |
| `CarVolumeGroup.java` / `CarAudioZone.java` / `CarAudioZoneConfig.java` | STOCK A14 | Thread through `useCoreAudioRouting`; add/remove event logger. |
| `ContentObserverFactory.java` / `CoreAudioVolumeGroupCallback.java` / `FocusInteraction.java` | STOCK A14 | Handler/executor plumbing changes (oscillate again in A17). |

> **Not in these car/audio patches:** the core-Audio-HAL **CAP engine** move to `getEngineConfig()` /
> `STRATEGY_` prefix that the release-note `master/` doc describes lives in `frameworks/av` +
> `android.hardware.audio.core`, a different repo. It is a separate workstream and is **unverified**
> here — confirm against real diffs before planning it. (`/a14full` currently uses the static-XML +
> parameter-framework CAP engine and lowercase strategy names, so if that AOSP change is real it is
> also a full delta.)

---

## A16 → A17 (small — 485-line patch, in-place only, **no new files**)

All modifications to files that will already exist post-A16. Mostly threading cleanup + reverts:

| File | AOSP A17 change |
|---|---|
| `CarAudioService.java` | Remove `mPersistFadeBalanceLevels`; audioserver callback → `getMainExecutor()`; replace `changeAudioPolicyForConfigChangeLocked()` with explicit `setupRoutingAudioPolicyLocked → setAllUserIdDeviceAffinities → swapRoutingAudioPolicy` (fixes a device-affinity bug on config switch). |
| `CarAudioFocus.java` | Remove `TimingsTraceLog` from focus hot path. |
| `CoreAudioVolumeGroup.java` | `updateDevices(boolean useCoreAudioRouting)` — early-return when core routing on. |
| `CarVolumeGroup.java` | Remove `mEventLogger`; restore `updateDevices(boolean)`. |
| `ContentObserverFactory.java` / `FocusInteraction.java` / `CoreAudioVolumeGroupCallback.java` | Main-looper / executor-at-init reverts. |
| `CarAudioZone.java` / `CarAudioZoneConfig.java` / `CarAudioVolumeGroup.java` | Propagate `useCoreAudioRouting`; javadoc fix. |

> **Oscillation warning** (from the diff's own summary table): `ContentObserverFactory.createObserver`,
> `CoreAudioVolumeGroupCallback.init`, `CarVolumeGroup.updateDevices`, `TimingsTraceLog`, and
> `mPersistFadeBalanceLevels` each change direction across A15→A16→A17. If migrating straight A14→A17,
> **apply only the net A17 end-state**, don't replay the intermediate flip-flops.

---

## Nissan files with no AOSP-changelog counterpart

These exist in `/a14full` but aren't mentioned in the AOSP `car/audio` changelog — they are either
Nissan/OEM additions or A14 files AOSP didn't touch. They won't be changed by the AOSP merge but they
**interact** with the changed classes and must be re-tested:

`CoreAudioVolumeGroupHelper.java` (#222006), `CarAudioPlaybackCallback.java`, `CarAudioContextInfo.java`,
`CarAudioGainMonitor/GainCallbackHandler/GainConfigInfo.java`, `CarAudioMirrorRequestHandler.java`,
`CarAudioModuleChangeMonitor.java`, `CarAudioPolicyVolumeCallback.java`, `CarAudioPowerListener.java`,
`CarDucking*.java`, `CarHalAudioUtils.java`, `CarVolumeCallbackHandler/EventHandler.java`,
`CarVolumeGroupMuting.java`, `CarVolume.java`, `CoreAudioHelper.java`, `MediaRequestHandler.java`,
`CarAudioZonesHelperLegacy.java`, `CarAudioZonesValidator.java`, etc.

---

## Bottom line for the migration

1. **A14→A15 is the heavy lift** (~11k lines): 8 new classes + the `AudioManagerWrapper` refactor that
   touches nearly every file. Two files (`CarAudioFocus`, `CoreAudioVolumeGroup`) are **already
   Nissan-forked** — those are the real merge-conflict hotspots.
2. **A15→A16** adds the **zone-config-via-AudioControl-HAL** path (`getAudioZones()` +
   `CarAudioZonesHelperAudioControlHAL`) and turns `CarAudioZonesHelper` into an interface. Deciding
   whether Nissan adopts HAL-delivered zones or keeps `car_audio_configuration.xml` is the key design
   call here.
3. **A16→A17** is small and mostly threading cleanup — take the net end-state, skip the oscillations.
4. **Separate, unverified track:** the core-Audio-HAL / CAP-engine "config on HAL" story from the
   release-note docs is outside these patches — validate it against real `frameworks/av` +
   `hardware/interfaces/audio` diffs before committing to it.

### To make the AOSP side verifiable on this machine
The Nissan Car repo has none of the AOSP release tags, so the diffs can't be regenerated locally as-is:
```bash
cd /home/worker/a14full/android/packages/services/Car
git remote add aosp https://android.googlesource.com/platform/packages/services/Car
git fetch aosp --tags
git diff aosp/android-14.0.0_r1..aosp/android-16.0.0_r1 -- service/src/com/android/car/audio/
```
