# Full Android OS Changelog: Android 14 → Android 15
## API Level 34 → API Level 35

> **Data Sources:**
> - [Android 15 for Developers](https://developer.android.com/about/versions/15)
> - [AAOS 24Q3 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-24q3)
> - [Android 15 Behavior Changes (all apps)](https://developer.android.com/about/versions/15/behavior-changes-all)
> - [CarAudioManager API Reference](https://developer.android.com/reference/android/car/media/CarAudioManager)
> - AOSP `platform/packages/services/Car` commit history (branch `android15-release`)

---

## 1. Platform & Core Framework

### 1.1 Predictive Back Animations (Enforcement)
- Predictive back gesture animations that were opt-in in Android 14 are now more broadly enabled.
- Apps must handle `OnBackPressedCallback` or risk undefined back-navigation behavior.
- New `Activity#finishAfterTransition()` integrations for smoother exit animations.

### 1.2 Large Screen & Foldable Improvements
- `ActivityEmbedding` API stable; multi-pane layouts now first-class on large-screen form factors.
- `WindowMetrics` API improvements for accurately querying display bounds across configurations.
- New `SurfaceView` performance improvements reducing jank during transitions.

### 1.3 Health Connect Platform Integration
- Health Connect promoted from APK to platform service (no longer requires a separate app install).
- New `HealthConnectManager` system service handles permission grants, data access, and aggregation.
- `READ_HEALTH_DATA_HISTORY` and `READ_HEALTH_DATA_IN_BACKGROUND` permissions added.

### 1.4 Photo Picker & Media Permissions
- `READ_MEDIA_VISUAL_USER_SELECTED` permission introduced for partial photo/video access.
- Apps targeting API 35 must handle the selective media access grant flow.
- `MediaStore` query results scoped based on grant level.

### 1.5 Privacy & Security
- **Intent Redirect Security:** Android 15 blocks apps from launching implicit intents directed at private components in other apps without explicit allowlisting.
- **Minimum SDK enforcement:** Sideloaded apps with `targetSdkVersion` below a system-defined threshold are blocked at install time; configurable per OEM.
- **KeyStore/StrongBox:** Hardware-backed key attestation for user-presence proofs during sensitive operations.
- **Credential Manager:** `CredentialManager` API for passkey support stable; replaces legacy `AccountAuthenticator` flow for passkeys.

### 1.6 OpenType Variable Fonts
- Full Variable Font support in `Paint` and `Typeface` APIs.
- `FontVariationAxis` updates for custom weight/width/italic axes.
- `ResourcesCompat` updated to support variable font loading from assets.

### 1.7 Bluetooth Improvements
- LE Audio broadcast (`BroadcastAssistant` API) stable for multi-stream audio sources.
- `BluetoothAdapter#getProfileConnectionState()` now returns `STATE_TURNING_ON` / `STATE_TURNING_OFF` transient states.
- Improved bonding latency for Bluetooth Classic over HFP.

---

## 2. Android Automotive OS (AAOS) — Release: 24Q3

### 2.1 Audio Framework

#### 2.1.1 System-Enforced Audio Fade (New Feature — `audioUseFadeManagerConfiguration`)
- CarAudioService can now automatically fade-out audio losing focus without app cooperation.
- **New class:** `FadeManagerConfiguration` with `Builder` API for per-usage fade durations.
- **New class:** `CarAudioFadeConfiguration` for runtime OEM focus service integration.
- **New configuration flag:** `audioUseFadeManagerConfiguration` (default: `false`).
- **New XML:** `car_audio_fade_configuration.xml` with `<carAudioFadeConfiguration>` root element.
- `car_audio_configuration.xml` updated to **version 4** with `<applyFadeConfigs>` and `<fadeConfig>` elements.
- `OemCarAudioFocusResult.Builder` gains `setAudioAttributesToCarAudioFadeConfigurationMap()` method.

#### 2.1.2 Min/Max Activation Volume (New Feature — `audioUseMinMaxActivationVolume`)
- Enforces minimum and maximum volume bounds when audio activates on a volume group.
- **New flag:** `audioUseMinMaxActivationVolume` (default: `false`).
- **New XML:** `<activationVolumeConfig>` and `<activationVolumeConfigEntry>` elements in `car_audio_configuration.xml` v4.
- `invocationType` values: `onBoot`, `onSourceChanged`, `onPlaybackChanged`.
- CarAudioService monitors `AudioPlaybackCallback`, call state, and HAL focus requests to apply constraints.

#### 2.1.3 Bluetooth Headset as Audio Output Device
- Users may designate a connected Bluetooth headset as a primary audio output device in AAOS Settings.
- Constraint: only a single media stream may be routed over BT simultaneously.
- New Settings UI entry point for audio output device designation.

#### 2.1.4 OEM Focus Service Enhancements
- `OemCarAudioFocusResult` builder extended to return per-attribute `CarAudioFadeConfiguration` maps.
- Allows the OEM focus service to dynamically dictate fade behavior at the time of each focus evaluation.

### 2.2 Connectivity (AAOS)
- **Wi-Fi Direct:** Improved peer discovery and connection stability for AAOS head unit ↔ mobile app communication.
- **Telephony/Emergency Calling:** Improved eCall (Emergency Call) stability over IMS networks.

### 2.3 OTA (Over-The-Air) Updates
- **Virtual A/B (VAB) compression:** Reduced OTA package size via better ZSTD delta compression.
- **Snapshot Merge:** Snapshot merge performance improvements; reduced post-reboot merge time.

### 2.4 Input & Controls
- **Rotary Controller API updates:** New `FocusArea` and `FocusParkingView` improvements in `car-ui-lib`.
- Steering wheel key injection now uses `KeyCharacterMap.VIRTUAL_KEYBOARD` as the device by default.
- New `RotaryInjector` configuration options for dial sensitivity.

### 2.5 App Distribution & CarAppLibrary
- `CarAppLibrary` updated to support map rendering templates.
- `ConstraintTemplate` additions for forms with constraints.
- Navigation session resume improvements: host no longer destroys navigation session on Home press if `NavigationManager.onStopNavigation()` was not called.

---

## 3. Android Runtime & Toolchain

### 3.1 ART Improvements
- OpenJDK 21 core library APIs backported (streams, records, sealed classes for use in code targeting API 35).
- `java.lang.invoke` performance improvements for `MethodHandle` and `VarHandle`.
- Improved GC throughput with the Concurrent Mark Compact (CMC) garbage collector.

### 3.2 Kotlin & Language Support
- `kotlinx.coroutines` 1.8+ aligned structured concurrency with Android lifecycle aware APIs.
- `androidx.lifecycle:lifecycle-runtime-ktx` `repeatOnLifecycle` block improvements.

---

## 4. car_audio_configuration.xml — Version 4 Schema Summary

```xml
<!-- New in version 4: activationVolumeConfig and applyFadeConfigs -->
<carAudioConfiguration version="4">
  <zones>
    <zone name="primary zone" isPrimary="true">
      <zoneConfigs>
        <zoneConfig name="default" isDefault="true">
          <volumeGroups>
            <group name="music_group">
              <!-- NEW: Activation volume constraints -->
              <activationVolumeConfig name="media_activation">
                <activationVolumeConfigEntry
                    minActivationVolumePercentage="10"
                    maxActivationVolumePercentage="80"
                    invocationType="onPlaybackChanged"/>
              </activationVolumeConfig>
            </group>
          </volumeGroups>
          <!-- NEW: Fade configuration references -->
          <applyFadeConfigs>
            <fadeConfig name="default_fade" isDefault="true"/>
          </applyFadeConfigs>
        </zoneConfig>
      </zoneConfigs>
    </zone>
  </zones>
</carAudioConfiguration>
```

---

## 5. New Configuration Flags Introduced in Android 15 (AAOS)

| Flag Name | Default | Purpose |
|-----------|---------|---------|
| `audioUseMinMaxActivationVolume` | `false` | Enforce min/max volume bounds at audio activation |
| `audioUseFadeManagerConfiguration` | `false` | Enable system-enforced fade on audio focus loss |

---

## 6. Key API Additions

| API | Package | Type |
|-----|---------|------|
| `FadeManagerConfiguration` | `android.car.media` | New class |
| `FadeManagerConfiguration.Builder` | `android.car.media` | New builder |
| `CarAudioFadeConfiguration` | `android.car.media` | New class |
| `OemCarAudioFocusResult.Builder#setAudioAttributesToCarAudioFadeConfigurationMap()` | `android.car.oem` | New method |

---

## 7. Sources

- [Android 15 Highlights](https://developer.android.com/about/versions/15)
- [AAOS 24Q3 Release](https://source.android.com/docs/automotive/start/releases/aaos-24q3)
- [Audio focus - AOSP](https://source.android.com/docs/automotive/audio/audio-focus)
- [Volume management - AOSP](https://source.android.com/docs/automotive/audio/volume-management)
- [Car audio plugin service - AOSP](https://source.android.com/docs/automotive/audio/car-audio-plugin)
- [Audio configuration flags - AOSP](https://source.android.com/docs/automotive/audio/config-flags)
