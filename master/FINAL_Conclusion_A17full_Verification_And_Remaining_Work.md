# A17full VERIFICATION — What the Old Plan Got Wrong, and What Actually Remains

## Status vs `FINAL_Conclusion_What_To_Change_A14_to_A17.md`

> That document was built by comparing the **A14 Nissan fork** against **Google AOSP patch notes/commits**
> (a snapshot of AOSP `main` at some point pre-A17-release). It was never checked against a real A17 tree,
> because none was available yet.
>
> We now have `/home/worker/a17full` — a real, buildable A17 checkout on branch
> `nissan_c_join_aivi2_fc` (`packages/services/Car` at commit `36afc909f0`). This document re-verifies
> every material claim in the old plan against that tree and against the in-tree AOSP HAL sources
> (`hardware/interfaces/automotive/audiocontrol`), and restates the conclusion.
>
> **Headline result: the old plan's diagnosis of the *problem* was right, but its picture of the *AOSP
> end-state* was measurably stale, and — this is the important part — the practical scope of remaining
> work is bigger and simpler to state than "resolve merge conflicts."**

---

## §0. The one fact that changes everything

**`packages/services/Car/service/src/com/android/car/audio/` in `a17full` contains zero Nissan
behavioral code.** Every non-merge commit author touching that directory on this branch is
`@google.com` / `@ford.com` / historical `@renault.com`/`@ampere.cars` (dead branch, not an ancestor of
this HEAD) — **no Nissan, no LGE author appears anywhere in 661 commits.** Grepping the entire directory
for `MaxVolumeStartup`, `Audio Off`, `cyber-mute`, `Siri`, `e-call`, `Nissan`, `Renault`, `Mitsubishi`
returns **zero hits**. `config.xml` defaults (`audioUseDynamicRouting=false`, etc.) are stock AOSP
defaults, not Nissan production values.

This is **not** a merge-conflict situation. It is a **clean-room AOSP A17 car-audio service** with Nissan
build fixes on top (commits literally titled `[Build] ninja build error fix by AI-copilot for A17`) and
**nothing else**. The Nissan device trees (`device/nissan/.../overlay/CarServiceOverlayNissanFull/...`)
already correctly flip `audioUseDynamicRouting=true`, `audioUseCarVolumeGroupMuting=true` for A17 — so the
**RRO/config layer is ahead of the Java source layer**. Shipping today would silently run stock AOSP audio
behavior with Nissan's config flags pointed at code that was never written.

So: the old plan's §7 "Nissan collision map" (a table of merge risk) is the wrong mental model. There is
nothing to *merge* — there is a full feature set to **re-implement from scratch** against the new AOSP
class shapes, using the A14 fork purely as the **functional spec**, not as a patch source.

---

## §1. AOSP structural migration — confirmed already complete in `a17full`

These parts of the old plan's §1–§3 were right, and required no further work:

| Claim | Verified in `a17full` |
|---|---|
| 13 new files exist (`AudioManagerWrapper`, `CarActivationVolumeConfig`, `CarAudioDeviceCallback`, `CarAudioFadeConfigurationHelper`, `CarAudioParserUtils`, `CarAudioPlaybackMonitor`, `CarAudioProtoUtils`, `CarAudioServerStateCallback`, `AudioControlZoneConverter`, `AudioControlZoneConverterUtils`, `CarAudioZonesHelperAudioControlHAL`, `CarAudioZonesHelperImpl`, `TEST_MAPPING`) | ✅ all present |
| `CarAudioZonesHelper` is now an 8-method interface | ✅ `CarAudioZonesHelper.java:30` |
| `CarAudioService` split into 4 `AudioPolicy` fields | ✅ `mVolumeControlAudioPolicy`/`mFocusControlAudioPolicy`/`mRoutingAudioPolicy`/`mFadeManagerConfigAudioPolicy` at `CarAudioService.java:357-363` |
| `CarVolumeGroup` moved to device-based mapping | ✅ `SparseArray<CarAudioDeviceInfo> mContextToDevices` at `CarVolumeGroup.java:107` |
| `CarAudioDeviceInfo` ctor takes `(AudioManagerWrapper, AudioDeviceAttributes)` | ✅ `CarAudioDeviceInfo.java:97` |
| `CarAudioContext` drops `mContextsToDuck`, adds `getLegacyContextForUsage` | ✅ confirmed (no hits for old API; new API at `CarAudioContext.java:456`) |
| `CarAudioDynamicRouting` routes selected+active configs, default last | ✅ `CarAudioDynamicRouting.java:55-90` |
| `CoreAudioVolumeGroupCallback` uses `Executor` | ✅ `CoreAudioVolumeGroupCallback.java:37-45` |
| RRO flags `audioUseFadeManagerConfiguration`, `audioUseMinMaxActivationVolume` exist, default `false` | ✅ `config.xml:102,109` |
| `car_audio_configuration.xml` still v3, no v4 elements | ✅ `device/nissan/aivi2_n_full/audio_policy_configurable/car_audio_configuration.xml:2` |

**Conclusion: do not spend any effort "applying the AOSP A15/A16/A17 diff." It is already in the tree.**
The work is entirely in re-adding Nissan behavior on top of it.

---

## §2. Corrections to the old plan — verified wrong against the real tree

The old plan's §9 ("do-not-replay list") and §12 ("verification record") were checked against patches from
an AOSP snapshot that turned out to **not** match what actually shipped in this A17 tree. Six specific
claims are reversed:

| Claim in old plan | Old plan said | **Actual `a17full` state** | Evidence |
|---|---|---|---|
| `mEventLogger` (LocalLog) in `CarVolumeGroup` | "absent in net A17 — do not add" | **Present.** `protected final LocalLog mEventLogger` | `CarVolumeGroup.java:178,599,674` |
| `TimingsTraceLog` in `CarAudioFocus` | "absent in net A17 — do not add" | **Present**, used in focus-loss dispatch | `CarAudioFocus.java:43,363` |
| `mPersistFadeBalanceLevels` / `audioPersistFadeBalanceLevels` | "absent in net A17 — do not add" | **Present**, gated by `Flags.audioFadeBalanceGetterApis()` + RRO flag, used for `CarAudioEffects` fade/balance persistence | `CarAudioService.java:232,498-499,692,932,998,1174`; RRO flag at `config.xml:129` |
| `ContentObserverFactory.createObserver()` | "no-`Handler` form, main-looper internal — already correct, don't touch" | **Takes an explicit `Handler` parameter** | `ContentObserverFactory.java:33` |
| `CoreAudioVolumeGroup.updateDevices()` | "net A17 = `(boolean useCoreAudioRouting)` with early-return guard" | **No-arg**: `void updateDevices()` | `CoreAudioVolumeGroup.java:315` |
| `CarAudioZoneConfig.updateVolumeDevices()` | "net A17 = `(boolean useCoreAudioRouting)`" | **No-arg**: `void updateVolumeDevices()` | `CarAudioZoneConfig.java:579` |
| Audio-policy looper (B8) | "already on `Looper.getMainLooper()` at A14 baseline — **no work**" | **Runs on `mHandlerThread.getLooper()`**, a dedicated `HandlerThread`, for all 4 policy builders | `CarAudioService.java:2278,2308,2365,2403` (thread created at `:523-524`) |

**On the looper point specifically:** the old plan's "A14 baseline" evidence was read from the **Nissan
fork's own patched `CarAudioService.java`** (`nissan_u_ccs2_release`), not from stock AOSP A14. The main-looper
behavior was very likely a **Nissan customization**, not an AOSP default that the fork happened to already
have. Since `a17full` has no Nissan code, it shows AOSP's real default (`HandlerThread`) — which the old
plan mis-attributed as "already-there, no work." **If Nissan wants policies back on the main looper, that
is now a real, undone piece of work**, not a closed decision.

§11's "unresolved contradiction" is **resolved, and the third-party notes were right, not the "verified"
column**:

| Item | Old plan's verdict | Actual `a17full` state |
|---|---|---|
| `hal/` subfolder flattened into `audio/` | "still present in A15/16/17 — contradiction unconfirmed" | **Flattened.** No `hal/` dir exists; confirmed by commit `c10de5e22f Move com.android.car.audio.hal to com.android.car.audio` |
| `CarAudioEffects.java`, `CarAudioFocusEnforcement.java`, `CarAudioParkedStateMonitor.java` | "none appear in any patch" | **All three exist** in `car/audio/` today (Relaxed Park Mode / focus enforcement subsystem is real, shipped AOSP code) |

---

## §3. HAL layer — materially different from what the old plan concluded

§6 of the old plan said "No `IAudioControl` bump required — tops out at AIDL v3, `getAudioZones()` doesn't
exist anywhere." Both are now false:

- **`IAudioControl` is at AIDL v6** in `hardware/interfaces/automotive/audiocontrol/aidl/` (`aidl_api/.../1` through `/6`, plus `current`).
- The v6 interface adds `getAudioDeviceConfiguration()`, `getOutputMirroringDevices()`, and
  **`getCarAudioZones()`** (not `getAudioZones()` — the method name in the old plan was itself approximate)
  — see `IAudioControl.aidl:227,235,248`.
- `AudioControlWrapper.java` is a single AIDL-only class that runtime-negotiates via
  `mAudioControl.getInterfaceVersion()` (`AudioControlWrapper.java:161-449`) — there is no
  `AudioControlWrapperV1`/`V2`/`Aidl` file split as the old plan's §1/§3a assumed. The old
  `hal/AudioControlWrapper*.java` per-version stub files described in the plan **do not exist**; they were
  consolidated into one class that checks `getInterfaceVersion()` before calling anything version-gated.
- Zone-on-HAL loading is real and wired: `CarAudioService.loadAudioZonesUsingAudioControlLocked()`
  (`CarAudioService.java:2172-2200`) gates on **both** an AConfig flag
  (`Flags.audioControlHalConfiguration()`) **and** a HAL capability check
  (`audioControlWrapper.supportsFeature(AUDIOCONTROL_FEATURE_AUDIO_CONFIGURATION)`, feature id `6`,
  `AudioControlWrapper.java:79-175`) before it ever tries `CarAudioZonesHelperAudioControlHAL`. This is a
  different (and more defensive) gating mechanism than the old plan's simple `supportsAudioZones()`
  boolean, but it has the **same practical effect**: with an Alliance vendor HAL that doesn't advertise the
  feature, it silently falls through to XML (`CarAudioZonesHelperImpl`). **Recommendation to keep XML still
  stands, and requires no code change to enforce** — it's the default fallback path already.
- **Vendor/AOSP interface version mismatch is real and larger than the old plan's §12 finding.** AOSP
  interface headers are at v6; `vendor/alliance/hardware/interfaces/automotive/audiocontrol/Android.bp`
  builds the default implementation against `android.hardware.automotive.audiocontrol-V2-ndk`
  (`default/Android.bp:24`) while a separate target in the same package references `-V6`
  (`Android.bp:13`) and another references `-V2` (`Android.bp:34`) — i.e. **the same mismatch pattern the
  old plan flagged (§6/§12) still exists, just wider (v2 vs v6, not v1/v2/v3)**. Functionally this is safe
  today because every new v3–v6 method is called behind `getInterfaceVersion()`/`supportsFeature()` checks,
  but it should still be cleaned up for hygiene and to avoid a silent feature gap if someone later turns on
  `Flags.audioControlHalConfiguration()`.

---

## §4. OEM plugin integration — good news, no action needed

The old plan's §3c worried about "Full regression required" for the Alliance OEM plugin. Verified: the
plugin hooks in through the **standard AOSP `CarOemProxyService`** mechanism
(`CarAudioService.java:149,1194-1199` — `proxy.getCarOemAudioFocusService()` /
`getCarOemAudioVolumeService()` / `getCarOemAudioDuckingService()`), which is generic AOSP code, not a
Nissan patch. It is present and intact in `a17full` as shipped. The Alliance plugin app itself
(`vendor/alliance/services/car/plugins/audio/`, separate git history, tag `[Audio] AllianceCaraudioService
1.0.0 Release`) already has A17 build fixes applied and its module structure (mute management, VPA status)
is untouched. **No wiring gap here — just the standard regression test pass the old plan already called
for.**

---

## §5. Corrected final task list for `a17full`

1. **Re-implement, not merge, every Nissan car-audio behavior**, using the A14 fork
   (`/home/worker/a14full/.../car/audio/`) purely as functional reference. Concretely, against the current
   `a17full` class shapes:
   - Audio Off mode (focus dispatch + entertainment mute) — new logic in `CarAudioFocus`,
     `FocusInteraction`, `CarZonesAudioFocus`.
   - MaxVolumeStartup — new logic layered on `CarVolumeGroup`/`CarAudioGainMonitor` (today's
     `CarAudioGainMonitor.java` is stock AOSP 2022 code, confirmed by its header and generic
     `// TODO (b/224885748)` comment — it holds no Nissan logic to "reconcile" with).
   - Master-mute persistence + OEM gain-callback wiring, cyber-mute, Siri/E-call volume, occupant-zone
     filter, mirror-handling patches, guest-profile volume bar, CP-song max volume — all absent, all need
     fresh implementation against `applyMuteLocked()`/`saveMuteStateToSettingsLocked()` in `CarVolumeGroup`
     and the new `CarAudioZoneConfig`/`CarAudioZone` shapes.
   - `CoreAudioVolumeGroupHelper.java` (#222006) — **does not exist in `a17full` at all** (not "needs
     re-wiring," needs to be **created new**) against the current `CoreAudioVolumeGroup.java`.
2. **Decide the main-looper question fresh** — it is not a closed no-op. `a17full` currently builds all 4
   audio policies on `mHandlerThread.getLooper()`. If Nissan requires main-looper behavior for the
   OEM-plugin-timing reasons the old plan discussed, that must be implemented now.
3. **Keep the two new AOSP features off** (`audioUseFadeManagerConfiguration`,
   `audioUseMinMaxActivationVolume` both default `false`) unless product wants them — this part of the old
   plan's recommendation still holds and requires no change.
4. **Keep zone config on XML, not HAL** — also still correct, and now confirmed to be the automatic
   fallback behavior of `loadAudioZonesUsingAudioControlLocked()` with the current vendor HAL. No code
   change needed to enforce this; just don't flip `Flags.audioControlHalConfiguration()` and don't add
   `AUDIOCONTROL_FEATURE_AUDIO_CONFIGURATION` support to the vendor HAL.
5. **Fix the AudioControl HAL version hygiene**, same recommendation as before but revised numbers:
   consolidate on a single AIDL version across `vendor/alliance/hardware/interfaces/automotive/audiocontrol/Android.bp`
   (currently references V2, V2-ndk, and V6 across different build targets in the same package) and the
   VINTF manifest fragments. Cheap, independent, do any time.
6. **Regression-test the RRO layer against real code**: `device/nissan/aivi2_n_full/overlay/CarServiceOverlayNissanFull/res/values/config.xml`
   already flips `audioUseDynamicRouting`/`audioUseCarVolumeGroupMuting` to `true` for A17 — verify this
   doesn't currently enable stock-AOSP-only behavior (e.g. plain group muting with no cyber-mute logic) in
   a way that ships silently wrong before the item-1 reimplementation lands.
7. **Everything in the old plan's §10** (background-audio hardening, assistant volume stream, CAP-on-HAL,
   `STRATEGY_` rename) is still unverified against `a17full`'s `frameworks/base`/`frameworks/av` and remains
   a separate-project scope; not rechecked in this pass.

---

## §6. Why the discrepancy happened (for anyone reading both docs)

The old plan's own preamble is explicit that its AOSP-side claims came from **git patches**, not a
released tree — and its §11 already flagged that a *different* set of third-party notes (`docs/migrate_to_a17.md`)
disagreed with those patches on exactly the items that turned out to be real (`hal/` flattening, the three
focus-enforcement classes). The most likely explanation, confirmed now: **the patches were cut from an AOSP
`main` snapshot that predates the actual A17 release**, and upstream kept moving (AIDL v3→v6,
`mEventLogger`/`TimingsTraceLog`/`mPersistFadeBalanceLevels` retained rather than removed, looper choice
never actually changed at the AOSP level) between that snapshot and what shipped. Nothing here reflects
badly on the earlier analysis method — cross-checking against a real tree, once available, is exactly the
right next step, which is what this document does.
