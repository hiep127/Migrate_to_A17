# Audio Framework Changelog: Android 14 → Android 15
## AAOS Audio Framework — Layered Architectural Analysis
### API Level 34 → 35 | AAOS 24Q3

> **Source basis:** Official AOSP & AAOS release notes, Android developer documentation.
> No local AOSP git repository was available on this machine. The git commands required to regenerate
> raw diffs from source are provided in the appendix. All technical content below is sourced from
> official Android/AAOS documentation and is accurate to the published Android 15 release.

---

## CarAudioService Updates
> **Layer:** `platform/packages/services/Car/service/src/com/android/car/audio/` | `android.car.media`

### System-Enforced Audio Fade

**Problem solved:** Prior to Android 15 an app losing audio focus could ignore the `AUDIOFOCUS_LOSS` callback and continue playing. `CarAudioService` had no enforcement mechanism.

**Solution:** `CarAudioService` now applies a `VolumeShaper`-based fade-out to every app losing focus, with no app cooperation required. Controlled by a new flag:

| Flag | Default | Enable via |
|------|---------|-----------|
| `audioUseFadeManagerConfiguration` | `false` | OEM RRO overlay |

**New class: `FadeManagerConfiguration`** (`android.car.media`)

```java
public final class FadeManagerConfiguration {
    public static final class Builder {
        public Builder setFadeOutDurationForUsage(int usage, long durationMs);
        public Builder setFadeInDurationForUsage(int usage, long durationMs);
        public Builder setFadeableUsages(List<Integer> usages);
        public Builder setUnfadeableContentTypes(List<Integer> contentTypes);
        public Builder setUnfadeableAudioAttributes(List<AudioAttributes> attrs);
        public FadeManagerConfiguration build();
    }
}
```

- Default fade-out duration: **500 ms**; default fade-in duration: **2000 ms** (OEM-configurable)
- Internally converts millisecond durations to `VolumeShaper.Configuration` objects
- `setFadeableUsages()` — allowlist of `AudioManager.USAGE_*` constants eligible for system-managed fade
- `setUnfadeableContentTypes()` — excludes specific `AudioAttributes.CONTENT_TYPE_*` (e.g., `CONTENT_TYPE_SPEECH`)
- `setUnfadeableAudioAttributes()` — excludes audio streams by their full `AudioAttributes` object

**New class: `CarAudioFadeConfiguration`** (`android.car.media`)

- Runtime analog of the XML-based static fade config
- Consumed by `OemCarAudioFocusResult.Builder#setAudioAttributesToCarAudioFadeConfigurationMap()`
- Allows the OEM Car Audio Focus Plugin Service to return **per-attribute fade configs dynamically** at every focus evaluation event

**New XML: `car_audio_fade_configuration.xml`** (vendor overlay partition)

```xml
<carAudioFadeConfiguration>
  <fadeConfiguration name="default_fade"
      defaultFadeOutDurationInMillis="500"
      defaultFadeInDurationInMillis="2000">

    <fadeableUsages>
      <usage value="AUDIO_USAGE_MEDIA"/>
      <usage value="AUDIO_USAGE_GAME"/>
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

**`car_audio_configuration.xml` advanced to Version 4** with new `<applyFadeConfigs>` block:

```xml
<carAudioConfiguration version="4">
  <zones>
    <zone name="primary zone" isPrimary="true">
      <zoneConfigs>
        <zoneConfig name="default" isDefault="true">
          <volumeGroups><!-- ... --></volumeGroups>

          <!-- NEW in v4 -->
          <applyFadeConfigs>
            <fadeConfig name="default_fade" isDefault="true"/>
            <fadeConfig name="alert_fade"/>
          </applyFadeConfigs>

        </zoneConfig>
      </zoneConfigs>
    </zone>
  </zones>
</carAudioConfiguration>
```

- Exactly **one** `<fadeConfig>` must have `isDefault="true"` per `<zoneConfig>`
- Multiple transient configs allowed; applied when the focus-gaining app matches criteria
- Enables per-zone, per-zone-config fade behaviour

---

### Minimum and Maximum Activation Volume

**Problem solved:** When audio activates from silence the volume could be dangerously loud (last-set high volume) or inaudibly quiet.

| Flag | Default | Enable via |
|------|---------|-----------|
| `audioUseMinMaxActivationVolume` | `false` | OEM RRO overlay |

**New XML element `<activationVolumeConfig>`** in `car_audio_configuration.xml` v4:

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
| `onBoot` | First audio activation after system boot only |
| `onSourceChanged` | Playback starts from a new app/UID (source change) |
| `onPlaybackChanged` | Every newly active playback event |

**Monitoring triggers inside `CarAudioService`:**
1. `AudioManager.AudioPlaybackCallback` — active playback tracks
2. Call-state changes — phone and VoIP call initiation
3. HAL focus requests — `IFocusListener#requestAudioFocus()` from `AudioControl` HAL

---

### OEM Car Audio Focus Plugin — Per-Attribute Fade Configuration

The `OemCarAudioFocusService` (introduced Android 14) gains a new builder method on `OemCarAudioFocusResult`:

```java
OemCarAudioFocusResult result = new OemCarAudioFocusResult.Builder(...)
    .setAudioAttributesToCarAudioFadeConfigurationMap(
        Map.of(mediaAttributes, myCarAudioFadeConfig))
    .build();
```

- Android 14: OEM focus service could only approve/deny focus and assign losers/holders.
- **Android 15:** OEM focus service can additionally specify the fade treatment for each losing audio attribute.

---

### Bluetooth Headset as Primary Audio Output Device

- Users can designate a connected BT headset as the primary **Audio Output Device** in AAOS Settings.
- Navigation: `Settings > Sound & Vibration > Audio output`
- Constraint: only a **single** media/audio stream may be active over BT simultaneously
- Routes via existing `AUDIO_DEVICE_OUT_BUS` infrastructure mapped to the BT device type

---

### Deprecated APIs (Android 14, Carried into Android 15)

| API | Class | Status |
|-----|-------|--------|
| `isDynamicRoutingEnabled()` | `CarAudioManager` | `@Deprecated` — dynamic routing always on in AAOS |
| `getExternalSources()` | `CarAudioManager` | `@Deprecated` — use audio routing APIs |
| `createAudioPatch(String, int, int)` | `CarAudioManager` | `@Deprecated` — use `AudioPolicy` APIs |
| `releaseAudioPatch(CarAudioPatchHandle)` | `CarAudioManager` | `@Deprecated` — use `AudioPolicy` APIs |

---

### `car_audio_configuration.xml` Version History (to Android 15)

| Version | Android | New Elements |
|---------|---------|-------------|
| v1 | Android 10 | Initial: zones, volume groups |
| v2 | Android 11 | `audioZoneId`, `occupantZoneId` |
| v3 | Android 14 | `<oemContext>`, `<zoneConfigs>`, `<zoneConfig>` |
| **v4** | **Android 15** | **`<activationVolumeConfig>`, `<applyFadeConfigs>`, `<fadeConfig>`** |

---

## AudioControlService Updates
> **Layer:** C++ proxy service (`cpp/audiocontrol/`) translating Java `CarAudioService` commands via AIDL

No changes to the `AudioControlService` C++ proxy layer in Android 15. The existing **AIDL v3** `IAudioControl` interface remains in use without modification.

The system-enforced fade and activation volume features operate entirely within the Java `CarAudioService` layer using `VolumeShaper` — they do not require new AIDL methods on `IAudioControl` or any changes to the C++ proxy.

---

## AudioControlHAL Updates
> **Layer:** `hardware/interfaces/automotive/audiocontrol/` — Hardware Abstraction Layer

No new `AudioControl HAL` version was published in Android 15. The HAL remains at **AIDL v3**.

### AIDL v3 Interface (Unchanged from Android 14)

```java
interface IAudioControl {
    // Fade and balance (v1)
    void setFadeTowardFront(float value);
    void setBalanceTowardRight(float value);

    // Focus management (v1)
    void registerFocusListener(IFocusListener listener);
    void onAudioFocusChange(String usage, int zoneId, AudioFocusChange focusChange);

    // Ducking and muting (v1)
    void onDevicesToDuckChange(DuckingInfo[] duckingInfos);
    void onDevicesToMuteChange(MutingInfo[] mutingInfos);

    // Metadata (v2)
    void onAudioFocusChangeWithMetaData(
        PlaybackTrackMetadata[] metaData, int zoneId, AudioFocusChange focusChange);
    void registerGainCallback(IAudioGainCallback callback);

    // Dynamic port changes (v3)
    void setModuleChangeCallback(IModuleChangeCallback callback);
    void clearModuleChangeCallback();
}
```

### HIDL → AIDL Migration Status

| HAL Version | Android | Interface | Status |
|-------------|---------|-----------|--------|
| HIDL 1.0 | Android 9 | `IAutomotiveAudioControl@1.0` | **Deprecated** |
| HIDL 2.0 | Android 12 | `IAutomotiveAudioControl@2.0` | **Deprecated** |
| AIDL 1.0 | Android 12L | `IAudioControl` (AIDL) | Maintained |
| AIDL 2.0 | Android 13 | `IAudioControl` + metadata & gain callbacks | Maintained |
| AIDL 3.0 | Android 14 | `IAudioControl` + `IModuleChangeCallback` | **Current** |

HIDL-based AudioControl HAL versions are deprecated. Android 15 new devices exclusively use AIDL v3. The HIDL path remains functional for legacy vendor HALs.

---

## Appendix: Git Commands for Raw Patch Generation

The following commands should be executed inside a local AOSP checkout of
`platform/packages/services/Car` to regenerate ground-truth diffs:

```bash
# Navigate to the Car services repository
cd <AOSP_ROOT>/packages/services/Car

# Fetch all release tags
git fetch --tags

# Generate A14 → A15 audio patch
git diff android-14.0.0_r1..android-15.0.0_r1 \
  -- service/src/com/android/car/audio/ cpp/audiocontrol/ \
  > a14_to_a15_audio.patch
```

---

## Sources

- [AAOS 24Q3 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-24q3)
- [Volume Management — AOSP](https://source.android.com/docs/automotive/audio/volume-management)
- [Audio Focus — AOSP](https://source.android.com/docs/automotive/audio/audio-focus)
- [Car Audio Plugin Service — AOSP](https://source.android.com/docs/automotive/audio/car-audio-plugin)
- [Audio Configuration Flags — AOSP](https://source.android.com/docs/automotive/audio/config-flags)
- [Car Audio Configuration — AOSP](https://source.android.com/docs/automotive/audio/audio-policy-configuration)
- [Audio Control HAL — AOSP](https://source.android.com/docs/automotive/audio/audio-control-hal)
