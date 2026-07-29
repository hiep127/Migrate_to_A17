# Final Migration Summary — Car Audio, Android 14 → 17 (Nissan `/a14full`)

## What this document answers

Google rewrote big parts of the car-audio code between Android 14 and 17. Our tree is still at the
Android 14 baseline, with Nissan fixes layered on top. This document says, concretely:

- **which files** we have to change,
- **which behaviors** change (and which of ours they collide with), and
- **where the merge is dangerous** — the files Google rewrote that we've *also* already modified.

**How to read it:** Section 1 is the behavior changes. Sections 2–3 identify the collision hotspots.
Section 4 is the full file list, grouped by how hard each group is. Section 4b zooms into the volume
classes. Sections 5–7 are the decisions, the unverified extras, and the suggested order of work.

**How reliable is this:** the car-audio parts are taken from the **real Google source diffs**
(`raw/AOSP_Car_Audio_File_Diff.md`) compared against our **actual code and commit history**, so they're
solid. A second set of topics (the low-level audio-policy engine, app background-audio rules, the
assistant volume stream) comes from **release-note summaries we couldn't verify locally** — those are
quarantined in Section 6 and flagged clearly.

A note on wording: "**forked**" means a file we've already patched, so Google's edits will conflict.
"**stock A14**" means a file we haven't touched, so Google's edits drop in cleanly. "**ctor**" is a
constructor. "**Oscillates**" means Google changed something one way, then changed it back — so jumping
straight to A17 we take only the final result.

---

## 1. What changes at runtime (the behaviors)

These are the actual behavior changes from A14 to A17, with a note on why each matters to us. The
"Introduced" column is the Android version that added it; the "Net A17 state" is where it lands once all
three transitions are collapsed.

| # | Behavior | Introduced | Net A17 state | Why it matters to Nissan |
|---|----------|-----------|---------------|------------------|
| B1 | **The system fades apps out when they lose focus** — instead of hard-cutting, a losing app is smoothly volume-shaped down (`dispatchAudioFocusChangeWithFade`), even if the app ignores its focus-loss callback | A15 | On when `audioUseFadeManagerConfiguration` + a fade XML are present | **New, independent capability.** *Not* related to our Audio Off (which is just an entertainment mute) — they do different jobs and can coexist. The only overlap is textual: the fade edits `CarAudioFocus` / `FocusInteraction`, which we've also patched, so it's a **merge**, not a behavior clash. Decide separately whether to enable it. |
| B2 | **Volume is clamped into a safe range when audio starts** — gain forced between a min and max % at boot / new source / new playback | A15 | On when `audioUseMinMaxActivationVolume` | **Same idea as our MaxVolumeStartup**, which already lives in `CarVolumeGroup` / `CarAudioGainMonitor`. Two implementations of one feature — likely redundant/conflicting. |
| B3 | **Smarter routing fallback** — only the selected+active zone config is routed, and the default config is always kept last as a fallback | A15 | Same | Cleaner behavior on device connect/disconnect. Little overlap with our code. |
| B4 | **Recovers from an audio-server crash + starts more carefully** — defers init if the audio server isn't up, reinitializes when it restarts | A15/A16 | Same | New resilience. Check it against our boot order and Audio-Off-persist init. |
| B5 | **The car's zone layout can come from the AudioControl HAL** (`getAudioZones()`) instead of the XML file | A16 | Runtime choice: HAL-provided **or** `car_audio_configuration.xml` | **A design decision, not forced.** We use the XML file (v3, or v2+plugin on da2). Keeping XML is fully supported. (See the config-on-HAL docs.) |
| B6 | **The XML config gains version 4 elements** for fade + activation volume | A15/A16 | `car_audio_configuration.xml` v4 | Our configs are v3/v2. We only need the v4 bump **if** we adopt B1/B2. |
| B7 | **Device updates skip work under core routing** — `CoreAudioVolumeGroup.updateDevices(useCoreAudioRouting)` returns early when core routing is on | A17 | Same | Touches our **forked** `CoreAudioVolumeGroup`. |
| B8 | **Audio work moved to the main thread** — all audio policies and callbacks run on `Looper.getMainLooper()` / `getMainExecutor()`; the dedicated helper thread is gone | A16/A17 | Main looper | Re-test the timing of our callbacks (OEM gain callback, master-mute). |
| B9 | **Some things were added then removed** — the persist-fade-balance feature, a per-group event logger, and focus tracing all appear in A16 and are gone by A17 | A17 | Gone | Land on the A17 end-state; don't bother porting the A16 intermediates. |

---

## 2. Files we've already modified (the "forked" list)

These files carry Nissan `[Audio]` patches on top of Android 14 (identified from the commit history).
**Any Google edit to one of these is a merge conflict** — that's what makes them the real work.

| Forked file | What we changed in it |
|---|---|
| `CarAudioService.java` | Audio-Off persist, MaxVolumeStartup, OEM gain-callback wiring, master-mute callback, occupant-zone filter, Siri/E-call volume, cyber-mute, mirror handling |
| `CarVolumeGroup.java` | MaxVolumeStartup apply, skip master-mute event, skip ducking volume-change event, restore volume on unmute |
| `CarAudioFocus.java` | "Audio Off mode" focus dispatch, assistant/call-ring talkback fix, block BT voice-call focus |
| `FocusInteraction.java` | Audio Off mode; prevent entertainment resuming during Audio Off |
| `CarZonesAudioFocus.java` | Audio Off mode — dispatch focus-gain to Entertainment |
| `CoreAudioVolumeGroup.java` | Siri volume after E-call, restore on unmute, default-volume (ticket #222006) |
| `CoreAudioVolumeGroupHelper.java` | **Nissan-new file** (#222006) — per-usage default volumes read from system properties |
| `CarAudioGainCallbackHandler.java` | Adapt to the OEM Audio Gain Callback |
| `CarAudioGainMonitor.java` | MaxVolumeStartup handling |
| `CarAudioSettings.java` | Guest-profile volume bar, CP-song max-volume, Audio-Off persist |
| `CarAudioZoneConfig.java` | Cyber-mute |
| `ZoneAudioPlaybackCallback.java` | Null-device guard on a new playback config |

There is also the **Alliance OEM Car Audio Service plugin** (`vendor/alliance/.../oemcaraudioservices/`),
which `CarAudioService` calls on every focus/ducking/volume decision. It isn't in this list, but its
contract has to keep working through the merge.

---

## 3. The collision map (where "Google changed it" overlaps "we changed it")

This is the heart of the risk. A file is dangerous only when **both** sides touched it. Red = both sides
changed it heavily; orange = both changed it moderately; yellow = we changed it but Google didn't (low
risk, but still needs a re-test because it interacts with the changed code).

| File | Google changes it? | We forked it? | Risk | The conflict, in short |
|---|---|---|---|---|
| `CarAudioService.java` | Yes — heavily, all 3 versions | Yes — heavily | 🔴 **HIGH** | Biggest single conflict: Google's 4-way policy split + async/HAL init vs. our many hooks |
| `CarVolumeGroup.java` | Yes (A15 mapping rewrite, A16/A17) | Yes — heavily | 🔴 **HIGH** | Google's speaker-mapping rewrite vs. our MaxVolumeStartup/mute patches |
| `CarAudioFocus.java` | Yes (A15 fade, A16/A17 tracing) | Yes | 🔴 **HIGH** | Textual merge: Google adds fade dispatch, we've patched the same file for Audio Off (entertainment mute) — no behavior clash |
| `FocusInteraction.java` | Yes (A15 signature change, A16/A17) | Yes | 🔴 **HIGH** | Textual merge: Google changed method signatures, we've patched it for Audio Off — no behavior clash |
| `CoreAudioVolumeGroup.java` | Yes (A15 rewrite, A17 device update) | Yes | 🔴 **HIGH** | Google's ctor/mute rewrite vs. our Siri fix + #222006 |
| `CarZonesAudioFocus.java` | Yes (A15 wrapper) | Yes | 🟠 **MED** | Google's AudioManager wrapper vs. our Audio Off dispatch |
| `CarAudioZoneConfig.java` | Yes (A15/A16/A17) | Yes | 🟠 **MED** | Google's fade/selected-state fields vs. our cyber-mute |
| `CarAudioGainCallbackHandler.java` | No | Yes | 🟡 LOW | Interacts only — re-test |
| `CarAudioGainMonitor.java` | No | Yes | 🟡 LOW | Holds our MaxVolumeStartup — reconcile intent with Google's activation volume (B2) |
| `CarAudioSettings.java` | No | Yes | 🟡 LOW | Interacts only |
| `ZoneAudioPlaybackCallback.java` | No | Yes | 🟡 LOW | Google adds `CarAudioPlaybackMonitor` alongside it |
| `CoreAudioVolumeGroupHelper.java` | No (Nissan-only) | Yes (new) | 🟡 LOW | Keep it, but re-point it at the rewritten `CoreAudioVolumeGroup` |

---

## 4. The full file list, grouped by difficulty

**A. New files to add** — Google added these; none exist in our tree yet (~13 files).
Android 15: `AudioManagerWrapper`, `CarActivationVolumeConfig`, `CarAudioDeviceCallback`,
`CarAudioFadeConfigurationHelper`, `CarAudioParserUtils`, `CarAudioPlaybackMonitor`,
`CarAudioProtoUtils`, `CarAudioServerStateCallback`.
Android 16: `AudioControlZoneConverter`, `AudioControlZoneConverterUtils`,
`CarAudioZonesHelperAudioControlHAL`, `CarAudioZonesHelperImpl`, `TEST_MAPPING`.
> `AudioManagerWrapper` is the spine of the A15 refactor — it wraps every `AudioManager` call, so
> pulling it in touches almost every other file.

**B. Structural change** — `CarAudioZonesHelper` turns from a class into an interface; the `hal/`
wrapper classes gain `getAudioZones()` / `supportsAudioZones()`.

**C. Modify — stock A14 (mechanical, low risk)** — we haven't touched these, so Google's edits apply
cleanly: `CarAudioDeviceInfo`, `CarAudioDynamicRouting`, `CarAudioContext`, `CarAudioVolumeGroup`,
`CarAudioZone`, `CarVolumeGroupFactory`, `CoreAudioVolumeGroupCallback`, `CarAudioUtils`,
`ContentObserverFactory`, `hal/AudioControlWrapper*`.

**D. Modify — forked (the real merges, HIGH/MED risk)** — the red/orange rows from Section 3:
`CarAudioService`, `CarVolumeGroup`, `CarAudioFocus`, `FocusInteraction`, `CoreAudioVolumeGroup`,
`CarZonesAudioFocus`, `CarAudioZoneConfig`.

**E. Re-validate (Nissan-only, Google didn't change them but they interact)** —
`CoreAudioVolumeGroupHelper`, `CarAudioGainCallbackHandler`, `CarAudioGainMonitor`, `CarAudioSettings`,
`ZoneAudioPlaybackCallback`, and the Alliance OEM plugin.

---

## 4b. Deep dive — the volume-group classes

Volume is where a lot of our customization lives, so here's that family in detail.

The classes and how they relate today:
```
CarVolumeGroup (abstract base)
 ├─ CarAudioVolumeGroup      (dynamic-routing path)
 └─ CoreAudioVolumeGroup     (core-audio path)   ← we've forked this one
CarVolumeGroupFactory         (builds the two subclasses)
CoreAudioVolumeGroupCallback
CoreAudioVolumeGroupHelper    ← Nissan-only (#222006), Google doesn't have it
CarActivationVolumeConfig     ← Google-new (A15), missing from our tree
```

The single biggest thing Google changed here: **how a volume group maps a sound to a speaker.** Today we
map **by address** (a text name). Google's version maps **by device object** and adds an "activation
volume" config (the min/max-on-start feature). Google also **split muting into two steps** — one that
mutes the hardware, one that saves the setting. Both of those are exactly where our patches sit.

Per-class detail:

| Class | What it looks like today (Nissan/A14) | What it becomes by A17 | Risk |
|---|---|---|---|
| **`CarVolumeGroup`** (base) | Address-based: fields `SparseArray<String> mContextToAddress` + `ArrayMap<String,CarAudioDeviceInfo> mAddressToCarAudioDeviceInfo`; ctor takes `(context, settings, contextToAddress, addressToDeviceInfo,…)` | Device-based: `SparseArray<CarAudioDeviceInfo> contextToDevices`, **plus a new `CarActivationVolumeConfig` parameter**; new activation methods (`getMax/MinActivationGainIndex`, `getActivationVolumeInvocationType`, `handleActivationVolume`); new `isActive` / `audioDevicesAdded/Removed` / `updateDevices(boolean)` / `validateDeviceTypes` / `getAudioDeviceAttributes`; **mute split**: `setMute()` → `applyMuteLocked()` + `saveMuteStateToSettingsLocked()`. (An event logger was added in A16 and removed in A17 — **don't port it**.) | 🔴 HIGH |
| **`CoreAudioVolumeGroup`** (subclass) | Raw `AudioManager`; `isAmGroupMuted()` via `AudioManagerHelper.isVolumeGroupMuted` behind an `isPlatformVersionAtLeastU` guard; our #222006 default-volume + `storeAndPersistGainIndexLocked`; Siri/E-call + restore-on-unmute patches | Uses `AudioManagerWrapper`; the `isPlatformVersionAtLeastU` guards are dropped; mute goes through `adjustVolumeGroupVolume()`; **`updateDevices(boolean useCoreAudioRouting)` returns early when core routing is on** | 🔴 HIGH |
| **`CarAudioVolumeGroup`** (subclass) | Stock A14 | Inherits the base ctor/mapping change; its only standalone A16→A17 edit is a javadoc fix | 🟢 LOW |
| **`CarVolumeGroupFactory`** | Raw `android.media.AudioManager`; address maps | `AudioManagerWrapper`; `SparseArray<CarAudioDeviceInfo>`; passes the new `CarActivationVolumeConfig` | 🟢 LOW (stock) |
| **`CoreAudioVolumeGroupCallback`** | Stock A14, raw `AudioManager` | `AudioManagerWrapper`; **net A17 = `init(Executor executor)`** — the executor is injected at init time, not stored in the ctor (this one oscillates across A15→A16→A17) | 🟢 LOW |
| **`CarActivationVolumeConfig`** | Missing | **New file to add** — holds per-group min/max % and the invocation type (`ON_BOOT` / `ON_SOURCE_CHANGED` / `ON_PLAYBACK_CHANGED`) | add |
| **`CoreAudioVolumeGroupHelper`** | Nissan-only (#222006) | Keep it, but re-wire it onto the rewritten `CoreAudioVolumeGroup` — its `getDefaultVolumeForAttributes(attrs, max, min)` call sits inside the ctor Google rewrote | 🟡 re-wire |

**Why the base and core classes are HIGH risk:** Google rewrote the *exact surfaces we patched* — the
mute path (now `applyMuteLocked` + `saveMuteStateToSettingsLocked`) and the sound→speaker mapping
(address → device). Our patches in `CarVolumeGroup` (MaxVolumeStartup, master-mute skip, ducking
volume-change skip, restore-on-unmute) and in `CoreAudioVolumeGroup` (#222006 default volume +
`storeAndPersistGainIndexLocked`, Siri/E-call volume) all land in those same methods.

**Two decisions this forces:**
- **MaxVolumeStartup (ours) vs. Google's min/max activation volume (B2)** — same goal, two mechanisms.
  Pick one, or the boot volume gets clamped twice. Google's lives in `CarActivationVolumeConfig` +
  `handleActivationVolume()`; ours lives in `CarVolumeGroup` / `CarAudioGainMonitor`.
- **The #222006 default-volume + persistence logic** has to be re-applied on top of Google's new
  constructor and mute split — not pasted back as-is.

---

## 5. Decisions to make (behavior)

1. **Decide whether to enable Google's system fade at all (B1).** It's a new, optional capability
   (fade a non-compliant app to silence on focus loss). It is **not** related to our Audio Off
   (entertainment mute) — they can coexist. The only Audio-Off connection is that the fade code edits the
   same focus files we've patched, so that's a textual merge to manage, not a behavior decision.
   *(Files: `CarAudioFocus`, `FocusInteraction`, `CarZonesAudioFocus`.)*
2. **Reconcile MaxVolumeStartup with Google's activation volume (B2).** Same feature, two
   implementations — pick one to avoid double-clamping. *(`CarVolumeGroup`, `CarAudioGainMonitor`,
   `CarActivationVolumeConfig`.)*
3. **Zone-config source (B5).** Keep `car_audio_configuration.xml` (the fallback path stays supported) or
   move to the HAL's `getAudioZones()`. **Recommendation: keep XML** — less churn, and da2 already routes
   through the Alliance plugin.
4. **Only bump the XML config to v4 if we adopt fade/activation-volume (B6).** Otherwise our v3/v2 configs
   stay valid.
5. **Re-check OEM gain callback + master-mute timing** after the move to the main thread (B8).
6. **Take the A17 end-state, skip the intermediates (B9)** — don't port the event logger / tracing /
   persist-fade-balance that A16 added and A17 removed.

---

## 6. Separate, UNVERIFIED track (release-note only — outside the car-audio diffs)

These come from release-note summaries, not the source diffs. **Confirm them against real
`frameworks/av` + `hardware/interfaces/audio` diffs before planning any work.**

- **Low-level audio-policy (CAP) engine config → core Audio HAL `getEngineConfig()`, plus the
  `STRATEGY_` rename** (A16). We use static XML + parameter-framework with lowercase strategy names, so
  this would be a full rewrite — and it depends on migrating the core Audio HAL from HIDL to AIDL (we're
  on HIDL 6.0/7.0). See the config-on-HAL docs for the detail and recommendation (short version: keep
  XML). 
- **Background Audio Hardening** (A17) — background audio/focus/volume fails without a visible screen or a
  proper foreground service. This is app-side (foreground-service types), not `car/audio`.
- **Dedicated assistant volume stream** (A17) — `USAGE_ASSISTANT` gets its own stream; map it to a volume
  group and add a Settings slider.
- **AudioControl HAL version mismatch** — our manifests say v1, a fragment says v3, the build says V2.
  Settle on v3. (Cheap; do it regardless.)

---

## 7. Suggested order of work

1. **Do the easy Android 15 layer first** — add `AudioManagerWrapper` and the other 7 new A15 classes,
   and apply the stock-A14 modifications (group 4C). Large but low-risk; it's the foundation.
2. **Merge the HIGH-risk forked files (group 4D) one at a time**, each followed by a targeted test of the
   Nissan feature living in it (Audio Off, MaxVolumeStartup, Siri/E-call volume, master mute, ducking).
3. **Take the Android 16 zone/interface change** — `CarAudioZonesHelper` as an interface +
   `CarAudioZonesHelperImpl` — but keep reading our XML (defer the HAL `getAudioZones()` path).
4. **Apply the Android 17 cleanups** — main thread, the core-routing early-return, and dropping the
   added-then-removed intermediates.
5. **Settle the two overlaps with the audio team** (decisions 5.1 and 5.2) before merging the focus and
   volume files.
6. **Run the Section 6 items as their own projects** (core-HAL/CAP, background-audio hardening, assistant
   stream).

### How we'll verify it
- Boot each variant (`aivi2_n_full`, `aivi2_n_da2`, `aasp_n` emulator). Check `dumpsys car_service`
  (audio) and `dumpsys audio` for zones/groups/policies, and watch `CarAudioService` / `AudioPolicy`
  logcat for config-parse errors.
- Regression-test the Nissan-custom flows: **Audio Off mode**, **MaxVolumeStartup**, master-mute,
  ducking, Siri/E-call volume, guest-profile volume bar, OEM gain callback.
- Standard audio checks: volume keys, focus arbitration, ADAS ducking, ANC.
