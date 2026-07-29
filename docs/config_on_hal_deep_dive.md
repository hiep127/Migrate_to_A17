# Deep Dive — "Audio Config moves to the HAL" (A14→A17) + Recommendation for Nissan

**Purpose:** the migration docs repeatedly say Android is moving audio **configuration onto the HAL**.
That phrase actually covers **two completely separate mechanisms, in two different HALs, replacing two
different config files**. This file separates them, states exactly where `/a14full` stands on each,
what it would take to adopt, and what I recommend.

---

## The two "config on HAL" mechanisms (do not conflate)

| | **Mechanism 1 — Zone config on HAL** | **Mechanism 2 — Policy-engine (CAP) config on HAL** |
|---|---|---|
| What moves | Car audio **zones / volume groups / contexts / routing** | Core **audio policy engine** rules (routing/volume per product strategy) |
| HAL | `android.hardware.automotive.audiocontrol` (**AudioControl HAL**) | `android.hardware.audio.core` (**core Audio HAL**) |
| API | `IAudioControl.getAudioZones()` + `getAudioDeviceConfiguration()` | `IModule.getAudioPolicyConfig()/getEngineConfig()` → `AudioHalEngineConfig.CapSpecificConfig` |
| Replaces | `car_audio_configuration.xml` | `audio_policy_engine_configuration.xml` + parameter-framework `.pfw` |
| Consumed by | `CarAudioService` (Java) via `CarAudioZonesHelperAudioControlHAL` | `libaudiopolicymanager` / `AudioPolicyManager` (native) |
| Introduced | **A16**, confirmed in the real `car/audio` git diffs | **A16**, release-note only — **unverified here** |
| Backward compat | XML path fully retained (runtime choice) | `EngineConfigXmlConverter.h` converts existing XML→AIDL |

They are independent. You can adopt one, both, or neither.

---

## Mechanism 1 — Zone config via the AudioControl HAL (`getAudioZones()`)

### What AOSP added (A16, from the real diffs)
- `IAudioControl` gained `boolean supportsAudioZones()` (implicit via feature), `AudioZone[] getAudioZones()`,
  `AudioDeviceConfiguration getAudioDeviceConfiguration()`, and a set of new parcelables (`AudioZone`,
  `AudioZoneConfig`, `AudioZoneContext`, `VolumeGroupConfig`, `AudioZoneFadeConfiguration`,
  `VolumeActivationConfiguration`, `AudioDeviceConfiguration`).
- `CarAudioZonesHelper` became an **interface** with two impls chosen at runtime:
  ```java
  boolean halSupportsZoneConfig = mAudioControlWrapper.supportsAudioZones();
  mCarAudioZonesHelper = halSupportsZoneConfig
          ? new CarAudioZonesHelperAudioControlHAL(mAudioControlWrapper, ...)  // HAL-provided
          : new CarAudioZonesHelperImpl(mContext, ...);                        // XML (existing)
  ```
- `AudioControlZoneConverter` + `AudioControlZoneConverterUtils` translate the HAL AIDL objects into
  `CarAudioZone` / `CarVolumeGroup` / `CarAudioContext`, validating each step (null + log on failure).

### Where `/a14full` stands  ← **the blocker is the HAL interface, not just CarService**
- **AudioControl AIDL in-tree tops out at v3.** The v3 (and `current`) `IAudioControl.aidl` at
  `hardware/interfaces/automotive/audiocontrol/aidl/.../IAudioControl.aidl` declares only:
  `onAudioFocusChange`, `onDevicesToDuckChange`, `onDevicesToMuteChange`, `registerFocusListener`,
  `setBalanceTowardRight`, `setFadeTowardFront`, `onAudioFocusChangeWithMetaData`,
  `setAudioDeviceGainsChanged`, `registerGainCallback`, `setModuleChangeCallback`, `clearModuleChangeCallback`.
  **`getAudioZones` / `AudioZone` / `getAudioDeviceConfiguration` do not exist anywhere in the tree.**
- The Nissan vendor HAL (`vendor/alliance/.../audiocontrol/default/`) is built against **V2-ndk** and
  implements none of the zone APIs. Manifests declare v1 / fragment v3 / build V2 (inconsistent).
- `CarAudioService` loads zones **only** from `car_audio_configuration.xml` (`CarAudioZonesHelper`);
  the sole HAL fallback is the deprecated `CarAudioZonesHelperLegacy` (V1).

### What adopting Mechanism 1 would cost
1. **Upgrade the AudioControl HAL AIDL v3 → v5** (pull the A16 interface + all new parcelables into
   `hardware/interfaces/automotive/audiocontrol`).
2. **Implement `getAudioZones()` / `getAudioDeviceConfiguration()` in the Alliance vendor HAL** — i.e.
   re-express today's `car_audio_configuration.xml` topology as AIDL objects returned by the HAL.
3. Add the CarService side (`CarAudioZonesHelperAudioControlHAL` + converters) — the "add" files from §4A.
4. Reconcile the v1/v2/v3 manifest inconsistency.

### Value vs. cost
Value: OTA-updatable zone topology without a system-image XML; one binary serving multiple vehicle
configs. For a **single-zone Nissan head unit whose topology is stable**, that value is low, and da2
already routes through the Alliance plugin + XML.

---

## Mechanism 2 — CAP policy-engine config via the core Audio HAL (`getEngineConfig()`)

### What AOSP changed (A16, release-note sourced — treat as unverified)
- The **Configurable Audio Policy (CAP)** engine config can be delivered by the core Audio HAL via
  `getEngineConfig()` returning `AudioHalEngineConfig.CapSpecificConfig`
  (`AudioHalCapConfiguration` / `AudioHalCapCriterionV2` / `AudioHalCapDomain` / `AudioHalCapParameter` /
  `AudioHalCapRule`) instead of reading `audio_policy_engine_configuration.xml` + `.pfw` at boot.
- **Breaking rename:** product strategy names require a `STRATEGY_` prefix.
- **Backward compat:** `EngineConfigXmlConverter.h` converts the existing vendor XML into those AIDL
  parcelables — so existing XML keeps working under an **AIDL** core HAL.

### Where `/a14full` stands
- Engine is the **configurable engine driven by static XML + parameter-framework**:
  `engine_library="configurable"`, `audio_policy_engine_product_strategies.xml` (21 strategies,
  **lowercase, no `STRATEGY_` prefix**: `music`, `nav_guidance`, `voice_call`, `oem_ipa`, …),
  `device_for_product_strategies.pfw` / `device_for_input_source.pfw`.
- Core audio HAL is still **HIDL** (6.0 on da2, 7.0 on full). There is **no AIDL `android.hardware.audio.core`**.

### Hard dependency
`getEngineConfig()` exists only on the **AIDL** core Audio HAL. So Mechanism 2 is **gated behind a full
core-audio HIDL→AIDL migration** — the single largest piece of work in the whole A17 effort, and it sits
in `frameworks/av` + `hardware/interfaces/audio`, **outside** the `car/audio` diffs this analysis verified.

---

## Recommendation for Nissan

### Headline: keep the static XML configs; do **not** move config into either HAL for A17.
Both mechanisms are **opt-in and backward-compatible by design** — AOSP kept the XML path (Mechanism 1)
and ships an XML→AIDL converter (Mechanism 2). Nissan gets A17 behavior without rewriting config.

### Mechanism 1 (zone config on HAL) — **DEFER**
- Requires an AudioControl HAL **AIDL v3→v5 interface upgrade + a new vendor HAL implementation of
  `getAudioZones()`** — high cost, and the XML path (`CarAudioZonesHelperImpl`) stays fully supported.
- **Do:** take `CarAudioZonesHelper`-as-interface + `CarAudioZonesHelperImpl` (needed anyway for the A16
  refactor) and keep `supportsAudioZones()` returning **false** so the XML impl is selected.
- **Don't:** implement `getAudioZones()` in the Alliance HAL for A17. Revisit only if a future multi-zone
  / dynamic-topology product needs it.

### Mechanism 2 (CAP engine on HAL) — **KEEP XML; converge only if forced**
- If A17 forces the AIDL core Audio HAL, keep the existing `audio_policy_engine_*.xml` + `.pfw` and rely
  on `EngineConfigXmlConverter` — **do not hand-author AIDL `CapSpecificConfig`**.
- The **`STRATEGY_` rename is the one config edit likely unavoidable** if/when the CAP path is AIDL-native:
  rename all 21 strategies (+ every reference in `.pfw`, volumes XML, and any
  `/Policy/.../product_strategies/<name>` path). Plan it as an isolated, mechanical, well-tested change.
- **Verify first:** this whole mechanism is release-note-sourced. Confirm against real
  `frameworks/av` + `hardware/interfaces/audio` A16/A17 diffs before committing any effort.

### The one HAL config item to fix regardless — **AudioControl HAL version hygiene**
Independent of both mechanisms: settle the AudioControl HAL on a **single consistent AIDL version (v3)**
across device manifests (currently v1), the VINTF fragment (currently v3), and the build file (currently
V2). This is required for the existing `IModuleChangeCallback` path and is cheap.

### Net for A17
| Item | Recommendation | Effort |
|---|---|---|
| Zone config via `getAudioZones()` | Defer; keep XML, `supportsAudioZones()=false` | — (take interface refactor only) |
| CAP engine via `getEngineConfig()` | Keep XML+PFW via `EngineConfigXmlConverter`; don't author AIDL config | Low (if core HAL goes AIDL) |
| `STRATEGY_` product-strategy rename | Do only if CAP goes AIDL-native; isolated mechanical change | Medium, wide |
| Core Audio HAL HIDL→AIDL | Separate epic; gates Mechanism 2 | High |
| AudioControl HAL v1/v2/v3 reconcile | **Do now** — settle on v3 | Low |

---

## Verification for the config layer
- `adb shell lshal | grep -i audiocontrol` → one consistent AIDL version; `dumpsys media.audio_policy`
  shows the engine (`configurable`) and per-strategy routing loaded.
- `adb shell dumpsys car_service` (audio) → zones/volume-groups came from XML (`CarAudioZonesHelperImpl`),
  not HAL, confirming Mechanism 1 stayed off.
- If the `STRATEGY_` rename is applied: grep CAP/PFW logs for any unresolved
  `product_strategies/<oldname>` path.
- To make the AOSP side verifiable locally (tags absent on the Nissan fork):
  ```bash
  cd /home/worker/a14full/android/hardware/interfaces
  git remote add aosp https://android.googlesource.com/platform/hardware/interfaces
  git fetch aosp --tags
  git diff aosp/android-14.0.0_r1..aosp/android-16.0.0_r1 -- automotive/audiocontrol/ audio/
  ```
