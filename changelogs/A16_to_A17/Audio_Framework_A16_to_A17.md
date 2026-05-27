# Audio Framework Changelog: Android 16 → Android 17
## AAOS Audio Framework — Layered Architectural Analysis
### API Level 36 → 37 | Beta as of May 2026

> **Source basis:** Official Android 17 developer documentation and AAOS 25Q4 release notes.
> Android 17 (API Level 37) was in Beta as of the research date. AAOS-specific `CarAudioService`
> architecture changes and `AudioControl HAL` v4 **have not been confirmed** in public AOSP
> documentation. Platform-wide audio behavioral changes that directly affect AAOS apps are documented
> in full below.
>
> No local AOSP git repository was available on this machine. Git commands for patch generation are
> provided in the appendix.

---

## CarAudioService Updates
> **Layer:** `platform/packages/services/Car/service/src/com/android/car/audio/` | `android.car.media`

### Background Audio Hardening (Critical — Applies to All Apps)

> **This is the most impactful audio framework change in Android 17 for AAOS apps.**
>
> This change applies to **all apps running on Android 17 devices** regardless of their target SDK version.

**Problem solved:** Background audio interactions from apps without a visible UI or foreground service could be abused for audio spying, unexpected audio hijacking, and volume manipulation.

**Solution:** `CarAudioService` (and the platform audio stack) enforces that any app interacting with audio APIs from the background must hold a visible Activity or an active FGS.

#### Enforcement Tiers

| Tier | Scope | Requirement |
|------|-------|-------------|
| Tier 1 | All apps on Android 17 devices | Visible Activity OR FGS (not of type `SHORT_SERVICE`) |
| Tier 2 | Apps targeting API Level 37 only | FGS must have while-in-use (WIU) capability (e.g., `mediaPlayback`) |

**Exception to Tier 2:** WIU requirement is waived when:
1. App holds `SCHEDULE_EXACT_ALARM` permission, AND
2. App only interacts with `AudioManager.USAGE_ALARM` streams.

#### Affected Audio APIs and Failure Behavior

| API Category | Method | Failure When Requirements Not Met |
|---|---|---|
| Audio Playback | `AudioTrack.write()` | Silent stop — no exception thrown |
| Audio Playback | `AAudioStream_write()` (native) | Silent stop |
| Audio Playback | OpenSL ES write operations | Silent stop |
| Audio Playback | `ExoPlayer` / media3 | Silent stop (internally uses above) |
| Audio Focus | `AudioManager.requestAudioFocus()` | Returns `AUDIOFOCUS_REQUEST_FAILED` |
| Volume | `AudioManager.setStreamVolume()` | Silent failure |
| Volume | `AudioManager.adjustStreamVolume()` | Silent failure |
| Volume | `AudioManager.adjustVolume()` | Silent failure |
| Mute | `AudioManager.setStreamMute()` | Silent failure |
| Ringer | `AudioManager.setRingerMode()` | Silent failure |

#### AAOS-Specific Patterns Affected

| AAOS Pattern | Impact |
|---|---|
| Background music when display sleeps | Audio stops unless `mediaPlayback` FGS is active |
| Navigation audio with nav app backgrounded | Audio stops unless nav app holds `navigation` FGS type |
| Boot-triggered audio via `BOOT_COMPLETE` receiver | Service must start FGS before any audio interaction |
| VoIP / communication audio from background process | Must use `phoneCall` FGS type |
| Alarm / reminder audio from background service | `SCHEDULE_EXACT_ALARM` exemption applies for `USAGE_ALARM` only |
| HAL-initiated focus requests (`IFocusListener`) | Requester process must hold appropriate FGS |

#### Required Manifest Declaration (Apps Targeting API 37)

```xml
<!-- AndroidManifest.xml -->
<service
    android:name=".AudioPlaybackService"
    android:foregroundServiceType="mediaPlayback"
    android:exported="false"/>
```

#### Recommended Migration Path for AAOS Apps

**Option A — Preferred:** Adopt `androidx.media3:media3-session` `MediaSessionService`:

```java
public class CarMediaService extends MediaSessionService {
    @Override
    public MediaSession onGetSession(MediaSession.ControllerInfo controllerInfo) {
        return mediaSession;
    }
    // MediaSessionService manages FGS lifecycle automatically
}
```

**Option B:** Manually manage a `mediaPlayback` FGS:

```java
// Start FGS before any background audio operation
public void startAudio() {
    Intent intent = new Intent(context, AudioPlaybackService.class);
    ContextCompat.startForegroundService(context, intent);
}

// In AudioPlaybackService.onStartCommand()
@Override
public int onStartCommand(Intent intent, int flags, int startId) {
    Notification notification = buildAudioNotification();
    ServiceCompat.startForeground(this, NOTIFICATION_ID, notification,
        ServiceInfo.FOREGROUND_SERVICE_TYPE_MEDIA_PLAYBACK);
    return START_STICKY;
}
```

#### Debug and Testing Tools

```bash
# Enforce background audio hardening
adb shell cmd audio set-enable-hardening enable

# Enforce and throw exceptions on violations (useful in integration tests)
adb shell cmd audio set-enable-hardening throw

# Disable enforcement (restore legacy behavior for debugging)
adb shell cmd audio set-enable-hardening disable

# Audit active hardening violations
adb dumpsys audio | grep -A 5 "hardening"

# Real-time monitoring of violations
adb logcat | grep AudioHardening
```

**Logcat violation severity levels:**

| Level | Meaning |
|-------|---------|
| `level: partial` | App has no FGS at all (Tier 1 violation) |
| `level: full` | App has an FGS but lacks while-in-use capability (Tier 2 violation) |

---

### Dedicated Assistant Volume Stream

**Problem solved:** `USAGE_ASSISTANT` audio shared `STREAM_MUSIC` volume. Muting media also silenced the voice assistant; adjusting media volume affected assistant response volume.

#### New Constant: `AudioManager.MODE_ASSISTANT_CONVERSATION`

```java
// Signal that an active voice assistant conversation is underway
audioManager.setMode(AudioManager.MODE_ASSISTANT_CONVERSATION);

// When conversation ends
audioManager.setMode(AudioManager.MODE_NORMAL);
```

- Hints to the system audio policy that `USAGE_ASSISTANT` should be independently controllable even between TTS utterances during a multi-turn conversation.

#### Behavior Change

| Aspect | Android 16 | Android 17 |
|--------|-----------|-----------|
| `USAGE_ASSISTANT` volume stream | Shared with `STREAM_MUSIC` | Isolated independent volume stream |
| Media mute affects assistant | Yes | No — streams are independent |
| Volume panel | Single media slider | Separate assistant slider |
| BT routing | Combined with media | Both streams route separately to BT peripherals |

#### AAOS OEM Action Required

OEMs building integrated voice assistant experiences must:

1. Map the new `USAGE_ASSISTANT` stream to an appropriate **volume group** in `car_audio_configuration.xml`:

```xml
<!-- OEM may need to create or update a volume group for USAGE_ASSISTANT -->
<group name="assistant_group">
  <context>
    <usage value="AUDIO_USAGE_ASSISTANT"/>
  </context>
</group>
```

2. Expose an independent volume control for the Assistant stream in the system Audio Settings UI.
3. Verify the in-vehicle DSP/amplifier can handle separate gain levels for media vs. Assistant audio paths.

---

### Confirmed Absence of AAOS-Specific CarAudioService API Changes

Based on available Android 17 documentation, the following `CarAudioService` components show **no confirmed changes**:

| Component | Status |
|-----------|--------|
| `CarAudioService` internal architecture | No confirmed changes |
| `car_audio_configuration.xml` schema | v4 — no v5 confirmed |
| Audio zone management (`CarAudioZoneConfigInfo`) | No new APIs confirmed |
| `switchAudioZoneToConfig()` behavior | No changes confirmed |
| Volume group management (`CarVolumeGroupInfo`) | No new APIs confirmed |
| OEM Car Audio Plugin Services | No new interfaces confirmed |
| `car_audio_fade_configuration.xml` | No changes confirmed |

---

## AudioControlService Updates
> **Layer:** C++ proxy service (`cpp/audiocontrol/`) translating Java `CarAudioService` commands via AIDL

No changes to the `AudioControlService` C++ proxy layer in Android 17. The AIDL v3 `IAudioControl` interface remains in use without modification.

Background audio hardening enforcement occurs at the platform audio framework level (Java `AudioManager` + `AudioTrack` enforcement in `audioserver`) and does not require any AIDL changes to the automotive `IAudioControl` proxy.

---

## AudioControlHAL Updates
> **Layer:** `hardware/interfaces/automotive/audiocontrol/`

### No New AudioControl HAL Version

- **`AudioControl HAL` remains at AIDL v3** — no AIDL v4 confirmed in public AOSP documentation.
- No new `IModuleChangeCallback`, `IFocusListener`, or `IAudioGainCallback` method additions confirmed.

### AIDL v3 Interface (Unchanged)

```java
interface IAudioControl {
    void setFadeTowardFront(float value);
    void setBalanceTowardRight(float value);
    void registerFocusListener(IFocusListener listener);
    void onAudioFocusChange(String usage, int zoneId, AudioFocusChange focusChange);
    void onDevicesToDuckChange(DuckingInfo[] duckingInfos);
    void onDevicesToMuteChange(MutingInfo[] mutingInfos);
    void onAudioFocusChangeWithMetaData(
        PlaybackTrackMetadata[] metaData, int zoneId, AudioFocusChange focusChange);
    void registerGainCallback(IAudioGainCallback callback);
    void setModuleChangeCallback(IModuleChangeCallback callback);
    void clearModuleChangeCallback();
}
```

### HAL Version History Summary (No Change in Android 17)

| Version | Android | Key Capabilities |
|---------|---------|-----------------|
| HIDL 1.0 | Android 9 | `setFadeTowardFront`, `setBalanceTowardRight` |
| HIDL 2.0 | Android 12 | Focus listener, HAL focus requests, muting, ducking |
| AIDL 1.0 | Android 12L | HIDL→AIDL migration (same APIs) |
| AIDL 2.0 | Android 13 | `onAudioFocusChangeWithMetaData`, `registerGainCallback` |
| AIDL 3.0 | Android 14 | `setModuleChangeCallback`, `IModuleChangeCallback#onAudioPortsChanged` |
| (none) | Android 15 / 16 / 17 | No new HAL version — CAP migrated to core Audio HAL in Android 16 |

---

## Other Platform Audio Changes with AAOS Relevance

### BLE Hearing Aid Audio Device Type

**New constant:** `AudioDeviceInfo.TYPE_BLE_HEARING_AID`

- Distinguishes Bluetooth LE hearing aids from generic LE Audio headsets (`TYPE_BLE_HEADSET`).
- System handles routing automatically for hearing aid devices; no app code required for basic routing.
- AAOS relevance: OEMs supporting accessible audio output should update device-picker UIs to distinguish hearing aids from standard headsets.

### xHE-AAC Software Encoder

**New encoder:** `c2.android.xheaac.encoder`

- System-provided software encoder for xHE-AAC (MPEG-D Unified Speech and Audio Coding).
- Mandatory loudness metadata ensures consistent perceived volume across encode sessions.
- AAOS relevance: applicable to in-vehicle streaming, voice call recording, and OEM audio capture pipelines where consistent loudness across ignition cycles is required.

---

## Deprecated APIs Inherited from Earlier Android Versions

No new HAL or `CarAudioManager` API deprecations were introduced in Android 17.

| API | Class | Deprecated In |
|-----|-------|--------------|
| `isDynamicRoutingEnabled()` | `CarAudioManager` | Android 14 |
| `getExternalSources()` | `CarAudioManager` | Android 14 |
| `createAudioPatch(String, int, int)` | `CarAudioManager` | Android 14 |
| `releaseAudioPatch(CarAudioPatchHandle)` | `CarAudioManager` | Android 14 |

---

## Appendix: Git Commands for Raw Patch Generation

```bash
# Navigate to the Car services repository
cd <AOSP_ROOT>/packages/services/Car

# Fetch all release tags
git fetch --tags

# Generate A16 → A17 audio patch (targeting main branch as A17 tag is not yet published)
git diff android-16.0.0_r1..main \
  -- service/src/com/android/car/audio/ cpp/audiocontrol/ \
  > a16_to_a17_audio.patch
```

> **Note:** Replace `main` with the appropriate Android 17 release tag (e.g., `android-17.0.0_r1`)
> once it is published on android.googlesource.com.

---

## Sources

- [Background Audio Hardening — Android 17](https://developer.android.com/about/versions/17/changes/bg-audio)
- [Android 17 Features and APIs](https://developer.android.com/about/versions/17/features)
- [Android 17 Behavior Changes: All Apps](https://developer.android.com/about/versions/17/behavior-changes-all)
- [Android 17 Behavior Changes: Apps Targeting API 37](https://developer.android.com/about/versions/17/behavior-changes-17)
- [AAOS 25Q4 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q4)
- [CarAudioManager API Reference](https://developer.android.com/reference/android/car/media/CarAudioManager)
- [Audio Control HAL — AOSP](https://source.android.com/docs/automotive/audio/audio-control-hal)
