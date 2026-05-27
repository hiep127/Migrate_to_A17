# Full Android OS Changelog: Android 16 → Android 17
## API Level 36 → API Level 37

> **Data Sources:**
> - [Android 17 Features and APIs](https://developer.android.com/about/versions/17/features)
> - [Android 17 Behavior Changes: All Apps](https://developer.android.com/about/versions/17/behavior-changes-all)
> - [Android 17 Behavior Changes: Apps Targeting API 37](https://developer.android.com/about/versions/17/behavior-changes-17)
> - [Background Audio Hardening - Android 17](https://developer.android.com/about/versions/17/changes/bg-audio)
> - [Android 17 Beta Release Notes](https://developer.android.com/about/versions/17)
>
> **Note:** Android 17 (API Level 37) was in Beta as of May 2026. AAOS-specific documentation is limited. Changes to `CarAudioService` architecture and `AudioControl HAL` beyond those documented have not been confirmed. Extrapolated items are clearly labeled.

---

## 1. Platform & Core Framework

### 1.1 Health & Fitness
- New `HealthConnect` data types: Blood glucose trend, menstrual cycle temperature, food sensitivity records.
- Background read permissions for health apps tightened; must declare `ACTIVITY_RECOGNITION` for passive health monitoring.

### 1.2 Accessibility Enhancements
- `BrailleDisplay` API introduced for programmatic Braille display control without AccessibilityService.
- `AccessibilityService#takeScreenshot()` restricted; requires explicit accessibility feature declaration.
- Speech Recognition improvements: offline language packs auto-download via `RecognitionService`.

### 1.3 Text & Localization
- `Locale` API improvements: `Locale#getScript()` returns Unicode script code (e.g., `Latn`, `Arab`, `Hans`).
- Bidirectional text improvements in `TextView` for mixed LTR/RTL scripts.
- `CompactDecimalFormat` added to `android.icu.text` for short number formatting (e.g., "1.2K", "3M").

### 1.4 Privacy & Security
- **Minimum SDK bump:** Apps with `targetSdkVersion` below 28 are blocked at install on devices shipping with Android 17.
- **Partial Screen Sharing:** Screen capture APIs now support capturing individual app windows rather than full screen, limiting data exposure.
- **Credential Manager:** Passkey authentication flows updated for FIDO2 credential management improvements.

### 1.5 WindowManager & Display
- `WindowInsets` API improvements for better handling of status bar, navigation bar, and display cutouts.
- New `Display#getDisplayShape()` API for rounded corner region information.
- `OverlayDisplay` improvements for virtual display fidelity.

---

## 2. Audio Framework (Platform-Wide)

### 2.1 Background Audio Hardening (Major Behavioral Change — Affects ALL Apps)

> **Critical: This change applies to all apps running on Android 17 devices, regardless of target SDK.**

**Requirement:**
Apps that interact with audio APIs from the background must have either:
- A **visible Activity**, OR
- A **Foreground Service (FGS)** that is NOT `SHORT_SERVICE`

**Additional requirement (apps targeting API 37 only):**
- Background FGS must have **while-in-use (WIU) capabilities** (e.g., `android:foregroundServiceType="mediaPlayback"`)
- Exception: WIU waived if the app holds the `SCHEDULE_EXACT_ALARM` permission AND only interacts with `USAGE_ALARM` audio streams.

**APIs that fail silently when requirements not met:**

| API Category | Affected APIs | Failure Behavior |
|---|---|---|
| Audio Playback | `AudioTrack.write()`, `AAudioStream_write()`, OpenSL ES, ExoPlayer / media3 | Silent failure (audio stops, no exception thrown) |
| Audio Focus | `AudioManager.requestAudioFocus()` | Returns `AUDIOFOCUS_REQUEST_FAILED` |
| Volume & Ringer | `setStreamVolume()`, `setStreamMute()`, `adjustStreamVolume()`, `adjustVolume()`, `setRingerMode()` | Silent failure |

**AAOS / Automotive Impact:**
Highly impactful for AAOS-specific audio patterns including:
- Background music playback when display sleeps
- Navigation audio playback when navigation app not in foreground
- Boot-triggered audio services using `BOOT_COMPLETE` intent
- VoIP / communication audio from background processes
- Alarm/reminder playback from background services

**Recommended Migration:**
- Adopt `androidx.media3:media3-session` `MediaSessionService` (manages lifecycle automatically)
- Or manually manage `mediaPlayback` FGS:

```xml
<!-- AndroidManifest.xml -->
<service
    android:name=".AudioPlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false"/>
```

```java
// Start FGS before background audio
Intent intent = new Intent(context, AudioPlaybackService.class);
ContextCompat.startForegroundService(context, intent);
```

**Testing / Debugging Tools:**
```bash
# Enable hardening enforcement
adb shell cmd audio set-enable-hardening enable

# Enable enforcement with exceptions thrown (useful for detecting violations in tests)
adb shell cmd audio set-enable-hardening throw

# Disable enforcement (for testing legacy behavior)
adb shell cmd audio set-enable-hardening disable

# Check for violations
adb dumpsys audio | grep -i hardening

# Monitor violations in real time
adb logcat | grep AudioHardening
```

**Logcat violation levels:**
- `level: full` — App has an FGS but it lacks WIU capabilities
- `level: partial` — App has no FGS at all

### 2.2 Dedicated Assistant Volume Stream

**New audio mode constant:** `AudioManager.MODE_ASSISTANT_CONVERSATION`
- Signals to the system that an active voice assistant session is underway.
- Ensures `USAGE_ASSISTANT` audio streams are controllable even when no `USAGE_ASSISTANT` audio is actively playing.

**Volume isolation:**
- `USAGE_ASSISTANT` audio now routes to an **isolated volume stream** separate from `STREAM_MUSIC`.
- Users can:
  - Mute media independently while Assistant responses remain audible.
  - Silence Assistant without affecting media playback.
  - Control each stream from Quick Settings volume panel independently.
- Consistent behavior when audio is routed to Bluetooth peripherals.

**AAOS Relevance:**
- In-vehicle voice assistants (Google Assistant, OEM voice assistants) using `USAGE_ASSISTANT` now get independent volume control.
- Aligns with AAOS audio zone volume group management — OEMs may need to expose this new stream in their system audio settings UI.
- CarAudioService may need to map the new Assistant volume stream to an appropriate volume group in `car_audio_configuration.xml`.

### 2.3 BLE Hearing Aid Audio Device Type

**New constant:** `AudioDeviceInfo.TYPE_BLE_HEARING_AID`
- Distinguishes BLE hearing aids from generic LE Audio headsets (`TYPE_BLE_HEADSET`).
- Enables apps to provide tailored UI icons (e.g., hearing aid icon instead of generic headset icon) and device-specific routing behavior.
- System-level routing behavior changes automatically; no app code change required for basic routing.

### 2.4 xHE-AAC Software Encoder

**New system encoder:** `c2.android.xheaac.encoder`
- Extended High Efficiency AAC (xHE-AAC / MPEG-D Unified Speech and Audio Coding) software encoder.
- Supports both **high bitrate** (music quality) and **low bitrate** (speech) encoding in a single codec.
- Mandatory loudness metadata support to ensure consistent perceived volume across encode sessions.

**AAOS Relevance:**
- Useful for in-vehicle streaming audio, voice call recording, and OEM audio capture scenarios.
- Loudness metadata support ensures consistent perceived levels across ignition cycles and audio sources.

---

## 3. Android Automotive OS — Android 17

> **Note:** AAOS-specific release notes for Android 17 (equivalent to AAOS 26Q1/26Q2) have not been fully published as of this document's research date. The following reflects confirmed behavior changes plus architectural extrapolations based on the trajectory of the AAOS audio stack.

### 3.1 Audio Framework (AAOS-Specific)

- **Background Audio Hardening (Section 2.1)** is the primary audio behavior change affecting all AAOS apps.
- **Assistant Volume Stream (Section 2.2)** is relevant to in-vehicle voice assistant integration.
- **Confirmed absence of new AAOS-specific APIs** in the following areas (based on available documentation):
  - `CarAudioService` internal architecture — no confirmed changes
  - `AudioControl HAL` — remains at AIDL v3; no v4 confirmed in public AOSP documentation
  - `car_audio_configuration.xml` — no version 5 schema confirmed
  - Audio zone management (`CarAudioZoneConfigInfo`, `switchAudioZoneToConfig`) — no new APIs confirmed
  - Volume group management APIs — no new `CarAudioManager` system APIs confirmed beyond Android 16

### 3.2 *(Extrapolated)* Likely AAOS Audio Stack Evolution

> The following are informed extrapolations based on AOSP master branch trajectory. They are NOT confirmed in published release notes.

- **CAP Engine stability:** The Android 16 CAP AIDL migration is likely to see stabilization and broader OEM adoption in the Android 17 timeframe.
- **AudioControl HAL v4:** AIDL v4 may introduce support for the new `MODE_ASSISTANT_CONVERSATION` audio mode at the HAL level, enabling OEM amplifiers/DSPs to respond to Assistant activation.
- **Background hardening enforcement in AAOS:** OEMs will need to audit preloaded apps and system services that start audio on `BOOT_COMPLETE` or from background system services.

---

## 4. Connectivity

### 4.1 Wi-Fi
- Wi-Fi 7 (802.11be) Multi-Link Operation (MLO) APIs finalized.
- `WifiNetworkSpecifier` updated to prefer MLO links when available.

### 4.2 Bluetooth
- LE Audio Broadcast: `BroadcastAssistant` API supports multi-source broadcast aggregation.
- HFP 1.9 negotiation support for improved voice call codec quality.

### 4.3 NFC
- `NfcAdapter#enableReaderMode()` improvements for faster tag detection.
- Background NFC polling restrictions tightened for privacy.

---

## 5. Android Runtime & Toolchain

### 5.1 ART & Java
- Virtual Threads (Project Loom): `Thread.ofVirtual()` available for apps targeting API 37 on supported devices.
- `java.lang.foreign` (Panama): Initial preview of native memory access API.
- GC improvements: Reduced pause times for the Concurrent Mark Compact collector.

---

## 6. API Level 37 Behavior Changes Summary

| Change | Scope | Impact |
|--------|-------|--------|
| Background audio hardening | All apps on Android 17 | High — audio stops silently without FGS |
| WIU requirement for audio focus | Apps targeting API 37 | High — `requestAudioFocus()` fails without WIU FGS |
| `TYPE_BLE_HEARING_AID` device type | All apps | Low — informational, system handles routing |
| `MODE_ASSISTANT_CONVERSATION` | Apps using `USAGE_ASSISTANT` | Medium — independent volume stream |
| xHE-AAC encoder available | Apps using `MediaCodec` | Low — new encode option |

---

## 7. Sources

- [Android 17 Features & APIs](https://developer.android.com/about/versions/17/features)
- [Background Audio Hardening - Android 17](https://developer.android.com/about/versions/17/changes/bg-audio)
- [Android 17 Behavior Changes: All Apps](https://developer.android.com/about/versions/17/behavior-changes-all)
- [Android 17 Beta Release Notes](https://developer.android.com/about/versions/17)
- [AAOS 25Q4 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q4)
- [CarAudioManager API Reference](https://developer.android.com/reference/android/car/media/CarAudioManager)
