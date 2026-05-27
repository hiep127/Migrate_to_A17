# Master Audio Framework Changelog: Android 14 → Android 17
## AAOS Audio Framework — Complete Lifecycle (API Level 34 → 37)
### `CarAudioService` | `AudioControl HAL` | `Volume Management` | `Audio Focus` | `Audio Routing` | `Audio Fade`

> **Compiled from:** `Audio_Framework_A14_to_A15.md` · `Audio_Framework_A15_to_A16.md` · `Audio_Framework_A16_to_A17.md`
>
> **Scope:** `platform/packages/services/Car/service/src/com/android/car/audio/`, `android.car.media`, `CarAudioService`, `AudioControl HAL`, Audio Focus, Volume Management, Audio Zones, Audio Routing, Audio Fade configurations.

---

## Executive Summary: Architectural Shifts in the AAOS Audio Framework (A14 → A17)

The Android 14-to-17 span represents the most consequential four-year period of architectural evolution in the AAOS audio stack. Five shifts stand out as foundational:

### Shift 1: The OEM Programmability Revolution (Android 14)
Android 14 was a watershed release for AAOS audio. A single release introduced: OEM Car Audio Plugin Services (allowing OEMs to intercept every focus decision), Dynamic Audio Zone Configurations (first-class runtime zone switching), Audio Mirroring (multi-zone audio broadcast), Passenger Audio Cast (cross-zone media routing), and the Configurable Audio Policy (CAP) Engine flags (`useCoreAudioVolume`, `useCoreAudioRouting`). For the first time, OEMs could replace the core audio interaction matrix with their own business logic without forking the Android audio stack. The `car_audio_configuration.xml` schema jumped to version 3.

### Shift 2: System-Enforced Audio Fade (Android 15)
Android 15 closed the compliance gap in audio focus enforcement. Prior to Android 15, an app could ignore a focus loss event and continue playing audio — there was no system-level enforcement. Android 15 introduced `FadeManagerConfiguration`, `car_audio_fade_configuration.xml`, and the `audioUseFadeManagerConfiguration` flag, giving `CarAudioService` the ability to apply VolumeShaper-based fade-out to non-compliant apps without their cooperation. Simultaneously, min/max activation volume (`audioUseMinMaxActivationVolume`) provided a first-ever mechanism to prevent dangerously loud or inaudibly quiet audio activation events. The schema advanced to version 4.

### Shift 3: CAP Engine Goes AIDL-Native (Android 16)
The Configurable Audio Policy engine's configuration mechanism, which had been read from static XML files on the vendor partition since its introduction, was migrated to live AIDL API calls in Android 16. The addition of `AudioHalCapConfiguration`, `AudioHalCapCriterionV2`, `AudioHalCapDomain`, `AudioHalCapParameter`, and `AudioHalCapRule` parcelables — delivered via `AudioHalEngineConfig.CapSpecificConfig` — eliminated the vendor partition XML dependency for new devices. This also introduced a **breaking naming convention change**: all product strategy names now require the `STRATEGY_` prefix. The parallel addition of fade/balance getter APIs (AAOS 25Q4) completed the read/write symmetry for those controls.

### Shift 4: Background Audio Hardening (Android 17)
Android 17 introduced the most disruptive behavioral change for AAOS app developers: system-enforced Foreground Service requirements for all background audio interactions. Audio APIs fail silently — `AudioTrack.write()`, `AAudioStream_write()`, `requestAudioFocus()`, and volume adjustment APIs — unless the app holds a visible Activity or an active non-`SHORT_SERVICE` Foreground Service. Apps targeting API 37 additionally require while-in-use (WIU) FGS capability. This impacts every AAOS app that plays audio with the screen off, starts audio on BOOT_COMPLETE, or manages background VoIP/navigation audio.

### Shift 5: USAGE_ASSISTANT Volume Independence (Android 17)
The new `AudioManager.MODE_ASSISTANT_CONVERSATION` mode and the isolation of `USAGE_ASSISTANT` into its own volume stream break the long-standing coupling between Assistant audio and media volume. This is directly relevant to AAOS OEMs building integrated voice assistant experiences, who must now map the new stream to an appropriate volume group in `car_audio_configuration.xml` and expose independent volume controls in their Settings UI.

---

## Part 1: Android 14 → Android 15
### AAOS 24Q3 | API Level 34 → 35

---

### 1.1 System-Enforced Audio Fade

**Configuration flag:** `audioUseFadeManagerConfiguration` (default: `false`)

**New Classes:**
- `FadeManagerConfiguration` — per-usage fade durations with allowlist/exclusion lists
- `FadeManagerConfiguration.Builder` — fluent API: `setFadeOutDurationForUsage()`, `setFadeInDurationForUsage()`, `setFadeableUsages()`, `setUnfadeableContentTypes()`, `setUnfadeableAudioAttributes()`
- `CarAudioFadeConfiguration` — runtime analog for OEM focus service integration

**New XML: `car_audio_fade_configuration.xml`**
```xml
<carAudioFadeConfiguration>
  <fadeConfiguration name="default_fade"
      defaultFadeOutDurationInMillis="500"
      defaultFadeInDurationInMillis="2000">
    <fadeableUsages>
      <usage value="AUDIO_USAGE_MEDIA"/>
    </fadeableUsages>
    <unfadeableContentTypes>
      <contentType value="AUDIO_CONTENT_TYPE_SPEECH"/>
    </unfadeableContentTypes>
    <fadeOutConfigurations>
      <fadeConfiguration usage="AUDIO_USAGE_MEDIA" durationInMillis="300"/>
    </fadeOutConfigurations>
    <fadeInConfigurations>
      <fadeConfiguration usage="AUDIO_USAGE_MEDIA" durationInMillis="1500"/>
    </fadeInConfigurations>
  </fadeConfiguration>
</carAudioFadeConfiguration>
```

**`car_audio_configuration.xml` → Version 4:**
```xml
<applyFadeConfigs>
  <fadeConfig name="default_fade" isDefault="true"/>
</applyFadeConfigs>
```

---

### 1.2 Min/Max Activation Volume

**Configuration flag:** `audioUseMinMaxActivationVolume` (default: `false`)

**New XML elements in `car_audio_configuration.xml` v4:**
```xml
<volumeGroup name="music_group">
  <activationVolumeConfig name="media_activation">
    <activationVolumeConfigEntry
        minActivationVolumePercentage="10"
        maxActivationVolumePercentage="80"
        invocationType="onPlaybackChanged"/>
  </activationVolumeConfig>
</volumeGroup>
```

**`invocationType` values:**

| Value | Trigger |
|-------|---------|
| `onBoot` | First audio activation after system boot |
| `onSourceChanged` | New app/UID starts playback |
| `onPlaybackChanged` | Every newly active playback event |

**Monitoring triggers:** `AudioPlaybackCallback` · Call state changes · HAL `IFocusListener#requestAudioFocus()`

---

### 1.3 OEM Focus Plugin — Per-Attribute Fade Configs

New builder method on `OemCarAudioFocusResult.Builder`:
```java
.setAudioAttributesToCarAudioFadeConfigurationMap(
    Map.of(mediaAttributes, myCarAudioFadeConfig))
```
Allows the OEM focus service to return per-attribute fade configurations alongside each focus decision.

---

### 1.4 Bluetooth Headset as Audio Output Device

- Users can designate a BT headset as primary Audio Output Device in AAOS Settings.
- Constraint: only a single media stream may be active over BT simultaneously.
- New Settings UI: `Settings > Sound & Vibration > Audio output`.

---

### 1.5 Configuration Flags — Android 15

| Flag | Default | Purpose |
|------|---------|---------|
| `audioUseFadeManagerConfiguration` | `false` | System-enforced fade on focus loss |
| `audioUseMinMaxActivationVolume` | `false` | Enforce min/max volume at activation |

---

### 1.6 Deprecated APIs (Android 14, still deprecated in A15)

| API | Deprecated In |
|-----|--------------|
| `CarAudioManager#isDynamicRoutingEnabled()` | Android 14 |
| `CarAudioManager#getExternalSources()` | Android 14 |
| `CarAudioManager#createAudioPatch(String, int, int)` | Android 14 |
| `CarAudioManager#releaseAudioPatch(CarAudioPatchHandle)` | Android 14 |

---

## Part 2: Android 15 → Android 16
### AAOS 25Q2 + 25Q4 | API Level 35 → 36

---

### 2.1 Configurable Audio Policy (CAP) Engine — AIDL Migration

**The most architecturally significant audio change in Android 16.**

**New AIDL parcelables** (namespace: `android.hardware.audio.core`):
- `AudioHalCapConfiguration` — top-level CAP configuration structure
- `AudioHalCapCriterionV2` — criterion definitions
- `AudioHalCapDomain` — parameter domain with routing rules
- `AudioHalCapParameter` — individual configurable parameters (paths + values)
- `AudioHalCapRule` — routing and volume decision rules
- `AudioHalEngineConfig.CapSpecificConfig` — integration point

**Mechanism:**
- **Before:** Audio policy service parsed XML from vendor partition directly.
- **After:** Audio policy service calls `getEngineConfig()` on AIDL Audio HAL; receives structured AIDL parcelables.

**Backward compatibility:** `EngineConfigXmlConverter.h` converts legacy vendor XML to AIDL parcelables.

#### Breaking Change: `STRATEGY_` Prefix

All product strategy names now require the `STRATEGY_` prefix:

| Old | New |
|-----|-----|
| `media` | `STRATEGY_MEDIA` |
| `phone` | `STRATEGY_PHONE` |
| `dtmf` | `STRATEGY_DTMF` |
| `sonification` | `STRATEGY_SONIFICATION` |
| `ring` | `STRATEGY_RING` |
| `accessibility` | `STRATEGY_ACCESSIBILITY` |
| `rerouting` | `STRATEGY_REROUTING` |
| `call_assistant` | `STRATEGY_CALL_ASSISTANT` |

**OEM extension slots** (unchanged): `vx_1000` through `vx_1039`

**HAL Compatibility Matrix:**

| System | Vendor | CAP Config Source |
|--------|--------|-------------------|
| Android 16 | Android 16 AIDL | System partition via `getEngineConfig()` |
| Android 16 | Android 15 AIDL | Vendor partition (XML fallback) |
| Android 16 | Android 14 HIDL | Vendor partition (XML fallback) |

---

### 2.2 AAudio OEM-Defined Audio Attribute Tags

- AAudio native library supports OEM-defined tags on audio streams.
- Tags integrate with `<oemContext>` elements in `car_audio_configuration.xml` v3+.
- When CAP is active, OEM context must match a CAP engine product strategy.
- Enables per-stream routing customization without app code changes.

---

### 2.3 AudioControl HAL — Status

- Remains at **AIDL v3** in publicly documented AOSP for Android 16.
- No AIDL v4 confirmed.
- Architectural changes in Android 16 affect the core Audio HAL (`android.hardware.audio.core`), not the automotive AudioControl HAL.

---

### 2.4 HD Radio Emergency Alert System (EAS) API (25Q2)

- New structured AAOS API for EAS data delivery to radio applications.
- Supports HD Radio (North America) and DAB EWS (European Union).
- Standardized `EasEvent` callback interfaces from radio HAL to OEM radio apps.

---

### 2.5 Fade and Balance Getter APIs (25Q4)

- New getter parity APIs for fade/balance (existing setters since Android 9):
  - `getFadeTowardFront()` — returns current fade value [-1.0 to +1.0]
  - `getBalanceTowardRight()` — returns current balance value [-1.0 to +1.0]
- Values **persist per user across ignition cycles**.

---

### 2.6 Alternative App Audio Controls (25Q4)

- Users can independently control communication app volume while driving.
- Separate audio routing path for communication apps alongside media.

---

### 2.7 `car_audio_configuration.xml` — Unchanged at Version 4

No new schema version in Android 16.

---

## Part 3: Android 16 → Android 17
### API Level 36 → 37 (Beta as of May 2026)

> AAOS-specific `CarAudioService` architecture changes and `AudioControl HAL` v4 have not been confirmed in public documentation as of research date.

---

### 3.1 Background Audio Hardening (Critical)

**Applies to all apps on Android 17 devices regardless of target SDK.**

**Requirements:**

| Tier | Scope | Requirement |
|------|-------|-------------|
| Tier 1 | All apps on Android 17 | Visible Activity OR FGS (not `SHORT_SERVICE`) |
| Tier 2 | Apps targeting API 37 | FGS with while-in-use capability (e.g., `mediaPlayback`) |

**Exception:** WIU waived if app holds `SCHEDULE_EXACT_ALARM` AND only uses `USAGE_ALARM` streams.

**Silent failures when requirements not met:**

| API | Failure |
|-----|---------|
| `AudioTrack.write()` | Silent stop |
| `AAudioStream_write()` | Silent stop |
| `AudioManager.requestAudioFocus()` | Returns `AUDIOFOCUS_REQUEST_FAILED` |
| `setStreamVolume()`, `adjustVolume()` | Silent failure |
| `setStreamMute()`, `setRingerMode()` | Silent failure |

**AAOS patterns most affected:**
- Background music with display off
- Navigation audio with nav app backgrounded
- Boot-triggered audio via `BOOT_COMPLETE`
- VoIP audio from background processes
- Alarm/reminder audio from background services

**Migration:** Adopt `MediaSessionService` (media3) or manually manage `mediaPlayback` FGS.

**Debug tools:**
```bash
adb shell cmd audio set-enable-hardening enable   # Enforce
adb shell cmd audio set-enable-hardening throw    # Enforce + exceptions
adb shell cmd audio set-enable-hardening disable  # Disable
adb logcat | grep AudioHardening                  # Monitor violations
```

---

### 3.2 Dedicated Assistant Volume Stream

- **New:** `AudioManager.MODE_ASSISTANT_CONVERSATION`
- `USAGE_ASSISTANT` audio isolated into its own volume stream, separate from `STREAM_MUSIC`.
- Independent mute/adjust for media vs. Assistant audio.
- **AAOS impact:** OEMs must map new stream to a volume group in `car_audio_configuration.xml` and expose independent volume controls in Audio Settings UI.

---

### 3.3 BLE Hearing Aid Device Type

- **New constant:** `AudioDeviceInfo.TYPE_BLE_HEARING_AID`
- Distinguishes BLE hearing aids from `TYPE_BLE_HEADSET`.
- System handles routing automatically; no app code required.

---

### 3.4 xHE-AAC Software Encoder

- **New encoder:** `c2.android.xheaac.encoder`
- xHE-AAC with mandatory loudness metadata.
- AAOS relevance: in-vehicle streaming and audio capture.

---

### 3.5 Confirmed Absence of AAOS-Specific API Changes

| Component | Status |
|-----------|--------|
| `CarAudioService` architecture | No confirmed changes |
| `AudioControl HAL` | AIDL v3 — no v4 confirmed |
| `car_audio_configuration.xml` schema | v4 — no v5 confirmed |
| Audio zone management APIs | No new APIs confirmed |
| Volume group management APIs | No new APIs confirmed |

---

## Consolidated Reference Tables

### AudioControl HAL Version History

| Version | Android | Key New Capabilities |
|---------|---------|---------------------|
| HIDL 1.0 | Android 9 | `setFadeTowardFront`, `setBalanceTowardRight` |
| HIDL 2.0 | Android 12 | Focus listener, HAL focus requests, muting, ducking |
| AIDL 1.0 | Android 12L | HIDL→AIDL migration (same APIs) |
| AIDL 2.0 | Android 13 | `onAudioFocusChangeWithMetaData`, `registerGainCallback` |
| AIDL 3.0 | Android 14 | `setModuleChangeCallback`, `IModuleChangeCallback#onAudioPortsChanged` |
| (none) | Android 15/16 | No new published AudioControl HAL version; CAP migrated to core Audio HAL |

### `car_audio_configuration.xml` Version History

| Version | Android | New Elements |
|---------|---------|-------------|
| v1 | Android 10 | Zones, volume groups |
| v2 | Android 11 | `audioZoneId`, `occupantZoneId` |
| v3 | Android 14 | `<oemContext>`, `<zoneConfigs>`, `<zoneConfig>` |
| v4 | Android 15 | `<activationVolumeConfig>`, `<applyFadeConfigs>`, `<fadeConfig>` |

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

### New `CarAudioManager` APIs by Android Version

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

### Deprecated APIs

| API | Deprecated In |
|-----|--------------|
| `CarAudioManager#isDynamicRoutingEnabled()` | Android 14 |
| `CarAudioManager#getExternalSources()` | Android 14 |
| `CarAudioManager#createAudioPatch(String, int, int)` | Android 14 |
| `CarAudioManager#releaseAudioPatch(CarAudioPatchHandle)` | Android 14 |

---

## Sources

- [AAOS 24Q3 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-24q3)
- [AAOS 25Q2 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q2)
- [AAOS 25Q4 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q4)
- [Audio Control HAL - AOSP](https://source.android.com/docs/automotive/audio/audio-control-hal)
- [Volume Management - AOSP](https://source.android.com/docs/automotive/audio/volume-management)
- [Audio Focus - AOSP](https://source.android.com/docs/automotive/audio/audio-focus)
- [Car Audio Plugin Service - AOSP](https://source.android.com/docs/automotive/audio/car-audio-plugin)
- [Configurable Audio Policy AIDL HAL - AOSP](https://source.android.com/docs/core/audio/aidl-cap)
- [Configurable Audio Policy Engine - AOSP](https://source.android.com/docs/automotive/audio/configurable-audio-policy-engine)
- [Car Audio Configuration - AOSP](https://source.android.com/docs/automotive/audio/audio-policy-configuration)
- [Audio Configuration Flags - AOSP](https://source.android.com/docs/automotive/audio/config-flags)
- [Background Audio Hardening - Android 17](https://developer.android.com/about/versions/17/changes/bg-audio)
- [Android 17 Features and APIs](https://developer.android.com/about/versions/17/features)
- [Android 17 Behavior Changes: All Apps](https://developer.android.com/about/versions/17/behavior-changes-all)
- [CarAudioManager API Reference](https://developer.android.com/reference/android/car/media/CarAudioManager)
