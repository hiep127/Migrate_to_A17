# Full Android OS Changelog: Android 15 → Android 16                                
## API Level 35 → API Level 36

> **Data Sources:**
> - [Android 16 for Developers](https://developer.android.com/about/versions/16)
> - [AAOS 25Q2 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q2)
> - [AAOS 25Q4 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q4)
> - [Android 16 Release Notes - AOSP](https://source.android.com/docs/whatsnew/android-16-release)
> - [Configurable Audio Policy support in AIDL HAL - AOSP](https://source.android.com/docs/core/audio/aidl-cap)

---

## 1. Platform & Core Framework

### 1.1 Adaptive Refresh Rate (ARR)
- Android 16 introduces `DisplayManager#getSupportedModes()` returning display modes with variable refresh rates.
- `Surface#setFrameRate()` API now respects underlying display capability; system automatically selects best refresh rate.
- Reduced power consumption on OLED and high-refresh-rate panels during low-motion content.

### 1.2 Predictive Back — Full Enforcement
- Apps targeting API 36 that do not opt into predictive back gesture receive a default system-provided animation.
- Legacy `KEYCODE_BACK` dispatch removed for apps targeting API 36+; must use `OnBackPressedDispatcher`.
- `BackHandler` Compose API stable.

### 1.3 Accessibility
- `AccessibilityManager#performGlobalAction()` restricted for non-user-initiated actions (security hardening).
- `UiAutomation` API updates for better compatibility with accessibility service permissions.
- Text classification improvements for multi-language accuracy.

### 1.4 Privacy & Permissions
- **Non-dismissible notifications:** Android 16 restricts foreground service (FGS) notifications — user cannot dismiss them during active FGS lifecycle.
- **Background start restrictions tightened:** Stricter `SCHEDULE_EXACT_ALARM` usage; apps must justify each exact alarm use case.
- **Health permissions:** Additional granular permissions under `HealthConnect` for new data types (blood glucose, body temperature).

### 1.5 Camera
- `CameraExtensionSession` improvements for bokeh and night mode quality.
- `ZoomRatioRange` API now returns manufacturer-calibrated optical zoom steps.
- Camera2 simultaneous streams improvements for multi-camera setups.

### 1.6 Connectivity
- **Wi-Fi 7 (802.11be) APIs:** `WifiManager#getWifiState()` returns new `WIFI_STATE_7` constant; MLO (Multi-Link Operation) information exposed via `WifiInfo`.
- **Thread Network (Matter):** `ThreadNetworkController` API stable for commissioning Thread devices.
- **UWB:** Ultra-wideband ranging APIs updated; support for `FiraRangingSession` improvements.

### 1.7 WindowManager & Multi-Display
- `DisplayArea` API improvements for large-screen layout management.
- `TaskFragmentOrganizer` updates for embedding tasks across display areas.

---

## 2. Android Automotive OS (AAOS) — Releases: 25Q2 and 25Q4

### 2.1 Audio Framework

#### 2.1.1 Configurable Audio Policy (CAP) Engine — AIDL Migration (Major Architecture Change)

**Background:**
The Configurable Audio Policy (CAP) engine allows OEMs to define audio routing and volume rules via configuration rather than code. Prior to Android 16, CAP configuration was read from XML files on the vendor partition, and AIDL Audio HAL did not support CAP.

**Changes in Android 16:**
- Full CAP support added to the AIDL Audio HAL path via `AudioHalCapConfiguration.aidl`.
- Audio policy service now queries CAP config via `getEngineConfig()` AIDL call instead of parsing vendor partition XML directly.
- Parameter Framework XML files relocate from vendor partition to system partition.

**New AIDL Parcelables (namespace: `android.hardware.audio.core`):**
- `AudioHalCapConfiguration` — top-level CAP configuration structure
- `AudioHalCapCriterionV2` — criterion definitions (e.g., `available_output_devices`)
- `AudioHalCapDomain` — parameter domain container
- `AudioHalCapParameter` — individual configurable parameters
- `AudioHalCapRule` — routing and volume decision rules
- Integration point: `AudioHalEngineConfig.CapSpecificConfig` aggregates all above structures

**Naming Convention Change (Breaking for migration):**
- Product strategy names now require `STRATEGY_` prefix:
  - `media` → `STRATEGY_MEDIA`
  - `phone` → `STRATEGY_PHONE`
  - `dtmf` → `STRATEGY_DTMF`
  - `sonification` → `STRATEGY_SONIFICATION`
  - `ring` → `STRATEGY_RING`
- Parameter paths updated accordingly:
  - Before: `/Policy/policy/product_strategies/media/device_address`
  - After: `/Policy/policy/product_strategies/STRATEGY_MEDIA/device_address`
- OEM extension slots: `vx_1000` through `vx_1039` (40 slots for OEM-defined strategies)

**Migration Compatibility Matrix:**

| System Partition | Vendor Partition | CAP Config Source |
|-----------------|-----------------|-------------------|
| Android 16 | Android 16 AIDL HAL | System partition via AIDL `getEngineConfig()` |
| Android 16 | Android 15 AIDL HAL | Vendor partition (legacy XML fallback) |
| Android 16 | Android 14 HIDL HAL | Vendor partition (legacy XML fallback) |

**Helper tool:** `EngineConfigXmlConverter.h` converts legacy XML to AIDL parcelables for backward compatibility.

**Reference implementation:** `device/google/cuttlefish/shared/auto/audio/policy/engine/` (Cuttlefish `aosp_cf_x86_64_auto` target).

#### 2.1.2 AAudio OEM-Defined Audio Attributes Tags

- AAudio native library now supports OEM-defined audio attribute tags on audio streams.
- OEM-defined tags attach custom metadata to streams used by CarAudioService and CAP engine for routing/volume decisions.
- Integration with `car_audio_configuration.xml` v3+ `<oemContext>` elements.
- When CAP is active, OEM-defined context must match a CAP engine product strategy.

#### 2.1.3 AudioControl HAL — API-Based Configuration

- Audio feature configuration transitions from static XML vendor partition files to live AIDL API calls.
- Consistent with the CAP AIDL migration described in 2.1.1.
- AudioControl HAL interface remains documented at AIDL v3 in public AOSP; architectural integration with audio policy configuration layer changes significantly.

#### 2.1.4 HD Radio Emergency Alert System (EAS) API (25Q2)

- New AAOS API for passing Emergency Alert System (EAS) data to radio applications.
- Supports:
  - **HD Radio** (North America EAS standard)
  - **DAB EWS** (Digital Audio Broadcasting Emergency Warning System — European Union)
- Standardizes emergency alert data flow from radio HAL/stack to automotive radio applications.
- Radio apps receive structured EAS metadata via new callback interfaces.

#### 2.1.5 Fade and Balance Getter APIs (25Q4)

- Getter APIs for fade and balance added, providing parity with existing AudioControl HAL setters:
  - `setFadeTowardFront(float value)` (existing) ↔ new getter
  - `setBalanceTowardRight(float value)` (existing) ↔ new getter
- Fade/balance values **persist per user across ignition cycles**.
- Enables system audio settings UI to display current fade and balance state without re-deriving it from HAL state.

#### 2.1.6 Alternative App Audio Controls (25Q4)

- Users can now adjust volume for non-media-center communication apps while driving.
- Addresses use cases: phone call volume, VoIP app audio, while a separate media source is active.
- New system audio routing path for communication apps running alongside media.

### 2.2 CarAppLibrary & Navigation
- Map rendering templates performance improvements.
- `RoutePreviewNavigationTemplate` added for pre-navigation route previews.
- Cluster display APIs updated to support dual-screen cluster rendering.

### 2.3 Connectivity (AAOS)
- **Projection (Android Auto):** Performance improvements for video streaming over Wi-Fi projection.
- **Bluetooth Audio:** LE Audio broadcast improvements for in-vehicle broadcast scenarios.

### 2.4 Vehicle Hardware Abstraction
- `VehicleHal` improvements for faster signal propagation to VHAL clients.
- New `VehicleProperty` constants added for ADAS (Advanced Driver Assistance Systems) state reporting.

---

## 3. Android Runtime & Toolchain

### 3.1 ART & Java Platform
- OpenJDK 21 LTS alignment: `java.util.concurrent` virtual thread support (Project Loom subset).
- `MethodHandle` invocation optimized in ART interpreter.
- `java.lang.ref.Cleaner` promoted as preferred finalization mechanism over `finalize()`.

### 3.2 Build System
- Soong build system improvements for incremental compilation.
- `.bp` to `.bazel` migration tooling updates.

---

## 4. New Configuration Flags (AAOS Audio — Android 16)

*(No new AAOS audio feature flags confirmed in Android 16 beyond those in Android 15; the CAP AIDL migration is a structural change, not a feature flag.)*

---

## 5. Key Architectural Changes Summary

| Component | Before (Android 15) | After (Android 16) |
|-----------|--------------------|--------------------|
| CAP Engine config source | Vendor partition XML | System partition via AIDL HAL `getEngineConfig()` |
| Product strategy naming | Plain name (e.g., `media`) | `STRATEGY_` prefix (e.g., `STRATEGY_MEDIA`) |
| AAudio OEM tags | Not supported | Supported; integrates with `<oemContext>` |
| Fade/balance query | Setter only | Getter + setter parity; persists per user |
| EAS radio data | No standardized API | Structured EAS API for HD Radio & DAB |

---

## 6. Sources

- [Android 16 Highlights](https://developer.android.com/about/versions/16)
- [AAOS 25Q2 Release](https://source.android.com/docs/automotive/start/releases/aaos-25q2)
- [AAOS 25Q4 Release](https://source.android.com/docs/automotive/start/releases/aaos-25q4)
- [Configurable Audio Policy AIDL HAL](https://source.android.com/docs/core/audio/aidl-cap)
- [Configurable Audio Policy Engine](https://source.android.com/docs/automotive/audio/configurable-audio-policy-engine)
- [Audio Control HAL](https://source.android.com/docs/automotive/audio/audio-control-hal)
- [Android 16 Release Notes - AOSP](https://source.android.com/docs/whatsnew/android-16-release)
