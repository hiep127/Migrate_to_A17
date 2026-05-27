# AOSP Car Audio Framework Changelog
## Android 14 → 15 → 16 → 17 (main)
**Focus: `service/src/com/android/car/audio/`**

---

## Section 1: Android 14 → Android 15

### 1. High-Level Architectural Changes

- **AudioManager Abstraction Layer**: A new `AudioManagerWrapper` class was introduced to wrap all calls to `android.media.AudioManager`. This decouples the car audio service from the system API and enables full unit-test coverage without requiring a real `AudioManager` instance.
- **Dynamic Routing Prioritization**: `CarAudioDynamicRouting.setupAudioDynamicRouting()` was refactored to only activate *selected and active* zone configurations. The default config is now always set up last as a routing fallback, preventing silent failures when a dynamically-connected device disconnects.
- **Fade Manager Configuration**: `FadeManagerConfiguration` support was wired into audio focus dispatch via `dispatchAudioFocusChangeWithFade()`, allowing smooth cross-fade between focus holders instead of hard cuts.
- **Audio Server Resilience**: A new `CarAudioServerStateCallback` is registered with `AudioManager` so the car audio service can detect audio server crashes (`isAudioServerRunning() == false`) and reinitialize routing and policy automatically.
- **Playback Session Tracking**: `CarAudioPlaybackMonitor` was added to observe `AudioPlaybackConfiguration` changes in real time, enabling the service to know which audio contexts are actively playing at any moment.

### 2. File Movements & Refactoring

**New files introduced (all under `car/audio/`):**

| File | Purpose |
|---|---|
| `AudioManagerWrapper.java` | Wraps all `AudioManager` calls for testability |
| `CarActivationVolumeConfig.java` | Stores per-group min/max activation volume percentages |
| `CarAudioDeviceCallback.java` | Handles dynamic audio device connect/disconnect events |
| `CarAudioFadeConfigurationHelper.java` | Builds `FadeManagerConfiguration` from XML/config |
| `CarAudioParserUtils.java` | Extracted XML parsing utilities from `CarAudioZonesHelper` |
| `CarAudioPlaybackMonitor.java` | Monitors `AudioPlaybackCallback` to track playing contexts |
| `CarAudioProtoUtils.java` | Serializes car audio state to protobuf for debugging |
| `CarAudioServerStateCallback.java` | Implements `AudioManager.AudioServerStateCallback` |

**HAL subfolder (`audio/hal/`) — modified in place, not yet moved:**
- `AudioControlWrapper.java`, `AudioControlWrapperAidl.java`, `AudioControlWrapperV1.java`, `AudioControlWrapperV2.java`
- `HalAudioDeviceInfo.java`, `HalAudioFocus.java`, `HalFocusListener.java`

### 3. New Feature Implementations

- **`CarActivationVolumeConfig`**: Stores an activation volume range (percentage min/max) for each volume group. When a volume group is activated (e.g., incoming call, alarm), the framework clamps the initial gain index into this range.
- **`CarAudioDeviceCallback`**: Implements `AudioDeviceCallback` and calls back into `CarAudioService` on device add/remove events. Used to re-evaluate routing for dynamic devices (BT, USB, etc.).
- **`CarAudioFadeConfigurationHelper`**: Parses `<fadeConfiguration>` entries from the car audio XML and produces `FadeManagerConfiguration` objects used in audio focus dispatch to enable smooth crossfades.
- **`CarAudioPlaybackMonitor`**: Wraps `AudioManager.registerAudioPlaybackCallback()`. Maintains a live set of `AudioPlaybackConfiguration` entries and notifies zone focus handlers so they can make better ducking/focus decisions based on what is actually audible.
- **`CarAudioServerStateCallback`**: When the audio server restarts (e.g., after a crash), this callback triggers a full reinitialization of `CarAudioService`'s audio policies, volume groups, and routing — preventing a silent broken audio state.

### 4. Logic Deep Dive: Dynamic Routing Fallback

**Old behavior (A14):** Every zone configuration was unconditionally added to the routing policy builder:
```java
static void setupAudioDynamicRouting(AudioPolicy.Builder builder,
        SparseArray<CarAudioZone> carAudioZones, CarAudioContext carAudioContext) {
    for (...) {
        for (CarAudioZoneConfig config : zoneConfigs) {
            setupAudioDynamicRoutingForZoneConfig(builder, config, carAudioContext);
        }
    }
}
```

**New behavior (A15):** Only the active selected configuration is routed; the default config is always appended last as a fallback:
```java
static void setupAudioDynamicRouting(CarAudioContext carAudioContext,
        AudioManagerWrapper audioManager, AudioPolicy.Builder builder,
        SparseArray<CarAudioZone> carAudioZones) {
    for (...) {
        CarAudioZoneConfig defaultConfig = null;
        for (CarAudioZoneConfig config : zoneConfigs) {
            if (config.isDefault()) { defaultConfig = config; continue; }
            if (!config.isSelected() || !config.isActive()) { continue; }
            setupAudioDynamicRoutingForZoneConfig(...);
        }
        // Default config always added last as routing fallback
        setupAudioDynamicRoutingForZoneConfig(..., defaultConfig, ...);
    }
}
```

**Why this matters:** When a zone uses a dynamic device configuration (e.g., a USB audio output) and that device disconnects, the routing policy must fall back to the default (built-in speaker) config without any manual intervention. By ensuring the default config is always registered in the policy — even when another config is selected — the audio framework has a valid routing path to fall back to automatically.

---

## Section 2: Android 15 → Android 16

### 1. High-Level Architectural Changes

- **`CarAudioZonesHelper` Promoted from Class to Interface**: The monolithic 1056-line XML-parsing implementation was replaced with a small interface. Two concrete implementations now coexist:
  - `CarAudioZonesHelperImpl` — the backward-compatible XML-based loader (1234 lines, refactored from the old class)
  - `CarAudioZonesHelperAudioControlHAL` — a new HAL API-driven loader using `AudioControlZoneConverter`
- **API-Driven Zone Configuration**: OEMs can now deliver audio zone topology (zones, volume groups, contexts, ports) through the Audio Control HAL AIDL interface (`android.hardware.automotive.audiocontrol.AudioZone`) instead of committing to a static XML file baked into the system image.
- **Zone Conversion Pipeline**: `AudioControlZoneConverter` + `AudioControlZoneConverterUtils` implement a structured conversion pipeline from HAL AIDL types to car audio service types, with explicit error reporting at each conversion stage.

### 2. File Movements & Refactoring

**New files introduced:**

| File | Purpose |
|---|---|
| `AudioControlZoneConverter.java` | Converts HAL `AudioZone` AIDL objects to `CarAudioZone` |
| `AudioControlZoneConverterUtils.java` | Conversion helpers (device ports, volume groups, fade configs) |
| `CarAudioZonesHelperAudioControlHAL.java` | Zone helper that reads config from Audio Control HAL API |
| `CarAudioZonesHelperImpl.java` | Zone helper that reads config from XML (refactored old class) |
| `TEST_MAPPING` | Test mapping for CI routing |

**Converted:**
- `CarAudioZonesHelper.java` — converted from a `final class` (~1056 lines) to an `interface` (~64 lines)

**HAL subfolder still present and modified** (`audio/hal/AudioControlWrapper*.java`)

### 3. New Feature Implementations

- **`AudioControlZoneConverter`**: Accepts a HAL `AudioZone` and `AudioDeviceConfiguration` and emits a `CarAudioZone`. It validates each step: contexts must be non-empty, volume groups must parse cleanly, input device addresses must resolve to real hardware devices. On any failure it returns `null` and logs to the local ring buffer.
- **`AudioControlZoneConverterUtils`**: Contains the low-level conversion logic — `convertAudioDevicePort()` maps an AIDL `AudioPort` to a `CarAudioDeviceInfo`, `convertVolumeGroupConfig()` maps `VolumeGroupConfig` to `CarVolumeGroup`, `convertAudioFadeConfiguration()` maps `AudioZoneFadeConfiguration` to `CarAudioFadeConfiguration`.
- **`CarAudioZonesHelperAudioControlHAL`**: Calls `AudioControlWrapper.getAudioZones()` at service init time and feeds the results through `AudioControlZoneConverter` to populate the zone topology. No XML required.

### 4. Logic Deep Dive: CarAudioZonesHelper Interface Extraction

**Old (A15):**
```java
/* package */ final class CarAudioZonesHelper {
    private static final String TAG_ROOT = "carAudioConfiguration";
    // ... 50+ XML tag constants
    // ... 1000+ lines of XmlPullParser logic

    CarAudioZones loadAudioZones() throws IOException, XmlPullParserException {
        // Opens InputStream, parses <carAudioConfiguration> XML
    }
}
```

**New (A16):**
```java
/**
 * Interface for loading car audio service configurations
 */
interface CarAudioZonesHelper {
    SparseArray<CarAudioZone> loadAudioZones() throws IOException, XmlPullParserException;
    // + a few default helper signatures
}
```

`CarAudioService.init()` now selects the implementation:
```java
boolean halSupportsZoneConfig = mAudioControlWrapper.supportsAudioZones();
mCarAudioZonesHelper = halSupportsZoneConfig
        ? new CarAudioZonesHelperAudioControlHAL(mAudioControlWrapper, mAudioManagerWrapper, ...)
        : new CarAudioZonesHelperImpl(mContext, mAudioManagerWrapper, ...);
```

**Why this matters:** OEM devices that implement the `getAudioZones()` HAL API can now deliver their zone layout entirely at runtime — no need for a car_audio_configuration.xml baked into the system image. This allows zone topology updates via HAL without OTA, and lets the same service binary handle radically different vehicle audio configurations.

---

## Section 3: Android 16 → Android 17 (main / Beta 4)

### 1. High-Level Architectural Changes

- **Audio Policy Looper Consolidation**: All `AudioPolicy.Builder` instances now use `Looper.getMainLooper()` instead of the `CarAudioService` `HandlerThread` looper. The separate `HandlerThread` and `HandlerExecutor` are no longer injected into audio policy builders.
- **Zone Config Change Routing Unified**: The intermediate `changeAudioPolicyForConfigChangeLocked()` method was removed. Zone configuration switches now call the three routing steps (`setupRoutingAudioPolicyLocked()` → `setAllUserIdDeviceAffinitiesToNewPolicyLocked()` → `swapRoutingAudioPolicyLocked()`) directly and explicitly, making the flow transparent.
- **Feature Flag Cleanup**: `AUDIO_FEATURE_PERSIST_FADE_BALANCE_VALUES` / `mPersistFadeBalanceLevels` was removed from `CarAudioService` — the persist-fade-balance feature was consolidated into the core fade manager path.
- **Tracing Overhead Removed from Hot Paths**: `TimingsTraceLog` instrumentation was stripped from the focus evaluation inner loop (`CarAudioFocus`, `FocusInteraction`) — these paths are now considered stable and the per-call trace overhead was hurting performance on high-frequency focus requests.
- **Core Audio Routing Awareness in Device Updates**: `updateVolumeDevices()` now accepts a `useCoreAudioRouting` boolean. When core audio routing is active, `CoreAudioVolumeGroup` skips the device-to-attribute mapping update that is only needed for legacy AudioPolicy routing.

### 2. File Movements & Refactoring

No new or deleted files — all changes are in-place modifications:

| File | Key Change |
|---|---|
| `CarAudioFocus.java` | Removed `TimingsTraceLog` from focus result build path |
| `CarAudioService.java` | Removed `HandlerExecutor`, `mPersistFadeBalanceLevels`; unified config-change routing |
| `CarAudioVolumeGroup.java` | Javadoc reference fix |
| `CarAudioZone.java` | Passes `useCoreAudioRouting` flag into `updateVolumeDevices()` |
| `CarAudioZoneConfig.java` | Propagates `useCoreAudioRouting` to volume groups |
| `CarVolumeGroup.java` | Removed `mEventLogger` (LocalLog); added `useCoreAudioRouting` param |
| `ContentObserverFactory.java` | Binds ContentObserver to `Looper.getMainLooper()` directly |
| `CoreAudioVolumeGroup.java` | Skips device update when `useCoreAudioRouting == true` |
| `CoreAudioVolumeGroupCallback.java` | Moved executor injection from constructor to `init(Executor)` |
| `FocusInteraction.java` | Removed `TimingsTraceLog`; simplified ContentObserver registration |

### 3. New Feature Implementations

No new classes. Key behavioral improvements:

- **Core Audio Routing Aware Device Updates**: `CoreAudioVolumeGroup.updateDevices(boolean useCoreAudioRouting)` returns early when core routing is active. This prevents the framework from unnecessarily re-deriving device-to-AudioAttribute mappings on every zone config synchronization, reducing CPU overhead on devices that use the CoreAudio volume group path.
- **Executor Injection Cleanup**: `CoreAudioVolumeGroupCallback` no longer stores the `Executor` as a field — it is passed in at `init()` call time. This matches the pattern used by `AudioManagerWrapper.setAudioServerStateCallback()`, making the executor lifecycle consistent across all audio callbacks.

### 4. Logic Deep Dive: AudioPolicy Looper Consolidation

**Old (A16):** Audio policies used a dedicated `HandlerThread` looper:
```java
// CarAudioService constructor:
mHandlerThread = CarServiceUtils.getHandlerThread(CarAudioService.class.getSimpleName());

// Audio policy builders:
AudioPolicy.Builder builder = new AudioPolicy.Builder(mContext);
builder.setLooper(mHandlerThread.getLooper()); // 4 separate builders all used this

var executor = new HandlerExecutor(mHandler);
mAudioManagerWrapper.setAudioServerStateCallback(executor, mAudioServerStateCallback);

mCoreAudioVolumeGroupCallback = new CoreAudioVolumeGroupCallback(
        carVolumeInfoWrapper, mAudioManagerWrapper, executor); // executor stored in constructor
mCoreAudioVolumeGroupCallback.init();
```

**New (A17):**
```java
// Audio policy builders:
AudioPolicy.Builder builder = new AudioPolicy.Builder(mContext);
builder.setLooper(Looper.getMainLooper()); // All 4 builders now use main looper

mAudioManagerWrapper.setAudioServerStateCallback(
        mContext.getMainExecutor(), mAudioServerStateCallback);

mCoreAudioVolumeGroupCallback = new CoreAudioVolumeGroupCallback(
        carVolumeInfoWrapper, mAudioManagerWrapper); // no executor in constructor
mCoreAudioVolumeGroupCallback.init(mContext.getMainExecutor()); // injected at init
```

Also, the zone config change method was simplified:
```java
// Old: indirection through changeAudioPolicyForConfigChangeLocked()
zone.setCurrentCarZoneConfig(zoneConfig);
newAudioPolicy = changeAudioPolicyForConfigChangeLocked(); // had a core-audio bypass branch

// New: direct three-step sequence, always executed
zone.setCurrentCarZoneConfig(zoneConfig);
newAudioPolicy = setupRoutingAudioPolicyLocked();
setAllUserIdDeviceAffinitiesToNewPolicyLocked(newAudioPolicy);
swapRoutingAudioPolicyLocked(newAudioPolicy);
```

**Why this matters:** Using `Looper.getMainLooper()` instead of a custom `HandlerThread` removes one background thread and ensures all audio policy callbacks are serialized on the main looper. This aligns with Android's system service best practices post-Android 16, where `Context.getMainExecutor()` is the preferred executor for framework callbacks. The removal of the core-audio bypass in `changeAudioPolicyForConfigChangeLocked()` also fixes a latent bug where zone config switches on core-audio-routing devices would not update user device affinities.

---

## Summary Table

| Aspect | A14 → A15 | A15 → A16 | A16 → A17 |
|---|---|---|---|
| Patch size | 10,942 lines | 5,754 lines | 485 lines |
| New files | 8 new classes | 4 new classes + TEST_MAPPING | 0 |
| Deleted/moved | None | `CarAudioZonesHelper` converted to interface | None |
| Key theme | Testability, resilience, playback monitoring | API-driven zone config, HAL abstraction | Threading cleanup, trace removal, routing simplification |
| HAL subfolder | Still in `audio/hal/` | Still in `audio/hal/` | Still in `audio/hal/` |
| Breaking changes | `setupAudioDynamicRouting()` signature changed | `CarAudioZonesHelper` is now an interface | `updateVolumeDevices()` now requires `boolean` param |
