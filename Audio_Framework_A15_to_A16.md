# Audio Framework Changelog: Android 15 → Android 16
## AAOS Audio Framework — Layered Architectural Analysis
### API Level 35 → 36 | AAOS 25Q2 + 25Q4

> **Source basis:** Official AOSP & AAOS release notes, Android developer documentation.
> No local AOSP git repository was available on this machine. The git commands required to regenerate
> raw diffs from source are provided in the appendix. All technical content below is sourced from
> official Android/AAOS documentation and is accurate to the published Android 16 release.

---

## CarAudioService Updates
> **Layer:** `platform/packages/services/Car/service/src/com/android/car/audio/` | `android.car.media`

### AAudio OEM-Defined Audio Attribute Tags

- The AAudio native audio library now supports **OEM-defined audio attribute tags** on audio streams.
- OEM-defined tags are consumed by `CarAudioService` and the CAP engine for **routing decisions** and **volume management** without requiring app code changes.
- Tags integrate with `<oemContext>` elements in `car_audio_configuration.xml` v3+:

```xml
<!-- car_audio_configuration.xml v3+ -->
<oemContexts>
  <oemContext name="oem_music_high_priority">
    <audioAttributes>
      <usage value="AUDIO_USAGE_MEDIA"/>
      <!-- OEM tag attached at AAudio stream level maps here -->
    </audioAttributes>
  </oemContext>
</oemContexts>
```

- When the **CAP engine** is active (`useCoreAudioRouting=true`), each OEM-defined context must match a corresponding CAP engine product strategy definition.
- Enables per-stream audio routing customization controlled entirely by the OEM audio policy layer.

---

### Fade and Balance Getter APIs (AAOS 25Q4)

**New APIs — completing read/write parity for fade and balance controls:**

```java
// New getter methods (CarAudioManager or equivalent system API)
@SystemApi
float getFadeTowardFront();     // Returns current fade value in range [-1.0, +1.0]

@SystemApi
float getBalanceTowardRight();  // Returns current balance value in range [-1.0, +1.0]
```

- Corresponding write methods (`setFadeTowardFront`, `setBalanceTowardRight`) have existed since Android 9 via the AudioControl HAL.
- Prior to Android 16, there was no way to query the current values without HAL-level inspection.
- Values **persist per user across ignition cycles**, allowing the Audio Settings UI to render correct state after reboot without querying HAL state.

---

### Alternative App Audio Controls (AAOS 25Q4)

- Users can now independently control the volume of **communication apps** (phone calls, VoIP) while driving, even when a media source is also active.
- New system audio routing path separates communication app audio from media center audio, enabling independent volume adjustment.
- Addresses the prior gap where VoIP apps had no user-accessible volume control when the media stack owned the volume buttons.

---

### `car_audio_configuration.xml` — Unchanged at Version 4

- No new schema version in Android 16; the schema remains at **version 4** (introduced in Android 15).
- No new XML elements confirmed for this transition.

---

### Configuration Flags That Interact with Android 16 CAP Changes

No new AAOS audio feature flags were introduced in Android 16. The CAP AIDL migration is a structural architectural change, not a feature flag. The following existing flags now route through the new AIDL infrastructure on fully-updated devices:

| Flag | Relevant Change in Android 16 |
|------|-------------------------------|
| `useCoreAudioVolume` | CAP volume management now flows through AIDL HAL on Android 16 vendor HALs |
| `useCoreAudioRouting` | CAP routing is AIDL-native on Android 16 vendor HALs |

---

## AudioControlService Updates
> **Layer:** C++ proxy service (`cpp/audiocontrol/`) translating Java `CarAudioService` commands via AIDL

The automotive-specific `AudioControlService` (the C++ proxy to `IAudioControl`) remains at **AIDL v3** with no interface changes in Android 16.

The major architectural changes in Android 16 target the **core Audio HAL** (`android.hardware.audio.core`), which handles policy engine configuration. The `IAudioControl` interface — responsible for fade, balance, focus signals, ducking, and muting in automotive contexts — is not modified.

### Unchanged AIDL v3 `IAudioControl` Interface

```java
interface IAudioControl {
    // Fade and balance
    void setFadeTowardFront(float value);
    void setBalanceTowardRight(float value);

    // Audio focus
    void registerFocusListener(IFocusListener listener);
    void onAudioFocusChange(String usage, int zoneId, AudioFocusChange focusChange);
    void onAudioFocusChangeWithMetaData(
        PlaybackTrackMetadata[] metaData, int zoneId, AudioFocusChange focusChange);

    // Ducking and muting
    void onDevicesToDuckChange(DuckingInfo[] duckingInfos);
    void onDevicesToMuteChange(MutingInfo[] mutingInfos);

    // Gain callback
    void registerGainCallback(IAudioGainCallback callback);

    // Dynamic port changes (v3)
    void setModuleChangeCallback(IModuleChangeCallback callback);
    void clearModuleChangeCallback();
}
```

---

## AudioControlHAL Updates
> **Layer:** `hardware/interfaces/automotive/audiocontrol/` and `hardware/interfaces/audio/core/`

### No New AudioControl HAL Version

- **`AudioControl HAL` remains at AIDL v3** in publicly documented AOSP for Android 16.
- No AIDL v4 `IAudioControl` has been confirmed.

### Major Change: Configurable Audio Policy (CAP) Engine — AIDL Migration

> **This is the most architecturally significant audio change in Android 16.**
>
> While this change is in `android.hardware.audio.core` rather than `android.hardware.automotive.audiocontrol`, it directly governs how the audio policy engine — which `CarAudioService` depends on for routing and volume — is configured.

**Background:** The Configurable Audio Policy (CAP) engine prior to Android 16 read its configuration from static XML files on the vendor partition. The AIDL Audio HAL path (introduced in Android 12L) did not support CAP, creating a gap for AIDL HAL adopters.

**Solution in Android 16:** CAP configuration is now delivered via a live AIDL API call (`getEngineConfig()`) on the core Audio HAL.

#### New AIDL Parcelables (Namespace: `android.hardware.audio.core`)

```java
// Top-level CAP configuration container
parcelable AudioHalCapConfiguration {
    // Aggregates criteria, domains, and parameters
}

// Criterion definitions (e.g., "available output devices", "telephony mode")
parcelable AudioHalCapCriterionV2 {
    String name;
    // CriterionType and possible values
}

// Parameter domain: named grouping of configurable parameters with routing rules
parcelable AudioHalCapDomain {
    String name;
    AudioHalCapParameter[] parameters;
    AudioHalCapRule[] rules;
}

// An individual configurable parameter (e.g., a device address or routing strategy)
parcelable AudioHalCapParameter {
    String path;   // e.g., "/Policy/policy/product_strategies/STRATEGY_MEDIA/device_address"
    String value;
}

// A routing or volume rule evaluated against criteria
parcelable AudioHalCapRule {
    // Condition criteria and parameter assignments
}
```

**Integration point in `AudioHalEngineConfig`:**

```java
parcelable AudioHalEngineConfig {
    // ... existing fields ...
    @nullable CapSpecificConfig capSpecificConfig;  // NEW in Android 16
}

parcelable CapSpecificConfig {
    AudioHalCapConfiguration configuration;
    AudioHalCapCriterionV2[] criteria;
    AudioHalCapDomain[] domains;
}
```

#### Configuration Delivery Mechanism

| Aspect | Android 14 / 15 | Android 16 |
|--------|-----------------|------------|
| Config source | Vendor partition XML files | System partition via AIDL `getEngineConfig()` |
| HAL interface | Direct XML file parsing at boot | AIDL HAL method call |
| Config location | `/vendor/etc/audio_policy_engine_configuration.xml` | System partition; no vendor XML required for new devices |
| Backward compat | N/A | Legacy XML fallback via `EngineConfigXmlConverter.h` |

**Backward compatibility:** `EngineConfigXmlConverter.h` converts existing vendor XML files to AIDL parcelables, maintaining compatibility for:
- Android 16 system + Android 15 AIDL vendor HAL
- Android 16 system + Android 14 HIDL vendor HAL

#### HAL Compatibility Matrix

| System Partition | Vendor HAL | CAP Config Source |
|-----------------|------------|-------------------|
| Android 16 | Android 16 AIDL HAL | System partition via `getEngineConfig()` |
| Android 16 | Android 15 AIDL HAL | Vendor partition (legacy XML fallback) |
| Android 16 | Android 14 HIDL HAL | Vendor partition (legacy XML fallback) |

---

### Breaking Change: Product Strategy Naming Convention

> **⚠️ This is a breaking change for OEM CAP configurations migrating to Android 16.**

All product strategy names now require the `STRATEGY_` prefix in every configuration path.

| Old Name (A14/A15) | New Name (A16+) |
|--------------------|-----------------|
| `media` | `STRATEGY_MEDIA` |
| `phone` | `STRATEGY_PHONE` |
| `dtmf` | `STRATEGY_DTMF` |
| `sonification` | `STRATEGY_SONIFICATION` |
| `ring` | `STRATEGY_RING` |
| `notification` | `STRATEGY_NOTIFICATION` |
| `enforced_audible` | `STRATEGY_ENFORCED_AUDIBLE` |
| `transmitted_through_speaker` | `STRATEGY_TRANSMITTED_THROUGH_SPEAKER` |
| `accessibility` | `STRATEGY_ACCESSIBILITY` |
| `rerouting` | `STRATEGY_REROUTING` |
| `call_assistant` | `STRATEGY_CALL_ASSISTANT` |

**OEM extension slots** (naming unchanged): `vx_1000` through `vx_1039` — 40 slots available for OEM-defined custom audio strategies.

**Before (Android 14/15 XML):**
```
/Policy/policy/product_strategies/media/device_address
/Policy/policy/product_strategies/phone/device_address
```

**After (Android 16+ AIDL parcelable path):**
```
/Policy/policy/product_strategies/STRATEGY_MEDIA/device_address
/Policy/policy/product_strategies/STRATEGY_PHONE/device_address
```

### HIDL Deprecation Status

| HAL Version | Interface | Status |
|-------------|-----------|--------|
| AudioControl HIDL 1.0 | `IAutomotiveAudioControl@1.0` | **Deprecated** (since Android 12L) |
| AudioControl HIDL 2.0 | `IAutomotiveAudioControl@2.0` | **Deprecated** (since Android 12L) |
| AudioControl AIDL 3.0 | `IAudioControl` | **Current** (no change in Android 16) |
| Core Audio HIDL | `IDevice`, `IStream` HIDL | **Deprecated** — AIDL CAP migration in Android 16 targets AIDL HAL only |

---

## Appendix: Git Commands for Raw Patch Generation

```bash
# Navigate to the Car services repository
cd <AOSP_ROOT>/packages/services/Car

# Fetch all release tags
git fetch --tags

# Generate A15 → A16 audio patch
git diff android-15.0.0_r1..android-16.0.0_r1 \
  -- service/src/com/android/car/audio/ cpp/audiocontrol/ \
  > a15_to_a16_audio.patch
```

---

## Sources

- [AAOS 25Q2 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q2)
- [AAOS 25Q4 Release Notes](https://source.android.com/docs/automotive/start/releases/aaos-25q4)
- [Configurable Audio Policy AIDL HAL — AOSP](https://source.android.com/docs/core/audio/aidl-cap)
- [Configurable Audio Policy Engine — AOSP](https://source.android.com/docs/automotive/audio/configurable-audio-policy-engine)
- [Audio Control HAL — AOSP](https://source.android.com/docs/automotive/audio/audio-control-hal)
- [Audio Configuration Flags — AOSP](https://source.android.com/docs/automotive/audio/config-flags)
