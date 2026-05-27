# Architectural Master Audio Changelog: Android 14 → Android 17
## AAOS Audio Framework — Complete Lifecycle (API Level 34 → 37)
### `CarAudioService` | `AudioControlService` | `AudioControlHAL`

> **Compiled from:** `Audio_Framework_A14_to_A15.md` · `Audio_Framework_A15_to_A16.md` · `Audio_Framework_A16_to_A17.md`
>
> **Scope:** `platform/packages/services/Car/service/src/com/android/car/audio/`, `android.car.media`,
> `CarAudioService`, `AudioControl HAL`, Audio Focus, Volume Management, Audio Zones, Audio Routing, Audio Fade.
>
> **Source basis:** Official AOSP & AAOS release notes. No local AOSP git repository was available;
> git commands for raw diff generation are provided in the appendix.

---

## Executive Summary: Macro Architectural Shifts (A14 → A17)

The Android 14-to-17 span is the most consequential four-year period of architectural evolution in the AAOS audio stack. Five shifts stand out as foundational.

### Shift 1 — The OEM Programmability Revolution (Android 14)
A single release introduced: OEM Car Audio Plugin Services (allowing OEMs to intercept every focus decision via `OemCarAudioFocusService`), Dynamic Audio Zone Configurations (runtime zone switching via `switchAudioZoneToConfig()`), Audio Mirroring (`enableMirrorForAudioZones()`), Passenger Audio Cast (`requestMediaAudioOnPrimaryZone()`), and the CAP Engine flags (`useCoreAudioVolume`, `useCoreAudioRouting`). `car_audio_configuration.xml` advanced to version 3.

### Shift 2 — System-Enforced Audio Fade (Android 15)
Android 15 closed the compliance gap in audio focus enforcement. `FadeManagerConfiguration`, `car_audio_fade_configuration.xml`, and `audioUseFadeManagerConfiguration` gave `CarAudioService` the ability to apply `VolumeShaper`-based fade-out to non-compliant apps without their cooperation. Simultaneously, `audioUseMinMaxActivationVolume` provided a first-ever mechanism to prevent dangerously loud or inaudibly quiet audio activation events. The schema advanced to version 4.

### Shift 3 — CAP Engine Goes AIDL-Native (Android 16)
The CAP engine's configuration mechanism migrated from static vendor partition XML to live AIDL API calls via new parcelables: `AudioHalCapConfiguration`, `AudioHalCapCriterionV2`, `AudioHalCapDomain`, `AudioHalCapParameter`, `AudioHalCapRule`, and `AudioHalEngineConfig.CapSpecificConfig`. This also introduced a **breaking naming convention change**: all product strategy names now require the `STRATEGY_` prefix. This transition effectively marks the **final phase of HIDL retirement** for automotive audio on new devices.

### Shift 4 — Background Audio Hardening (Android 17)
Android 17 introduced the most disruptive behavioral change for AAOS app developers: system-enforced Foreground Service requirements for all background audio interactions. `AudioTrack.write()`, `AAudioStream_write()`, `requestAudioFocus()`, and volume adjustment APIs fail silently unless the app holds a visible Activity or an active non-`SHORT_SERVICE` FGS. This impacts every AAOS app that plays audio with the screen off, starts audio on `BOOT_COMPLETE`, or manages background VoIP/navigation audio.

### Shift 5 — USAGE_ASSISTANT Volume Independence (Android 17)
`AudioManager.MODE_ASSISTANT_CONVERSATION` and the isolation of `USAGE_ASSISTANT` into its own volume stream break the long-standing coupling between assistant audio and media volume. AAOS OEMs must now map the new stream to a volume group in `car_audio_configuration.xml` and expose independent volume controls in the Audio Settings UI.

---

## CarAudioService Updates — Full History (A14 → A17)

### Android 15 Changes

#### System-Enforced Audio Fade
- **New flag:** `audioUseFadeManagerConfiguration` (default: `false`, OEM RRO)
- **New class:** `FadeManagerConfiguration` (`android.car.media`)
  - Builder: `setFadeOutDurationForUsage()`, `setFadeInDurationForUsage()`, `setFadeableUsages()`, `setUnfadeableContentTypes()`, `setUnfadeableAudioAttributes()`
  - Default fade-out: 500 ms; default fade-in: 2000 ms; converted to `VolumeShaper.Configuration`
- **New class:** `CarAudioFadeConfiguration` (`android.car.media`) — runtime fade config for OEM focus service integration
- **New XML:** `car_audio_fade_configuration.xml` on vendor overlay partition
- **`car_audio_configuration.xml` → Version 4** with `<applyFadeConfigs>` / `<fadeConfig>` elements

#### Minimum and Maximum Activation Volume
- **New flag:** `audioUseMinMaxActivationVolume` (default: `false`, OEM RRO)
- **New XML element:** `<activationVolumeConfig>` with `invocationType` values: `onBoot`, `onSourceChanged`, `onPlaybackChanged`
- Monitoring via `AudioPlaybackCallback`, call-state changes, HAL `IFocusListener#requestAudioFocus()`

#### OEM Focus Plugin — Per-Attribute Fade Configuration
- **New builder method:** `OemCarAudioFocusResult.Builder#setAudioAttributesToCarAudioFadeConfigurationMap(Map<AudioAttributes, CarAudioFadeConfiguration>)`

#### Bluetooth Headset as Audio Output Device
- User-selectable in `Settings > Sound & Vibration > Audio output`
- Routes via `AUDIO_DEVICE_OUT_BUS` infrastructure; single BT stream constraint

---

### Android 16 Changes

#### AAudio OEM-Defined Audio Attribute Tags
- OEM-defined tags on AAudio streams consumed by `CarAudioService` for routing and volume decisions
- Integrates with `<oemContext>` in `car_audio_configuration.xml` v3+
- When CAP is active (`useCoreAudioRouting=true`), OEM context must match a CAP engine product strategy

#### Fade and Balance Getter APIs (AAOS 25Q4)
- **New `@SystemApi`:** `getFadeTowardFront()` — returns current fade value `[-1.0, +1.0]`
- **New `@SystemApi`:** `getBalanceTowardRight()` — returns current balance value `[-1.0, +1.0]`
- Values **persist per user across ignition cycles**
- `car_audio_configuration.xml` unchanged at **Version 4**

#### Alternative App Audio Controls (AAOS 25Q4)
- Independent volume control for communication apps while driving
- Separate audio routing path for communication vs. media audio

---

### Android 17 Changes

#### Background Audio Hardening (Critical)

Tier 1 (all apps): Visible Activity OR FGS (not `SHORT_SERVICE`) required for background audio.
Tier 2 (apps targeting API 37): FGS must have while-in-use capability (e.g., `mediaPlayback`).

```xml
<!-- AndroidManifest.xml — required for apps targeting API 37 -->
<service
    android:name=".AudioPlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false"/>
```

Silent failures: `AudioTrack.write()`, `AAudioStream_write()`, `requestAudioFocus()` (returns `AUDIOFOCUS_REQUEST_FAILED`), `setStreamVolume()`, `adjustVolume()`, `setStreamMute()`, `setRingerMode()`.

AAOS patterns affected: background music with display off, boot-triggered audio, backgrounded navigation audio, background VoIP, HAL focus requests.

```bash
adb shell cmd audio set-enable-hardening enable   # Enforce
adb shell cmd audio set-enable-hardening throw    # Enforce + exceptions
adb shell cmd audio set-enable-hardening disable  # Disable
adb logcat | grep AudioHardening                  # Monitor violations
```

#### Dedicated Assistant Volume Stream
- **New constant:** `AudioManager.MODE_ASSISTANT_CONVERSATION`
- `USAGE_ASSISTANT` isolated into its own independent volume stream (no longer shares `STREAM_MUSIC`)
- OEM action: update `car_audio_configuration.xml` volume group mapping + Audio Settings UI

#### No New AAOS-Specific APIs Confirmed
- `CarAudioService` architecture: no confirmed changes
- `car_audio_configuration.xml`: v4 — no v5 confirmed
- Audio zone, volume group, OEM plugin interfaces: no new APIs confirmed

---

### Deprecated APIs — Complete History

| API | Class | Deprecated In |
|-----|-------|--------------|
| `isDynamicRoutingEnabled()` | `CarAudioManager` | Android 14 |
| `getExternalSources()` | `CarAudioManager` | Android 14 |
| `createAudioPatch(String, int, int)` | `CarAudioManager` | Android 14 |
| `releaseAudioPatch(CarAudioPatchHandle)` | `CarAudioManager` | Android 14 |

### New CarAudioManager APIs by Android Version

| Android | API | Type |
|---------|-----|------|
| 14 | `registerCarVolumeGroupEventCallback()` | `@SystemApi` |
| 14 | `getAudioZoneConfigInfos(int zoneId)` | `@SystemApi` |
| 14 | `switchAudioZoneToConfig(...)` | `@SystemApi` |
| 14 | `enableMirrorForAudioZones(List<Integer>)` | `@SystemApi` |
| 14 | `requestMediaAudioOnPrimaryZone(...)` | `@SystemApi` |
| 14 | `allowMediaAudioOnPrimaryZone(long, boolean)` | `@SystemApi` |
| 16 (25Q4) | `getFadeTowardFront()` | `@SystemApi` |
| 16 (25Q4) | `getBalanceTowardRight()` | `@SystemApi` |

### Configuration Flags — Complete History

| Flag | Introduced | Default | Purpose |
|------|-----------|---------|---------|
| `audioUseDynamicRouting` | Android 9 | Required `true` | Enable AAOS bus-based routing |
| `audioUseHalDuckingSignals` | Android 12 | `true` | Ducking signals to AudioControl HAL |
| `audioUseCarVolumeGroupMuting` | Android 12 | `false` | Per-group muting via HAL |
| `audioPersistMasterMuteState` | Android 12 | `true` | Restore global mute on boot |
| `audioUseCarVolumeGroupEvent` | Android 14 | `false` | Enable `CarVolumeGroupEventCallback` |
| `useCoreAudioVolume` | Android 14 | `false` | CAP-based volume management |
| `useCoreAudioRouting` | Android 14 | `false` | CAP-based routing |
| `audioUseFadeManagerConfiguration` | **Android 15** | `false` | **System-enforced fade on focus loss** |
| `audioUseMinMaxActivationVolume` | **Android 15** | `false` | **Min/max volume at activation** |

### `car_audio_configuration.xml` Version History

| Version | Android | New Elements |
|---------|---------|-------------|
| v1 | Android 10 | Zones, volume groups |
| v2 | Android 11 | `audioZoneId`, `occupantZoneId` |
| v3 | Android 14 | `<oemContext>`, `<zoneConfigs>`, `<zoneConfig>` |
| v4 | Android 15 | `<activationVolumeConfig>`, `<applyFadeConfigs>`, `<fadeConfig>` |
| _(none)_ | Android 16/17 | Schema unchanged |

---

## AudioControlService Updates — Full History (A14 → A17)

The `AudioControlService` C++ proxy layer had **no interface changes** in any of the Android 15, 16, or 17 releases. It remains on **AIDL v3** throughout this entire period.

The major architectural changes across these releases (system-enforced fade, CAP AIDL migration, background audio hardening) each operate at a different layer:
- Fade/volume enforcement: Java `CarAudioService` layer using `VolumeShaper`
- CAP engine config: `android.hardware.audio.core` (not `IAudioControl`)
- Background audio hardening: `audioserver` enforcement layer

The `IAudioControl` AIDL v3 interface is identical in Android 14, 15, 16, and 17.

---

## AudioControlHAL Updates — Full History (A14 → A17)

### AudioControl HAL Version Table

| Version | Android | Key New Capabilities | Status |
|---------|---------|---------------------|--------|
| HIDL 1.0 | Android 9 | `setFadeTowardFront`, `setBalanceTowardRight` | **Deprecated** |
| HIDL 2.0 | Android 12 | Focus listener, HAL focus requests, muting, ducking | **Deprecated** |
| AIDL 1.0 | Android 12L | HIDL→AIDL migration (same APIs, stable interface) | Maintained |
| AIDL 2.0 | Android 13 | `onAudioFocusChangeWithMetaData`, `registerGainCallback` | Maintained |
| AIDL 3.0 | Android 14 | `setModuleChangeCallback`, `IModuleChangeCallback#onAudioPortsChanged` | **Current** |
| _(none)_ | Android 15/16/17 | No new `IAudioControl` version; CAP migrated to core Audio HAL in Android 16 | — |

### Android 16: CAP Engine AIDL Migration (Core Audio HAL)

The most significant HAL-layer change across the A14–A17 span occurred in Android 16 within `android.hardware.audio.core` (not `IAudioControl` directly):

**New AIDL parcelables delivering CAP configuration via `getEngineConfig()`:**
- `AudioHalCapConfiguration` — top-level CAP config container
- `AudioHalCapCriterionV2` — criterion definitions (available output devices, telephony mode, etc.)
- `AudioHalCapDomain` — parameter domain with routing rules
- `AudioHalCapParameter` — individual configurable parameters (path + value)
- `AudioHalCapRule` — routing and volume rules evaluated against criteria
- `AudioHalEngineConfig.CapSpecificConfig` — integration point in existing `AudioHalEngineConfig`

**Breaking change:** All product strategy names require `STRATEGY_` prefix (e.g., `media` → `STRATEGY_MEDIA`).

**HAL compatibility:** `EngineConfigXmlConverter.h` converts legacy vendor XML to AIDL parcelables for older vendor HALs.

### HIDL Final Retirement Status

HIDL-based AudioControl HAL versions (1.0, 2.0) have been deprecated since Android 12L. The Android 16 CAP AIDL migration marks the point where new devices with fully updated vendor HALs have no remaining HIDL dependency in the automotive audio stack. Devices upgrading from Android 14/15 vendor HALs retain a supported XML-fallback path.

---

## Appendix: Git Commands for Raw Patch Generation

```bash
cd <AOSP_ROOT>/packages/services/Car
git fetch --tags

# A14 → A15
git diff android-14.0.0_r1..android-15.0.0_r1 \
  -- service/src/com/android/car/audio/ cpp/audiocontrol/ \
  > a14_to_a15_audio.patch

# A15 → A16
git diff android-15.0.0_r1..android-16.0.0_r1 \
  -- service/src/com/android/car/audio/ cpp/audiocontrol/ \
  > a15_to_a16_audio.patch

# A16 → A17 (use main until android-17.0.0_r1 tag is published)
git diff android-16.0.0_r1..main \
  -- service/src/com/android/car/audio/ cpp/audiocontrol/ \
  > a16_to_a17_audio.patch
```

---

## Sources

- [AAOS 24Q3 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-24q3)
- [AAOS 25Q2 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q2)
- [AAOS 25Q4 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q4)
- [Audio Control HAL — AOSP](https://source.android.com/docs/automotive/audio/audio-control-hal)
- [Volume Management — AOSP](https://source.android.com/docs/automotive/audio/volume-management)
- [Audio Focus — AOSP](https://source.android.com/docs/automotive/audio/audio-focus)
- [Car Audio Plugin Service — AOSP](https://source.android.com/docs/automotive/audio/car-audio-plugin)
- [Configurable Audio Policy AIDL HAL — AOSP](https://source.android.com/docs/core/audio/aidl-cap)
- [Configurable Audio Policy Engine — AOSP](https://source.android.com/docs/automotive/audio/configurable-audio-policy-engine)
- [Car Audio Configuration — AOSP](https://source.android.com/docs/automotive/audio/audio-policy-configuration)
- [Audio Configuration Flags — AOSP](https://source.android.com/docs/automotive/audio/config-flags)
- [Background Audio Hardening — Android 17](https://developer.android.com/about/versions/17/changes/bg-audio)
- [Android 17 Features and APIs](https://developer.android.com/about/versions/17/features)
- [Android 17 Behavior Changes: All Apps](https://developer.android.com/about/versions/17/behavior-changes-all)
- [CarAudioManager API Reference](https://developer.android.com/reference/android/car/media/CarAudioManager)
