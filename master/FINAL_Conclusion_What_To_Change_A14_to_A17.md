# FINAL CONCLUSION — What Must Change to Go From A14 to A17

## Car Audio Framework · Net End-State Delta (not version-by-version)

> **What this document is.** Every other file in `Migrate_to_A17/` is organised *per version transition*
> (A14→A15, A15→A16, A16→A17). This one collapses all three into a **single net delta**: if you jump
> straight from the Android 14 baseline to the Android 17 end-state, this is the complete list of what
> changes — and nothing else.
>
> **Why that matters.** Several APIs changed direction two or three times across the three releases
> (see §9). Replaying the transitions in order would make you write code twice and delete it once.
> Everything below is stated as the **final A17 form only**.
>
> **Baseline verified in-tree** (`/a14full`, `nissan_u_ccs2_release`): `CarAudioService` has a single
> `mAudioPolicy` (line 331), `CarAudioZonesHelper` is still a `final class` parsing XML v1–v3, the
> `hal/` subfolder is still present, and **none** of the AOSP A15/A16 new files exist. So the whole
> A15+A16+A17 `car/audio` delta is still to be applied — roughly **17,200 patch lines**
> (10,942 + 5,754 + 485).
>
> **Confidence marking.** §1–§9 come from real AOSP git patches cross-checked against the actual fork.
> §10 is release-note-sourced and **unverified locally**. §11 is a contradiction that must be resolved
> before work starts.
>
> **⚠️ Step 1 audit update (2026-08-05).** This doc's baseline assumption — a clean A14 tree with none
> of the new files present — does not match `/a17full` (branch `nissan_c_join_aivi2_fc`), which already
> has the full Step 1 file set. See
> [`STEP1_AUDIT_RESULT_Actual_A17_State.md`](STEP1_AUDIT_RESULT_Actual_A17_State.md) for the full,
> AOSP-verified findings before using this doc's §9 oscillation table or its `hal/`-layout assumptions
> for that tree — several rows there no longer apply as written.

---

## §0. The conclusion in ten lines

1. **Add 13 new files** to `car/audio/` (8 from A15, 5 from A16). `AudioManagerWrapper` is the spine — it touches nearly every other file.
2. **Convert `CarAudioZonesHelper` from a class to an interface**, and move its 1,000-line XML body into a new `CarAudioZonesHelperImpl`.
3. **Rewrite volume-group mapping** from *address-based* (`String`) to *device-based* (`CarAudioDeviceInfo`), and **split mute** into apply-vs-persist.
4. **Split `CarAudioService`'s single `AudioPolicy` into four**, and make init async + audioserver-crash-tolerant.
5. **Audio policies and framework callbacks run on the main looper** — but the A14 baseline is *already* there, so this nets to no change (see §9). The `HandlerThread` is **not** removed; it survives for internal posted work.
6. **Two new optional features** to accept or decline: system-enforced fade on focus loss, and min/max activation volume.
7. **XML config stays valid at v3** unless you adopt those two features — then bump to **v4** and add `car_audio_fade_configuration.xml`.
8. **No `IAudioControl` version bump is required** unless you choose HAL-delivered zones. Recommendation: don't — keep XML, `supportsAudioZones()` = false.
9. **Seven Nissan-forked files are real merge conflicts**; five more need re-validation without being edited by Google.
10. **Three A17 items live outside `car/audio` entirely** (background-audio hardening, assistant volume stream, CAP-on-HAL) and are separate projects.

---

## §1. Files to ADD — 13 new files

None of these exist in the fork. They drop in cleanly; the cost is wiring, not conflict.

| File | Origin | What it does | Why you need it |
|---|---|---|---|
| `AudioManagerWrapper.java` | A15 | Wraps every `android.media.AudioManager` call | **Prerequisite for everything else.** Nearly every modified file changes its `AudioManager` field to this type. Add it first. |
| `CarActivationVolumeConfig.java` | A15 | Per-group min/max activation % + invocation type | Required by the new `CarVolumeGroup` constructor — needed even if you never enable the feature |
| `CarAudioDeviceCallback.java` | A15 | `AudioDeviceCallback` → `CarAudioService.onAudioDevicesAdded/Removed()` | Dynamic device connect/disconnect re-routing |
| `CarAudioFadeConfigurationHelper.java` | A15 | Parses `/vendor/etc/car_audio_fade_configuration.xml` | Only exercised if fade is enabled; still compiled in |
| `CarAudioParserUtils.java` | A15 | XML parse helpers extracted from the zones helper | De-duplication; required by `CarAudioZonesHelperImpl` |
| `CarAudioPlaybackMonitor.java` | A15 | Tracks active playback per zone → drives activation volume | Sits alongside the fork's existing `CarAudioPlaybackCallback` — they are *different* classes |
| `CarAudioProtoUtils.java` | A15 | Protobuf dump of fade config for bugreports | Dump-only |
| `CarAudioServerStateCallback.java` | A15 | Detects audioserver crash → triggers reinit | New resilience; interacts with boot order |
| `AudioControlZoneConverter.java` | A16 | HAL AIDL `AudioZone` → `CarAudioZone` | Only used on the HAL-zone path; compiled regardless |
| `AudioControlZoneConverterUtils.java` | A16 | ~529 lines of HAL→service type conversion | ditto |
| `CarAudioZonesHelperAudioControlHAL.java` | A16 | Loads zone topology from `getAudioZones()` | ditto — **keep it inert** (see §5, decision 3) |
| `CarAudioZonesHelperImpl.java` | A16 | The old XML parser body, now implementing the interface (~1,234 lines) | **This is the file that keeps XML working.** Mandatory. |
| `TEST_MAPPING` | A16 | CI test routing | Optional but cheap |

---

## §2. The one STRUCTURAL change

`CarAudioZonesHelper.java`: **`final class` (~1,056 lines of XML parsing) → `interface` (8 signatures)**

```java
interface CarAudioZonesHelper {
    SparseArray<CarAudioZone> loadAudioZones() throws IOException, XmlPullParserException;
    CarAudioContext getCarAudioContext();
    SparseIntArray getCarAudioZoneIdToOccupantZoneIdMapping();
    List<CarAudioDeviceInfo> getMirrorDeviceInfos();
    boolean useCoreAudioRouting();
    boolean useCoreAudioVolume();
    boolean useHalDuckingSignalOrDefault();
    boolean useVolumeGroupMuting();
}
```

`CarAudioService.init()` picks the implementation at runtime:

```java
boolean halSupportsZoneConfig = mAudioControlWrapper.supportsAudioZones();
mCarAudioZonesHelper = halSupportsZoneConfig
        ? new CarAudioZonesHelperAudioControlHAL(mAudioControlWrapper, ...)   // HAL topology
        : new CarAudioZonesHelperImpl(mContext, ...);                        // XML — Nissan path
```

Keeping `supportsAudioZones()` false selects the XML impl and makes the entire HAL-zone branch dead code. That is the recommended posture.

---

## §3. Files to MODIFY — net A17 end-state per file

Grouped by whether the fork already touched them. **"Net A17"** means: apply this form once; do not implement the A15 or A16 intermediate form.

### 3a. Stock A14 in the fork — mechanical, low risk

| File | Net A17 state to reach |
|---|---|
| `CarAudioDeviceInfo.java` | Constructor takes `(AudioManagerWrapper, AudioDeviceAttributes)` instead of `(AudioManager, AudioDeviceInfo)`. Add `isActive()`, `audioDevicesAdded/Removed()`, `setAudioDeviceInfo()`, `getAudioDevice()` (replaces `getAudioDeviceInfo()`). New constants `DEFAULT_NUM_CHANNELS=1`, `DEFAULT_ENCODING_FORMAT=ENCODING_PCM_16BIT`, `UNINITIALIZED_GAIN=-1`. |
| `CarAudioDynamicRouting.java` | `setupAudioDynamicRouting()` signature changes; routes **only selected + active** configs, and always appends the **default config last** as fallback. |
| `CarAudioContext.java` | Remove `mContextsToDuck` / `getContextsToDuck()`. Add `getLegacyContextForUsage(int)`. `AudioAttributesWrapper` gains `mCarAudioContextId` + a 2-arg constructor; `equals()`/`hashCode()` become context-id-based when the id is valid. |
| `CarAudioVolumeGroup.java` | Inherits the base constructor change (device map + `CarActivationVolumeConfig`); `calculateNewGainStageFromDeviceInfos()` moves inside `synchronized(mLock)`. Only standalone A17 edit is a javadoc fix. |
| `CarVolumeGroupFactory.java` | `AudioManager` → `AudioManagerWrapper`; internal map `SparseArray<String>` + `ArrayMap<String,…>` → `SparseArray<CarAudioDeviceInfo>`; passes the new `CarActivationVolumeConfig`. |
| `CarAudioZone.java` | `getCurrentAudioDeviceInfos()` → `getCurrentAudioDevices()` returning `AudioDeviceAttributes`. Add `getDefaultAudioZoneConfigInfo()`, `getVolumeGroupForAudioAttributes()`, `audioDevicesAdded/Removed()`. `init()` selects the default config + `setIsSelected(true)` + `updateVolumeDevices()`. **Net A17:** call sites read the flag from `mCarAudioContext.useCoreAudioRouting()` rather than taking it as a parameter. |
| `CarAudioUtils.java` | Absorbs `generateCarAudioDeviceInfos()`, `generateAddressToCarAudioDeviceInfoMap()`, `generateAddressToInputAudioDeviceInfoMap()` from the zones helper. Add `isMicrophoneInputDevice()`, `isInvalidActivationPercentage()`. |
| `ContentObserverFactory.java` | **Already at the A17 form — leave it alone.** `createObserver(ContentChangeCallback)` with no `Handler` parameter, binding `Looper.getMainLooper()` internally, is what the fork has today (`ContentObserverFactory.java:37`). Do **not** implement the A16 explicit-`Handler` form. |
| `CoreAudioVolumeGroupCallback.java` | `AudioManager` → `AudioManagerWrapper` is the only change. The executor plumbing is **already at the A17 form**: `init(Executor)` (`CoreAudioVolumeGroupCallback.java:43`), called as `init(mContext.getMainExecutor())` (`CarAudioService.java:1692`). Do **not** implement the A16 ctor-executor form. |
| `hal/AudioControlWrapper.java` | Add `supportsAudioZones()`, `getAudioZones()`, `getAudioDeviceConfiguration()`. |
| `hal/AudioControlWrapperAidl.java` | Implement `getAudioZones()`; the rest of the A15 HAL-capability updates. |
| `hal/AudioControlWrapperV1.java`, `hal/AudioControlWrapperV2.java` | Stub the new zone API as unsupported. |
| `hal/HalAudioDeviceInfo.java`, `hal/HalAudioFocus.java`, `hal/HalFocusListener.java` | Interface updates for dynamic device handling. |

### 3b. Nissan-forked in the fork — real merges

| File | Net A17 state to reach | Nissan code that lands in the same methods |
|---|---|---|
| `CarAudioService.java` | Single `mAudioPolicy` → **four**: `mVolumeControlAudioPolicy`, `mFocusControlAudioPolicy`, `mRoutingAudioPolicy`, `mFadeManagerConfigAudioPolicy`. Constructor gains `AudioManagerWrapper` + `audioFadeConfigPath`. New fields: `mUseMinMaxActivationVolume`, `mUseFadeManagerConfiguration`, `mAudioServerStateCallback`, `mAudioDeviceInfoCallback`, `mCarAudioPlaybackMonitor`, `mCarAudioFadeConfigurationHelper`. `init()` checks `isAudioServerRunning()` and defers; async init with `waitForInitComplete()`. HAL-first zone load with XML fallback. All 4 policy builders use `Looper.getMainLooper()`; audioserver callback uses `mContext.getMainExecutor()`. Config switch becomes the explicit 3-step `setupRoutingAudioPolicyLocked()` → `setAllUserIdDeviceAffinitiesToNewPolicyLocked()` → `swapRoutingAudioPolicyLocked()`. **Do not add** `mPersistFadeBalanceLevels`. | Audio-Off persist, MaxVolumeStartup, OEM gain-callback wiring, master-mute callback, occupant-zone filter, Siri/E-call volume, cyber-mute, mirror handling |
| `CarVolumeGroup.java` | Mapping `SparseArray<String> contextToAddress` + `ArrayMap<String,CarAudioDeviceInfo>` → **`SparseArray<CarAudioDeviceInfo> contextToDevices`**, plus a `CarActivationVolumeConfig` ctor parameter. Add `getMax/MinActivationGainIndex()`, `getActivationVolumeInvocationType()`, `handleActivationVolume()`, `isActive()`, `audioDevicesAdded/Removed()`, `validateDeviceTypes()`, `getAudioDeviceAttributes()`. **Mute splits** into `applyMuteLocked()` + `saveMuteStateToSettingsLocked()`. **Net A17:** `updateDevices(boolean useCoreAudioRouting)`; **no** `mEventLogger`. | MaxVolumeStartup apply, skip master-mute event, skip ducking volume-change event, restore volume on unmute |
| `CarAudioFocus.java` | `sendFocusLossLocked(loser, lossType)` → `sendFocusLossLocked(loser, lossType, winner, shouldFade, transientFadeManagerConfig)` using `dispatchAudioFocusChangeWithFade()`. `evaluateRequest()` switches to `int requestedUsage` instead of `@AudioContext int`. Add `isFadeManagerSupported()`, `getTransientFadeManagerConfig()`. Remove `removeDelayedAudioFocusRequestLocked()` (merged into `removeFocusEntryLocked`). `AudioManager` → `AudioManagerWrapper`. **Do not add** `TimingsTraceLog`. | "Audio Off mode" focus dispatch, assistant/call-ring talkback fix, block BT voice-call focus |
| `FocusInteraction.java` | Drop `CarAudioContext` from the constructor. `evaluateRequest(@AudioContext int, FocusEntry)` → `evaluateRequest(int requestedUsage, FocusEntry)`; visibility `public` → package-private. **Net A17:** `mContentObserverFactory.createObserver(callback)` with no explicit `Handler` (main looper). | Audio Off mode; prevent entertainment resuming during Audio Off |
| `CoreAudioVolumeGroup.java` | `AudioManager` → `AudioManagerWrapper`; device-based context map + `CarActivationVolumeConfig`. **Remove all `VersionUtils.isPlatformVersionAtLeastU()` guards.** `isAmGroupMuted()` → `mAudioManager.isVolumeGroupMuted(mAmId)`; `applyMuteLocked()` → `mAudioManager.adjustVolumeGroupVolume()`. Add `EMPTY_FLAGS = 0`. **Net A17:** `updateDevices(boolean useCoreAudioRouting)` that **returns early when core routing is on**. | Siri volume after E-call, restore on unmute, per-usage default volume (#222006) + `storeAndPersistGainIndexLocked` |
| `CarZonesAudioFocus.java` | `createCarZonesAudioFocus()`: `AudioManager` → `AudioManagerWrapper`, plus a `@Nullable CarAudioFeaturesInfo` parameter. `mCarAudioService` and `mAudioPolicy` become `@GuardedBy("mLock")`; `setOwningPolicy()` synchronized. Shared `ContentObserverFactory` created once outside the per-zone loop. `FocusInteraction` no longer receives `CarAudioContext`. | Audio Off mode — dispatch focus-gain to Entertainment |
| `CarAudioZoneConfig.java` | Add `mDefaultCarAudioFadeConfiguration`, `mAudioAttributesToCarAudioFadeConfiguration`, `mIsFadeManagerConfigurationEnabled`, `@GuardedBy("mLock") mIsSelected`. Add `isSelected()`, `setIsSelected()`, `isFadeManagerConfigurationEnabled()`, `isActive()`, `audioDevicesAdded/Removed()`, the two fade getters. **Net A17:** `updateVolumeDevices(boolean useCoreAudioRouting)` (the parameter was dropped in A16 and restored in A17 — go straight to the parameterised form). | Cyber-mute |

### 3c. Nissan-only — Google never touched them, but they break if ignored

| File | Action |
|---|---|
| `CoreAudioVolumeGroupHelper.java` (#222006) | Keep. **Re-wire** onto the rewritten `CoreAudioVolumeGroup` — its `getDefaultVolumeForAttributes(attrs, max, min)` call sits inside the constructor Google rewrote. |
| `CarAudioGainMonitor.java` | Holds MaxVolumeStartup. Reconcile with Google's activation volume (§5, decision 2). |
| `CarAudioGainCallbackHandler.java` | Re-test only — OEM gain callback timing changes with the main-looper move. |
| `CarAudioSettings.java` | Re-test only — guest-profile volume bar, CP-song max volume, Audio-Off persist. |
| `ZoneAudioPlaybackCallback.java` | Re-test only — `CarAudioPlaybackMonitor` now runs alongside it. |
| `vendor/alliance/.../oemcaraudioservices/` (Alliance OEM plugin) | Not in the AOSP diff, but `CarAudioService` calls it on every focus/ducking/volume decision. Its contract must keep working through the merge. Full regression required. |

---

## §4. Configuration and XML changes

| Item | A14 today | A17 requirement |
|---|---|---|
| `car_audio_configuration.xml` schema | **v3** (v2 + plugin on da2) | Stays **valid at v3**. Bump to **v4 only if** you adopt fade or activation volume. |
| v4 new elements | — | `<activationVolumeConfig>` / `<activationVolumeConfigEntry>` (min/max %, `invocationType` = `onBoot` \| `onSourceChanged` \| `onPlaybackChanged`); `<applyFadeConfigs>` / `<fadeConfig>`; `<deviceConfigurations>` block (`useCoreAudioVolume`, `useCoreAudioRouting`, `useHalDuckingSignals`, `useCarVolumeGroupMuting`); `<activationVolumeConfigs>` |
| `car_audio_fade_configuration.xml` | Does not exist | New file at `/vendor/etc/`, parsed by `CarAudioFadeConfigurationHelper`. Only needed if fade is enabled. |
| RRO flag `audioUseFadeManagerConfiguration` | Does not exist | New, default `false`. Leave false to decline system fade. |
| RRO flag `audioUseMinMaxActivationVolume` | Does not exist | New, default `false`. Leave false to keep Nissan MaxVolumeStartup. |
| Existing flags | `audioUseDynamicRouting`, `audioUseHalDuckingSignals`, `audioUseCarVolumeGroupMuting`, `audioPersistMasterMuteState`, `audioUseCarVolumeGroupEvent`, `useCoreAudioVolume`, `useCoreAudioRouting` | All unchanged through A17 |

---

## §5. Behavioral changes and the decisions they force

| # | Behavior | Net A17 state | Decision |
|---|---|---|---|
| B1 | **System fades apps out on focus loss** — a losing app is volume-shaped down via `dispatchAudioFocusChangeWithFade`, even if it ignores its focus-loss callback | On when `audioUseFadeManagerConfiguration` + fade XML present | **Decide whether to enable.** Independent of Nissan Audio Off (an entertainment mute) — they do different jobs and can coexist. The only overlap is textual: the fade code edits `CarAudioFocus`/`FocusInteraction`/`CarZonesAudioFocus`, which the fork also patched. That is a merge, not a behavior clash. |
| B2 | **Volume clamped into a safe range at activation** — gain forced between min and max % on boot / new source / new playback | On when `audioUseMinMaxActivationVolume` | **Pick one implementation.** This is the same goal as Nissan MaxVolumeStartup (`CarVolumeGroup` + `CarAudioGainMonitor`). Running both double-clamps the boot volume. |
| B3 | **Smarter routing fallback** — only the selected+active config is routed; default config always registered last | Always on | Take it. Little overlap with fork code; improves device connect/disconnect behavior. |
| B4 | **Audioserver-crash recovery + deferred init** — init defers if audioserver is down, reinitialises when it restarts | Always on | Take it, then **verify against the fork's boot order and Audio-Off-persist init**. |
| B5 | **Zone layout can come from the AudioControl HAL** (`getAudioZones()`) instead of XML | Runtime choice | **Recommendation: keep XML.** `supportsAudioZones()` = false → `CarAudioZonesHelperImpl` is selected. Cost of the HAL path is a v3→v5 AIDL upgrade plus a new vendor HAL implementation, for a stable single-zone topology. |
| B6 | **XML v4 elements** for fade + activation volume | v4 available | Bump **only if** B1 or B2 is adopted. |
| B7 | **`CoreAudioVolumeGroup.updateDevices()` skips work under core routing** | Early return when `useCoreAudioRouting` | Take it — lands directly in a forked file. |
| B8 | **Audio policies and framework callbacks on the main looper** — `Looper.getMainLooper()` for the policy builders, `getMainExecutor()` for the audioserver-state and volume-group callbacks | Main looper | **No work — the A14 baseline is already here.** Verified in-tree: `CarAudioService.java:1612` already sets `Looper.getMainLooper()` (AOSP, 2019), `:1692` already calls `init(mContext.getMainExecutor())` (AOSP, 2022), `ContentObserverFactory.java:37` already binds the main looper. AOSP moved these *onto* a `HandlerThread` in A15/A16 and back off in A17 — a round trip. **`mHandlerThread` is not deleted**; it survives in A17 for internal posted work (mirror enable/disable, zone-config switch). The only genuinely new main-looper code is the A15 `CarAudioServerStateCallback`. |
| B9 | **Added-then-removed intermediates** — persist-fade-balance, `CarVolumeGroup` event logger, `TimingsTraceLog` in focus | All gone by A17 | **Do not port them.** |
| B10 | **Config-switch device-affinity fix** — `changeAudioPolicyForConfigChangeLocked()` replaced by an explicit 3-step sequence | Always on | Take it — fixes a latent bug where user device affinities were not updated on config switch under core audio routing. |

**Five open decisions to close before merging** (a sixth, listed below, is closed with no action):

1. Enable Google's system fade (B1)? — yes/no
2. MaxVolumeStartup **or** activation volume (B2)? — pick exactly one
3. Zone source: XML **or** HAL (B5)? — recommendation: XML
4. Bump `car_audio_configuration.xml` to v4 (B6)? — follows from 1 and 2
5. ~~Re-validate OEM gain callback + master-mute timing after the main-thread move (B8)~~ — **CLOSED, no action.** Those paths already run on the main looper at A14, so the migration does not change their timing. (The standing fact that focus callbacks execute on the watchdog-monitored CarService main thread — including the synchronous call out to the Alliance OEM plugin — is pre-existing behavior, not something A17 introduces.)
6. Confirm the "skip intermediates" rule (B9) with anyone reading the per-version docs

---

## §6. HAL-layer conclusion

| Item | Verdict |
|---|---|
| `IAudioControl` AIDL version | **No bump required.** The interface is identical across A14/15/16/17 at **AIDL v3**. Every architectural change in this span lands elsewhere: fade in Java `CarAudioService`, CAP in `android.hardware.audio.core`, background hardening in `audioserver`. |
| AudioControl HAL version hygiene | **Fix now, regardless of A17.** The tree is inconsistent: device manifests say **v1**, the VINTF fragment says **v3**, the build says **V2-ndk**. Settle on **v3** — it is required for the existing `IModuleChangeCallback` path and is cheap. |
| Zone-config-on-HAL (`getAudioZones()`) | **Defer.** `getAudioZones` / `AudioZone` / `getAudioDeviceConfiguration` **do not exist anywhere in the tree** — in-tree `IAudioControl.aidl` (v3 and `current`) declares only the A14 method set. Adopting it needs an AIDL v3→v5 interface import **plus** a new Alliance vendor HAL implementation. Take the CarService-side interface refactor (needed anyway), keep `supportsAudioZones()` false. |
| Core Audio HAL | Still **HIDL** (6.0 on da2, 7.0 on full). No AIDL `android.hardware.audio.core` in tree. This gates §10's CAP work. |
| HIDL AudioControl 1.0/2.0 | Deprecated since A12L; the fork is already off them. No action. |

---

## §7. Nissan collision map — where the effort actually is

Only files that **both** sides changed are dangerous.

| File | Google changed | Nissan forked | Risk | The conflict, in short |
|---|---|---|---|---|
| `CarAudioService.java` | Heavily, all 3 versions | Heavily | 🔴 **HIGH** | Google's 4-way policy split + async/HAL init vs. the fork's many hooks |
| `CarVolumeGroup.java` | A15 mapping rewrite, A16/A17 | Heavily | 🔴 **HIGH** | Address→device rewrite and the mute split land exactly on MaxVolumeStartup / mute patches |
| `CoreAudioVolumeGroup.java` | A15 rewrite, A17 device update | Yes | 🔴 **HIGH** | Google's ctor + mute rewrite vs. Siri/E-call fix and #222006 |
| `CarAudioFocus.java` | A15 fade, A16/A17 tracing | Yes | 🔴 **HIGH** | Textual merge — fade dispatch vs. Audio Off dispatch. No behavior clash. |
| `FocusInteraction.java` | A15 signature change, A16/A17 | Yes | 🔴 **HIGH** | Textual merge — signature changes vs. Audio Off. No behavior clash. |
| `CarZonesAudioFocus.java` | A15 wrapper | Yes | 🟠 **MED** | `AudioManagerWrapper` + `@GuardedBy` tightening vs. Audio Off focus-gain dispatch |
| `CarAudioZoneConfig.java` | A15/A16/A17 | Yes | 🟠 **MED** | Fade + selected-state fields vs. cyber-mute |
| `CoreAudioVolumeGroupHelper.java` | No (Nissan-only) | Yes (new) | 🟡 LOW | Re-point at the rewritten `CoreAudioVolumeGroup` |
| `CarAudioGainMonitor.java` | No | Yes | 🟡 LOW | Reconcile MaxVolumeStartup intent with B2 |
| `CarAudioGainCallbackHandler.java` | No | Yes | 🟡 LOW | Re-test after the main-thread move |
| `CarAudioSettings.java` | No | Yes | 🟡 LOW | Re-test |
| `ZoneAudioPlaybackCallback.java` | No | Yes | 🟡 LOW | Re-test alongside `CarAudioPlaybackMonitor` |

**Why the volume classes are the worst of it:** Google rewrote the *exact two surfaces* the fork patched — the mute path (now `applyMuteLocked()` + `saveMuteStateToSettingsLocked()`) and the sound→speaker mapping (address → device). The MaxVolumeStartup, master-mute-skip, ducking-skip, restore-on-unmute, #222006 default-volume and Siri/E-call patches all sit in those methods. They must be **re-applied on top of the new structure**, not pasted back.

---

## §8. Order of work

1. **Add the A15 foundation.** `AudioManagerWrapper` first, then the other 7 A15 classes, then the §3a stock-A14 modifications. Large but low-risk — nothing else compiles without it.
2. **Merge the HIGH/MED forked files one at a time** (§3b), each followed by a targeted test of the Nissan feature living in it: Audio Off, MaxVolumeStartup, Siri/E-call volume, master mute, ducking, cyber-mute.
3. **Take the A16 zone refactor** — `CarAudioZonesHelper` as interface + `CarAudioZonesHelperImpl` — but keep reading XML. Add the HAL-zone classes inert.
4. **Apply the A17 cleanups** — main looper, the core-routing early return, the explicit 3-step config-switch sequence, and the parameterised `updateDevices`/`updateVolumeDevices`.
5. **Close decisions 1 and 2 (§5) with the audio team** before finalising the focus and volume merges.
6. **Fix the AudioControl HAL v1/v3/V2 manifest inconsistency** — independent, cheap, do it any time.
7. **Run §10 items as separate projects.**

**Verification**
- Boot each variant (`aivi2_n_full`, `aivi2_n_da2`, `aasp_n` emulator). Check `dumpsys car_service` (audio) and `dumpsys audio` for zones/groups/policies; watch `CarAudioService` / `AudioPolicy` logcat for config-parse errors.
- Confirm zones came from `CarAudioZonesHelperImpl` (XML), not the HAL — proof that B5 stayed off.
- `adb shell lshal | grep -i audiocontrol` → one consistent AIDL version.
- Regression-test the Nissan flows: Audio Off mode, MaxVolumeStartup, master mute, ducking, Siri/E-call volume, guest-profile volume bar, OEM gain callback, cyber-mute, mirroring.
- Standard checks: volume keys, focus arbitration, ADAS ducking, ANC.

---

## §9. Do-not-replay list (the oscillations)

These changed direction across the three releases. Jumping A14→A17 means implementing **only the net A17 form**.

> **Verified against the tree:** the A14 baseline column below was read from the actual fork, not inferred.
> **Six of the nine rows require no work at all** — the A14 form already *is* the A17 form, and AOSP simply
> made a round trip in between. Only the three `updateDevices` / `updateVolumeDevices` rows are real work,
> and those arrive as part of the A15 device-mapping rewrite (§3) rather than as standalone changes.

| API / field | **A14 baseline (your tree)** | A15 | A16 | **Net A17 — implement this** | Work |
|---|---|---|---|---|---|
| `ContentObserverFactory.createObserver` | no `Handler` (main looper) | no `Handler` | explicit `Handler` | **no `Handler`** (main looper internally) | **none** |
| `CoreAudioVolumeGroupCallback.init()` | takes `Executor` | takes `Executor` | no-param (executor in ctor) | **takes `Executor`** | **none** |
| `AudioPolicy` looper | `Looper.getMainLooper()` | handler thread | main looper | **main looper** | **none** |
| `TimingsTraceLog` in `CarAudioFocus` / `FocusInteraction` | absent | absent | added | **absent** | **none** |
| `mPersistFadeBalanceLevels` / `AUDIO_FEATURE_PERSIST_FADE_BALANCE_VALUES` | absent | absent | added | **absent** | **none** |
| `mEventLogger` (LocalLog) in `CarVolumeGroup` | absent | absent | added | **absent** | **none** |
| `CarVolumeGroup.updateDevices()` | absent | `(boolean)` | no-param | **`(boolean)`** | new |
| `CoreAudioVolumeGroup.updateDevices()` | absent | `(boolean)` | no-param | **`(boolean useCoreAudioRouting)` with early-return guard** | new |
| `CarAudioZoneConfig.updateVolumeDevices()` | absent | — | no-param | **`(boolean useCoreAudioRouting)`** | new |

**Why this matters:** read version-by-version, these nine look like nine things to get right. Against the
A14 baseline, six are already correct and must simply be *left alone* — the risk is a well-meaning engineer
following the A15→A16 docs and actively introducing the `HandlerThread` / `mEventLogger` / `TimingsTraceLog`
forms that A17 then removes.

---

## §10. Separate projects — outside `car/audio`, release-note sourced, UNVERIFIED locally

These are real A16/A17 platform changes, but none of them appear in the three verified `car/audio` patches. **Confirm each against real `frameworks/av` + `hardware/interfaces/audio` diffs before planning any effort.**

| Item | Version | Impact on the fork | Recommendation |
|---|---|---|---|
| **CAP engine config moves to the core Audio HAL** — `getEngineConfig()` returning `AudioHalEngineConfig.CapSpecificConfig` (`AudioHalCapConfiguration`, `AudioHalCapCriterionV2`, `AudioHalCapDomain`, `AudioHalCapParameter`, `AudioHalCapRule`) replaces `audio_policy_engine_configuration.xml` + `.pfw` | A16 | The fork uses `engine_library="configurable"` with static XML + parameter-framework. **Hard-gated behind a full core-audio HIDL→AIDL migration** — the single largest piece of work in the whole effort. | **Keep XML.** `EngineConfigXmlConverter.h` converts existing vendor XML→AIDL, so the config survives even if the core HAL goes AIDL. Do not hand-author AIDL `CapSpecificConfig`. |
| **`STRATEGY_` prefix rename** — all product strategy names require it (`media` → `STRATEGY_MEDIA`, etc.). OEM slots `vx_1000`–`vx_1039` unchanged. | A16 | The fork has **21 lowercase strategies** (`music`, `nav_guidance`, `voice_call`, `oem_ipa`, …) referenced from `.pfw`, volumes XML, and `/Policy/.../product_strategies/<name>` paths. | Only required **if** the CAP path goes AIDL-native. Plan it as an isolated, mechanical, wide, well-tested rename. |
| **Background Audio Hardening** — `AudioTrack.write()`, `AAudioStream_write()`, `requestAudioFocus()`, `setStreamVolume()`, `adjustVolume()`, `setStreamMute()`, `setRingerMode()` fail silently without a visible Activity or a non-`SHORT_SERVICE` FGS; apps targeting API 37 additionally need while-in-use FGS capability (`mediaPlayback`). Waived only with `SCHEDULE_EXACT_ALARM` + `USAGE_ALARM`-only. | A17 | **App-side, not `car/audio`.** Hits background music with display off, boot-triggered audio via `BOOT_COMPLETE`, backgrounded navigation audio, background VoIP, alarm/reminder audio. | Audit every Nissan/Alliance app that plays audio without a visible Activity. Migrate to `MediaSessionService` (media3) or a manual `mediaPlayback` FGS. Debug: `adb shell cmd audio set-enable-hardening enable\|throw\|disable`, `adb logcat \| grep AudioHardening`. |
| **Dedicated assistant volume stream** — `AudioManager.MODE_ASSISTANT_CONVERSATION`; `USAGE_ASSISTANT` isolated from `STREAM_MUSIC` | A17 | Map the new stream to a volume group in `car_audio_configuration.xml` and expose an independent slider in the Audio Settings UI. Interacts with the fork's Siri/E-call volume handling. | Scope once verified. |
| **`AudioDeviceInfo.TYPE_BLE_HEARING_AID`** | A17 | New device type distinct from `TYPE_BLE_HEADSET`; routing handled by the system. | No app/service work expected. |
| **xHE-AAC software encoder** (`c2.android.xheaac.encoder`) | A17 | Codec availability with mandatory loudness metadata. | Informational. |
| **New `@SystemApi`s** `getFadeTowardFront()` / `getBalanceTowardRight()` (AAOS 25Q4) | A16 | Read-side parity with the A9 setters; values persist per user across ignition cycles. | Optional adoption. |
| **Alternative app audio controls** (AAOS 25Q4) | A16 | Independent volume + routing path for communication apps while driving. | Optional; evaluate against the fork's existing call-volume handling. |
| **AAudio OEM-defined attribute tags** | A16 | Integrates with `<oemContext>` in XML v3+. Under CAP, the OEM context must match a CAP product strategy. | Only relevant if CAP goes AIDL-native. |
| **Deprecated `CarAudioManager` APIs** — `isDynamicRoutingEnabled()`, `getExternalSources()`, `createAudioPatch()`, `releaseAudioPatch()` | Deprecated in **A14** | Already deprecated at the baseline; still present through A17. | Audit for callers; no forced removal. |

---

## §11. ⚠️ Unresolved contradiction — settle before starting

`docs/migrate_to_a17.md` contains **third-party notes** (explicitly "from other people") claiming an A15→A16 change set that the verified patches **do not show**:

| Third-party claim | What the verified diffs say |
|---|---|
| The `audio/hal/` subfolder is flattened; `AudioControlWrapper`, `HalAudioDeviceInfo`, `HalAudioFocus`, `HalAudioGainCallback`, `HalAudioModuleChangeCallback`, `HalFocusListener` move up to `audio/` | `hal/` is **still present and modified in place** in A15, A16 **and** A17 |
| New files `CarAudioEffects.java`, `CarAudioFocusEnforcement.java`, `CarAudioParkedStateMonitor.java` | **None appear** in any of the three patches |
| "Relaxed Park Mode" focus enforcement — `Flags.audioFocusEnforcement()`, `R.bool.audioEnableAudioFocusEnforcement`, `audioFocusEnforcementUsages`, `audioFocusEnforcementDoNotSilenceAttributes`, `R.bool.audioFocusEnforcementRelaxedWhileParked` | **No trace** in any patch |

**Most likely explanation:** those notes were taken against a **newer AOSP `main` snapshot** than the `android-16.0.0_r1` tag used to generate these patches — i.e. they are post-A16 `main` work that may or may not land in the A17 release.

**Action:** before committing to a plan, re-run the A16→A17 diff against current `main` and check specifically for the `hal/` flattening and the three focus-enforcement classes. If they are real, the migration gains a **package-move refactor plus a new focus-enforcement subsystem** — neither of which is costed anywhere in this document.

```bash
cd /home/worker/a14full/android/packages/services/Car
git remote add aosp https://android.googlesource.com/platform/packages/services/Car
git fetch aosp --tags
git diff --name-status aosp/android-16.0.0_r1..aosp/main -- service/src/com/android/car/audio/
git log --oneline aosp/android-16.0.0_r1..aosp/main -- service/src/com/android/car/audio/
```

---

## §12. Verification record

Every claim below was re-checked directly against `/a14full` on **2026-07-31**, not carried over from the
source documents. Line numbers are as of branch `nissan_u_ccs2_release`.

| Claim | Evidence | Result |
|---|---|---|
| None of the 13 new files exist | directory listing of `car/audio/` | ✅ confirmed |
| `CarAudioZonesHelper` is a class, not an interface | `CarAudioZonesHelper.java:61` — `/* package */ final class` | ✅ confirmed |
| XML schema v1–v3 only | `CarAudioZonesHelper.java:98–107` — `SUPPORTED_VERSION_1/2/3` | ✅ confirmed |
| Single `AudioPolicy` | `CarAudioService.java:331` — `private AudioPolicy mAudioPolicy` | ✅ confirmed |
| `hal/` subfolder still present | listing — 10 files under `car/audio/hal/` | ✅ confirmed |
| Address-based volume mapping | `CarVolumeGroup.java:80–81` — `SparseArray<String>` + `ArrayMap<String, CarAudioDeviceInfo>` | ✅ confirmed |
| `CarAudioDeviceInfo` old ctor | `CarAudioDeviceInfo.java:71` — `(AudioManager, AudioDeviceInfo)` | ✅ confirmed |
| `sendFocusLossLocked` 2-arg, no fade | `CarAudioFocus.java:187`; raw `AudioManager` at `:72` | ✅ confirmed |
| `FocusInteraction` takes `CarAudioContext`; `evaluateRequest` is `public` + `@AudioContext` | `FocusInteraction.java:343–345`, `:424` | ✅ confirmed |
| `CoreAudioVolumeGroup` has platform guards to remove | 4 × `isPlatformVersionAtLeastU` | ✅ confirmed |
| `updateDevices` / `updateVolumeDevices` absent | tree-wide grep — no hits | ✅ confirmed |
| Forked-file list (§7) | `[Audio]`-tagged commits per file: all 12 listed files have them; all files listed as stock have none | ✅ confirmed |
| XML config versions | `aivi2_n_full` v3 · `aivi2_n_da2` v2 · `aasp_n` emulator v3 | ✅ confirmed |
| Fade / activation-volume flags absent | tree-wide grep — no hits | ✅ confirmed |
| AudioControl AIDL tops out at v3 | `aidl_api/…/` contains `1 2 3 current` | ✅ confirmed |
| `getAudioZones()` absent tree-wide | grep across `hardware/interfaces/automotive/audiocontrol/` — no hits | ✅ confirmed |
| AudioControl version three-way mismatch | manifests `<version>1</version>` (`aivi2_n_full/manifest.xml:239`, `aivi2_n_da2/manifest.xml:339`) · VINTF fragment `<version>3</version>` (`vendor/alliance/…/audiocontrol/default/audiocontrol.xml`) · build links `audiocontrol-V2-ndk` (`Android.bp:26`) | ✅ confirmed — all three differ |
| ~~"Main-thread move" is a migration change~~ | `CarAudioService.java:1612` (`Looper.getMainLooper()`, AOSP 2019) · `:1692` (`getMainExecutor()`, AOSP 2022) · `ContentObserverFactory.java:37` — all already at the A17 form | ❌ **corrected** — nets to zero |
| ~~"The dedicated `HandlerThread` is gone"~~ | `mHandlerThread`/`mHandler` still used at `CarAudioService.java:1061, 1077, 1096, 1117, 2562` for mirroring and zone-config switch | ❌ **corrected** — not removed |

**Net effect of the two corrections:** decision 5 is closed with no action, and six of the nine §9
oscillation rows are reclassified from "implement the A17 form" to "already correct — leave alone".

---

## Sources

**Verified (real git patches vs. the actual fork):**
`raw/AOSP_Car_Audio_File_Diff.md` · `raw/AOSP_Car_Audio_Changelog.md` · `docs/a14_current_vs_a17_changes.md` · `docs/final_migration_summary_a14_to_a17.md` · `docs/config_on_hal_deep_dive.md` · direct inspection of `/a14full/.../car/audio/`

**Release-note sourced (§10, unverified locally):**
`master/Master_Audio_Changelog_A14_to_A17.md` · `master/Architectural_Master_Audio_Changelog_A14_to_A17.md` · `changelogs/*/`
[AAOS 24Q3](https://source.android.com/docs/automotive/start/releases/aaos-24q3) ·
[AAOS 25Q2](https://source.android.com/docs/automotive/start/releases/aaos-25q2) ·
[AAOS 25Q4](https://source.android.com/docs/automotive/start/releases/aaos-25q4) ·
[Audio Control HAL](https://source.android.com/docs/automotive/audio/audio-control-hal) ·
[Configurable Audio Policy AIDL HAL](https://source.android.com/docs/core/audio/aidl-cap) ·
[Car Audio Configuration](https://source.android.com/docs/automotive/audio/audio-policy-configuration) ·
[Audio Configuration Flags](https://source.android.com/docs/automotive/audio/config-flags) ·
[Background Audio Hardening — Android 17](https://developer.android.com/about/versions/17/changes/bg-audio)

**Third-party notes, contradicted (§11):** `docs/migrate_to_a17.md`
