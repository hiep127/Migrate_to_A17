# AOSP Car Audio Per-File Code Changes: A14 → A17

> **Scope:** `service/src/com/android/car/audio/`
> **Source:** Derived from `a14_to_a15_audio.patch`, `a15_to_a16_audio.patch`, `a16_to_a17_audio.patch`

---

## A14 → A15

### `AudioManagerWrapper.java` — NEW FILE

Wraps every `AudioManager` call to enable unit testing without a real `AudioManager`.

```java
public final class AudioManagerWrapper {
    private final AudioManager mAudioManager;

    public AudioManagerWrapper(AudioManager audioManager) {
        mAudioManager = audioManager;
    }

    public void setAudioDeviceGain(String address, int gain, boolean isOutput) {
        AudioManagerHelper.setAudioDeviceGain(mAudioManager, address, gain, isOutput);
    }

    // delegates: getMinVolumeIndexForAttributes, dispatchAudioFocusChangeWithFade,
    //            isMasterMuted, setMasterMute, isStreamMute, getDevices, ...
    // static: getAudioProductStrategies(), getAudioVolumeGroups()
}
```

---

### `CarActivationVolumeConfig.java` — NEW FILE

Stores per-group activation volume range (min/max %) and invocation type.

```java
public CarActivationVolumeConfig(int invocationType,
        int minActivationVolumePercentage, int maxActivationVolumePercentage) {
    Preconditions.checkArgument(
            minActivationVolumePercentage < maxActivationVolumePercentage, ...);
}

// @IntDef constants:
// ACTIVATION_VOLUME_ON_BOOT = 1
// ACTIVATION_VOLUME_ON_SOURCE_CHANGED = 1 << 1
// ACTIVATION_VOLUME_ON_PLAYBACK_CHANGED = 1 << 2
```

---

### `CarAudioDeviceCallback.java` — NEW FILE

Bridge between `AudioManager` device events and `CarAudioService`.

```java
final class CarAudioDeviceCallback extends AudioDeviceCallback {
    @Override
    public void onAudioDevicesAdded(AudioDeviceInfo[] addedDevices) {
        mCarAudioService.onAudioDevicesAdded(addedDevices);
    }
    @Override
    public void onAudioDevicesRemoved(AudioDeviceInfo[] removedDevices) {
        mCarAudioService.onAudioDevicesRemoved(removedDevices);
    }
}
```

---

### `CarAudioFadeConfigurationHelper.java` — NEW FILE

XML parser for `/vendor/etc/car_audio_fade_configuration.xml`.

```java
public final class CarAudioFadeConfigurationHelper {
    static final String FADE_CONFIGURATION_PATH =
            "/vendor/etc/car_audio_fade_configuration.xml";

    // Parses: fadeable usages, unfadeable content types,
    //         fade-in/fade-out configs per usage and per audio attribute
    public CarAudioFadeConfiguration getCarAudioFadeConfiguration(String configName) { ... }
    public boolean isConfigAvailable(String configName) { ... }
    public List<String> getAllConfigNames() { ... }
}
```

---

### `CarAudioParserUtils.java` — NEW FILE

XML parsing utilities extracted from `CarAudioZonesHelper` to avoid duplication.

```java
// Shared constants: TAG_AUDIO_ATTRIBUTES, TAG_USAGE, ATTR_USAGE_VALUE, etc.
static AudioAttributes parseAudioAttributes(XmlPullParser, String tagName) { ... }
static int parseUsageValue(XmlPullParser) { ... }
static int parsePositiveIntAttribute(String attr, XmlPullParser) { ... }
static void skip(XmlPullParser) { ... }
```

---

### `CarAudioPlaybackMonitor.java` — NEW FILE

Tracks which audio contexts are actively playing per zone; triggers activation volume logic.

```java
void onActiveAudioPlaybackAttributesAdded(
        List<Pair<AudioAttributes, Integer>> activePlaybackAttributes, int zoneId) {
    for (int i = 0; i < activePlaybackAttributes.size(); i++) {
        Pair<AudioAttributes, Integer> pair = activePlaybackAttributes.get(i);
        int groupId = mCarAudioZones.get(zoneId)
                .getVolumeGroupForAudioAttributes(pair.first);
        handleActivationVolumeForGroup(zoneId, groupId,
                ACTIVATION_VOLUME_ON_PLAYBACK_CHANGED);
    }
}
```

---

### `CarAudioProtoUtils.java` — NEW FILE

Static utilities for protobuf serialization of fade configurations (used in bugreports). No state, all methods static.

---

### `CarAudioServerStateCallback.java` — NEW FILE

Detects audio server crashes and triggers `CarAudioService` reinit.

```java
// Implements AudioManager.AudioServerStateCallback
// On restart: calls CarAudioService.releaseAudioCallbacks() + re-init sequence
```

---

### `CarAudioContext.java` — MODIFIED

```java
// REMOVED:
private final Map<Integer, int[]> mContextsToDuck;
public int[] getContextsToDuck(int context) { ... }

// ADDED:
public static int getLegacyContextForUsage(int usage) {
    for (int index = 0; index < CONTEXT_TO_USAGES.size(); index++) {
        int[] usages = CONTEXT_TO_USAGES.valueAt(index);
        if (Arrays.contains(usages, usage)) return CONTEXT_TO_USAGES.keyAt(index);
    }
    return CarAudioContext.INVALID;
}

// AudioAttributesWrapper inner class:
// ADDED field: int mCarAudioContextId (= INVALID in legacy path)
// ADDED constructor: AudioAttributesWrapper(AudioAttributes, int carAudioContextId)
// CHANGED equals()/hashCode(): context-id-based when valid
public int getCarAudioContextId() { ... }
```

---

### `CarAudioDeviceInfo.java` — MODIFIED (major refactor)

```java
// BEFORE (A14):
CarAudioDeviceInfo(AudioManager audioManager, AudioDeviceInfo audioDeviceInfo) {
    mAudioManager = audioManager;
    mAudioDeviceInfo = audioDeviceInfo;
    mAddress = audioDeviceInfo.getAddress();
}
public AudioDeviceInfo getAudioDeviceInfo() { ... }

// AFTER (A15):
CarAudioDeviceInfo(AudioManagerWrapper audioManager,
        AudioDeviceAttributes audioDeviceAttributes) {
    mAudioManager = audioManager;
    mAudioDeviceAttributes = audioDeviceAttributes;
    mSampleRate = DEFAULT_SAMPLE_RATE;
    mEncodingFormat = DEFAULT_ENCODING_FORMAT;
    mChannelCount = DEFAULT_NUM_CHANNELS;
    mCurrentGain = UNINITIALIZED_GAIN;
}

// ADDED methods:
boolean isActive()
void audioDevicesAdded(AudioDeviceInfo[] devices)
void audioDevicesRemoved(AudioDeviceInfo[] devices)
void setAudioDeviceInfo(@Nullable AudioDeviceInfo info)
AudioDeviceAttributes getAudioDevice()   // replaces getAudioDeviceInfo()

// NEW constants:
static final int DEFAULT_NUM_CHANNELS = 1;
static final int DEFAULT_ENCODING_FORMAT = ENCODING_PCM_16BIT;
static final int UNINITIALIZED_GAIN = -1;
```

---

### `CarAudioDynamicRouting.java` — MODIFIED

```java
// BEFORE (A14): routes every zone config unconditionally
static void setupAudioDynamicRouting(SparseArray<CarAudioZone> zones, ...) {
    for (CarAudioZoneConfig config : zoneConfigs) {
        setupAudioDynamicRoutingForZoneConfig(builder, config, ...);
    }
}

// AFTER (A15): only routes selected+active; default config always added last as fallback
static void setupAudioDynamicRouting(CarAudioContext carAudioContext,
        AudioManagerWrapper audioManager, SparseArray<CarAudioZone> zones, ...) {
    CarAudioZoneConfig defaultConfig = null;
    for (CarAudioZoneConfig config : zoneConfigs) {
        if (config.isDefault()) { defaultConfig = config; continue; }
        if (!config.isSelected() || !config.isActive()) { continue; }
        setupAudioDynamicRoutingForZoneConfig(...);
    }
    setupAudioDynamicRoutingForZoneConfig(..., defaultConfig, ...); // always last
}
```

---

### `CarAudioFocus.java` — MODIFIED

```java
// BEFORE (A14):
private void sendFocusLossLocked(AudioFocusInfo loser, int lossType) {
    mAudioManager.dispatchAudioFocusChange(loser, lossType, mAudioPolicy);
}

// AFTER (A15): cross-fade support added
private void sendFocusLossLocked(AudioFocusInfo loser, int lossType,
        AudioFocusInfo winner, boolean shouldFade,
        FadeManagerConfiguration transientFadeManagerConfig) {
    if (shouldFade && isFadeManagerSupported()) {
        mAudioManager.dispatchAudioFocusChangeWithFade(
                loser, lossType, transientFadeManagerConfig, winner, List.of());
        return;
    }
    mAudioManager.dispatchAudioFocusChange(loser, lossType, mAudioPolicy);
}

// ALSO:
// AudioManager mAudioManager -> AudioManagerWrapper mAudioManager
// evaluateRequest() uses int requestedUsage instead of @AudioContext int requestedContext
// ADDED: isFadeManagerSupported(), getTransientFadeManagerConfig()
// REMOVED: removeDelayedAudioFocusRequestLocked() (merged into removeFocusEntryLocked)
```

---

### `CarAudioService.java` — MODIFIED (very large)

```java
// BEFORE (A14): single AudioPolicy
private AudioPolicy mAudioPolicy;

// AFTER (A15): split into four policies
private AudioPolicy mVolumeControlAudioPolicy;
private AudioPolicy mFocusControlAudioPolicy;
private AudioPolicy mRoutingAudioPolicy;
private AudioPolicy mFadeManagerConfigAudioPolicy;

// Constructor:
// BEFORE: CarAudioService(Context, String audioConfigPath, CarVolumeCallbackHandler)
// AFTER:  CarAudioService(Context, AudioManagerWrapper, String audioConfigPath,
//                         CarVolumeCallbackHandler, String audioFadeConfigPath)

// NEW fields:
private boolean mUseMinMaxActivationVolume;
private boolean mUseFadeManagerConfiguration;
private CarAudioServerStateCallback mAudioServerStateCallback;
private CarAudioDeviceCallback mAudioDeviceInfoCallback;
private CarAudioPlaybackMonitor mCarAudioPlaybackMonitor;
private CarAudioFadeConfigurationHelper mCarAudioFadeConfigurationHelper;

// init() now checks isAudioServerRunning(); defers if server is down
// NEW: setupAudioDeviceInfoCallbackLocked() / releaseAudioDeviceInfoCallbackLocked()
// NEW: releaseAudioCallbacks(boolean isAudioServerDown)
```

---

### `CarAudioVolumeGroup.java` — MODIFIED

```java
// Constructor: replaced contextToAddress + addressToCarAudioDeviceInfo
//              with contextToDeviceInfo; added CarActivationVolumeConfig param
// calculateNewGainStageFromDeviceInfos() moved inside synchronized(mLock)
```

---

### `CarAudioZone.java` — MODIFIED

```java
// RENAMED: getCurrentAudioDeviceInfos() -> getCurrentAudioDevices() (returns AudioDeviceAttributes)
// RENAMED: getAudioDeviceForContext(int) -> returns AudioDeviceAttributes
// ADDED: getDefaultAudioZoneConfigInfo()
// ADDED: getVolumeGroupForAudioAttributes(AudioAttributes)
// ADDED: audioDevicesAdded(AudioDeviceInfo[])
// ADDED: audioDevicesRemoved(AudioDeviceInfo[])
// init() now sets mCurrentConfigId to default, calls setIsSelected(true) + updateVolumeDevices()
// setCurrentCarZoneConfig() deselects old, selects new, calls updateVolumeDevices()
```

---

### `CarAudioZoneConfig.java` — MODIFIED

```java
// ADDED fields:
private CarAudioFadeConfiguration mDefaultCarAudioFadeConfiguration;
private ArrayMap<AudioAttributes, CarAudioFadeConfiguration> mAudioAttributesToCarAudioFadeConfiguration;
private boolean mIsFadeManagerConfigurationEnabled;
@GuardedBy("mLock") private boolean mIsSelected;

// ADDED methods:
boolean isSelected()
void setIsSelected(boolean selected)
boolean isFadeManagerConfigurationEnabled()
boolean isActive()
void audioDevicesAdded(AudioDeviceInfo[])
void audioDevicesRemoved(AudioDeviceInfo[])
void updateVolumeDevices()
CarAudioFadeConfiguration getDefaultCarAudioFadeConfiguration()
CarAudioFadeConfiguration getCarAudioFadeConfigurationForAudioAttributes(AudioAttributes)
```

---

### `CarVolumeGroup.java` — MODIFIED (major)

```java
// BEFORE (A14): address-based mapping
CarVolumeGroup(... SparseArray<String> contextToAddress,
        ArrayMap<String, CarAudioDeviceInfo> addressToCarAudioDeviceInfo) { ... }

// AFTER (A15): device-info mapping + activation config
CarVolumeGroup(... SparseArray<CarAudioDeviceInfo> contextToDevices,
        CarActivationVolumeConfig carActivationVolumeConfig) {
    mContextToDevices = contextToDevices;
    mCarActivationVolumeConfig = carActivationVolumeConfig;
}

// ADDED:
int getMaxActivationGainIndex()
int getMinActivationGainIndex()
int getActivationVolumeInvocationType()
void handleActivationVolume(int invocationType)
boolean isActive()
void audioDevicesAdded(AudioDeviceInfo[])
void audioDevicesRemoved(AudioDeviceInfo[])
void updateDevices(boolean useCoreAudioRouting)
boolean validateDeviceTypes(Set<Integer> allowedTypes)
AudioDeviceAttributes getAudioDeviceAttributes()
```

---

### `CarVolumeGroupFactory.java` — MODIFIED

```java
// AudioManager mAudioManager -> AudioManagerWrapper mAudioManager
// Internal mapping changed from SparseArray<String> + ArrayMap<String, CarAudioDeviceInfo>
//   to SparseArray<CarAudioDeviceInfo>
```

---

### `CarZonesAudioFocus.java` — MODIFIED

```java
// createCarZonesAudioFocus(): AudioManager -> AudioManagerWrapper;
//   added @Nullable CarAudioFeaturesInfo features param
// mCarAudioService and mAudioPolicy made @GuardedBy("mLock")
// setOwningPolicy() now synchronized
// Shared ContentObserverFactory created once outside per-zone loop
// FocusInteraction no longer receives CarAudioContext in constructor
```

---

### `CoreAudioVolumeGroup.java` — MODIFIED

```java
// Constructor: AudioManager -> AudioManagerWrapper
//   contextToAddress + addressToCarAudioDeviceInfo -> contextToDevices
//   added CarActivationVolumeConfig
// Removed all VersionUtils.isPlatformVersionAtLeastU() guards (platform assumed >= U)
// isAmGroupMuted() -> mAudioManager.isVolumeGroupMuted(mAmId)
// applyMuteLocked() -> mAudioManager.adjustVolumeGroupVolume()
// ADDED: updateDevices(boolean useCoreAudioRouting)
//   sets preferred device per strategy when not using core routing
// ADDED constant: EMPTY_FLAGS = 0
```

---

### `CoreAudioVolumeGroupCallback.java` — MODIFIED

```java
// AudioManager mAudioManager -> AudioManagerWrapper mAudioManager
// Removed spurious return; statement from callback
```

---

### `FocusInteraction.java` — MODIFIED

```java
// BEFORE (A14):
FocusInteraction(CarAudioSettings settings, CarAudioContext carAudioContext,
        ContentObserverFactory observerFactory) { ... }

int evaluateRequest(@AudioContext int requestedContext, FocusEntry focusHolder) { ... }

// AFTER (A15): CarAudioContext removed; usage-based evaluation
FocusInteraction(CarAudioSettings settings,
        ContentObserverFactory observerFactory) { ... }

int evaluateRequest(int requestedUsage, FocusEntry focusHolder) { ... }
// visibility: public -> package-private
```

---

### `hal/AudioControlWrapper.java` — MODIFIED
Updated interface for new HAL capabilities introduced in A15.

### `hal/AudioControlWrapperAidl.java` — MODIFIED
Updated for AIDL HAL changes.

### `hal/AudioControlWrapperV1.java` — MODIFIED
### `hal/AudioControlWrapperV2.java` — MODIFIED
### `hal/HalAudioDeviceInfo.java` — MODIFIED
Updated for dynamic device handling.

### `hal/HalAudioFocus.java` — MODIFIED
### `hal/HalFocusListener.java` — MODIFIED

---

---

## A15 → A16

### `AudioControlZoneConverter.java` — NEW FILE

Converts HAL AIDL `AudioZone` objects to `CarAudioZone`. Returns `null` and logs on any validation failure.

```java
final class AudioControlZoneConverter {
    // Constructor:
    AudioControlZoneConverter(AudioManagerWrapper audioManager, CarAudioSettings settings,
            LocalLog serviceLog, boolean useFadeManagerConfiguration) { ... }

    @Nullable
    CarAudioZone convertAudioZone(AudioZone zone,
            AudioDeviceConfiguration deviceConfiguration) {
        // 1. validates contexts -> builds CarAudioContext
        // 2. iterates zone.audioZoneConfigs -> builds CarAudioZoneConfig each
        // 3. converts input audio devices
        // returns null (+ ring-buffer log) on any failure
    }

    List<CarAudioDeviceInfo> convertZonesMirroringAudioPorts(List<AudioPort> mirroringPorts) { ... }
}
```

---

### `AudioControlZoneConverterUtils.java` — NEW FILE

Static helpers for HAL→service type conversion (529 lines).

```java
static CarAudioContext convertCarAudioContext(
        AudioZoneContext, AudioDeviceConfiguration) { ... }
static CarAudioDeviceInfo convertAudioDevicePort(
        AudioPort, AudioManagerWrapper, ArrayMap<String, CarAudioDeviceInfo>) { ... }
static String convertVolumeGroupConfig(
        CarVolumeGroupFactory, VolumeGroupConfig, ...) { ... }  // returns error string
static CarAudioFadeConfiguration convertAudioFadeConfiguration(
        AudioFadeConfiguration) { ... }
static CarActivationVolumeConfig convertVolumeActivationConfig(
        VolumeActivationConfiguration) { ... }
static int convertToAudioDeviceInfoType(int halType, String connection) { ... }
static CarAudioContext convertCarAudioContext(AudioZoneContext, AudioDeviceConfiguration) { ... }
```

---

### `CarAudioZonesHelper.java` — STRUCTURAL CHANGE (class → interface)

```java
// BEFORE (A15): final class with ~1056 lines of XML parsing
public final class CarAudioZonesHelper {
    public SparseArray<CarAudioZone> loadAudioZones() throws IOException, XmlPullParserException {
        // full XML parsing implementation (~1000 lines)
    }
}

// AFTER (A16): interface with 8 signatures
interface CarAudioZonesHelper {
    SparseArray<CarAudioZone> loadAudioZones() throws IOException, XmlPullParserException;
    CarAudioContext getCarAudioContext();
    SparseIntArray getCarAudioZoneIdToOccupantZoneIdMapping();
    List<CarAudioDeviceInfo> getMirrorDeviceInfos();
    boolean useCoreAudioRouting();
    boolean useCoreAudioVolume();
    boolean useHalDuckingSignalOrDefault();
    boolean useVolumeGroupMuting();
}

// CarAudioService.init() selects implementation at runtime:
boolean halSupportsZoneConfig = mAudioControlWrapper.supportsAudioZones();
mCarAudioZonesHelper = halSupportsZoneConfig
        ? new CarAudioZonesHelperAudioControlHAL(mAudioControlWrapper, ...)
        : new CarAudioZonesHelperImpl(mContext, ...);
```

---

### `CarAudioZonesHelperAudioControlHAL.java` — NEW FILE

Loads zone topology from Audio Control HAL API — no XML file required.

```java
// implements CarAudioZonesHelper
public SparseArray<CarAudioZone> loadAudioZones() throws Exception {
    AudioDeviceConfiguration deviceConfig = getAudioDeviceConfiguration();
    if (deviceConfig.routingConfig == DEFAULT_AUDIO_ROUTING) {
        return new SparseArray<>();  // HAL doesn't support zone config
    }
    return initCarAudioZones(deviceConfig);  // uses AudioControlZoneConverter
}
```

---

### `CarAudioZonesHelperImpl.java` — NEW FILE

The old `CarAudioZonesHelper` class body, now implementing the interface (~1234 lines).

```java
// implements CarAudioZonesHelper
// Handles car_audio_configuration.xml versions 1-4
// NEW: parses <deviceConfigurations> section:
//   useCoreAudioVolume, useCoreAudioRouting, useHalDuckingSignals, useCarVolumeGroupMuting
// NEW: parses <activationVolumeConfigs> section
// generateCarAudioDeviceInfos() moved out to CarAudioUtils
```

---

### `TEST_MAPPING` — NEW FILE

CI test routing configuration for the car audio package.

---

### `AudioManagerWrapper.java` — MODIFIED

```java
// ADDED:
boolean isStreamMute(int stream)
List<AudioDeviceInfo> getAudioDevicesForAttributes(AudioAttributes attrs)
```

---

### `CarAudioFocus.java` — MODIFIED (traces added)

```java
// TimingsTraceLog instrumentation added to hot path
t.traceBegin("evaluate-focus-entry-build");
FocusEntry focusEntry = new FocusEntry(afi, audioContext, mPackageManager);
t.traceEnd();

t.traceBegin("evaluate-focus-result-build");
FocusEvaluationResult.Builder builder = FocusEvaluationResult.builder(focusEntry);
t.traceEnd();

// Also traces: "evaluate-focus-losers", "evaluate-focus-holders"
```

---

### `CarAudioService.java` — MODIFIED (major)

```java
// NEW: async init support
private volatile boolean mInitCompleted;
private volatile boolean mInitSuccess;

void waitForInitComplete(int timeoutInMs) {
    synchronized (mImplLock) {
        while (!mInitCompleted) mImplLock.wait(timeoutInMs);
    }
}

// NEW: HAL-first zone loading
private boolean loadAudioZonesUsingAudioControlLocked() {
    CarAudioZonesHelperAudioControlHAL helper = new CarAudioZonesHelperAudioControlHAL(...);
    mCarAudioZones = helper.loadAudioZones();
    return !mCarAudioZones.isEmpty();
}
// loadAndInitCarAudioZonesLocked() tries HAL first, falls back to XML

// ADDED: mPersistFadeBalanceLevels + AUDIO_FEATURE_PERSIST_FADE_BALANCE_VALUES feature case
private boolean mPersistFadeBalanceLevels;

// Audio policy Looper changed:
// BEFORE: builder.setLooper(mHandlerThread.getLooper())
// AFTER:  builder.setLooper(Looper.getMainLooper())
```

---

### `CarAudioUtils.java` — MODIFIED

```java
// ADDED (moved from CarAudioZonesHelperImpl):
static List<CarAudioDeviceInfo> generateCarAudioDeviceInfos(AudioManagerWrapper am)
static ArrayMap<String, CarAudioDeviceInfo> generateAddressToCarAudioDeviceInfoMap(List<CarAudioDeviceInfo>)
static ArrayMap<String, AudioDeviceInfo> generateAddressToInputAudioDeviceInfoMap(AudioDeviceInfo[])

// ADDED:
static boolean isMicrophoneInputDevice(AudioDeviceInfo info)
static boolean isInvalidActivationPercentage(int percentage)
```

---

### `CarAudioZone.java` — MODIFIED

```java
// updateVolumeDevices() and init() now pass useCoreAudioRouting to volume groups
```

---

### `CarAudioZoneConfig.java` — MODIFIED

```java
// CHANGED signature:
// BEFORE: void updateVolumeDevices()
// AFTER:  void updateVolumeDevices(boolean useCoreAudioRouting)
//         delegates flag down to each group's updateDevices(useCoreAudioRouting)
```

---

### `CarVolumeGroup.java` — MODIFIED

```java
// ADDED:
private static final int EVENT_LOGGER_QUEUE_SIZE = 10;
private final LocalLog mEventLogger = new LocalLog(EVENT_LOGGER_QUEUE_SIZE);

private void logEvent(String event) { mEventLogger.log(event); }

// handleActivationVolume() calls logEvent() when volume changes
// dump() adds EventLogger section

// setMute() refactored into two methods:
// applyMuteLocked() - applies hardware mute
// saveMuteStateToSettingsLocked() - persists to settings

// CHANGED: updateDevices() -> updateDevices() (no param in A16; restored in A17)
```

---

### `ContentObserverFactory.java` — MODIFIED

```java
// BEFORE (A15): no Handler
public ContentObserver createObserver(ContentChangeCallback wrapper) { ... }

// AFTER (A16): explicit Handler from car handler thread
public ContentObserver createObserver(ContentChangeCallback wrapper, Handler handler) {
    return new ContentObserver(handler) { ... };
}
```

---

### `CoreAudioVolumeGroupCallback.java` — MODIFIED

```java
// BEFORE (A15): executor passed at init() time
CoreAudioVolumeGroupCallback(CarVolumeInfoWrapper wrapper, AudioManagerWrapper am) { }
void init(Executor executor) {
    mAudioManager.registerAudioVolumeGroupCallback(executor, this);
}

// AFTER (A16): executor stored in constructor
CoreAudioVolumeGroupCallback(CarVolumeInfoWrapper wrapper, AudioManagerWrapper am,
        Executor executor) {
    mExecutor = executor;
}
void init() {
    mAudioManager.registerAudioVolumeGroupCallback(mExecutor, this);
}
```

---

### `FocusInteraction.java` — MODIFIED

```java
// ContentObserver now uses explicit car handler thread Handler
Handler handler = new Handler(CarServiceUtils.getHandlerThread(...).getLooper());
mContentObserver = mContentObserverFactory.createObserver(callback, handler);

// Removed TimingsTraceLog from evaluateRequest() (not yet added here until A16)
```

---

### `hal/AudioControlWrapper.java` — MODIFIED

```java
// ADDED:
boolean supportsAudioZones()
List<AudioZone> getAudioZones()
AudioDeviceConfiguration getAudioDeviceConfiguration()
```

### `hal/AudioControlWrapperAidl.java` — MODIFIED

```java
// Implements new getAudioZones() AIDL call
@Override
List<AudioZone> getAudioZones() {
    return mAudioControl.getAudioZones();
}
```

### `hal/AudioControlWrapperV1.java` — MODIFIED
Stub — returns empty/unsupported for `supportsAudioZones()` and `getAudioZones()`.

### `hal/AudioControlWrapperV2.java` — MODIFIED
Same stub behavior as V1 for the new zone API.

---

---

## A16 → A17

No new files. All changes are in-place modifications only (patch: ~485 lines).

---

### `CarAudioFocus.java` — MODIFIED

Removes all `TimingsTraceLog` traces added in A15→A16. These paths are now considered stable and the per-call overhead was hurting performance on high-frequency focus requests.

```java
// REMOVED:
- t.traceBegin("evaluate-focus-entry-build");
  FocusEntry focusEntry = new FocusEntry(afi, audioContext, mPackageManager);
- t.traceEnd();
- t.traceBegin("evaluate-focus-result-build");
  FocusEvaluationResult.Builder builder = FocusEvaluationResult.builder(focusEntry);
- t.traceEnd();
// Also removed: "evaluate-focus-losers" and "evaluate-focus-holders" trace blocks
```

---

### `CarAudioService.java` — MODIFIED

```java
// REMOVED: persist-fade-balance feature (consolidated into core fade manager path)
- private boolean mPersistFadeBalanceLevels;
- case AUDIO_FEATURE_PERSIST_FADE_BALANCE_VALUES: return mPersistFadeBalanceLevels;

// CHANGED: AudioServerStateCallback executor
// BEFORE:
mAudioManagerWrapper.setAudioServerStateCallback(new HandlerExecutor(mHandler), callback);
// AFTER:
mAudioManagerWrapper.setAudioServerStateCallback(mContext.getMainExecutor(), callback);

// REMOVED: changeAudioPolicyForConfigChangeLocked() indirection
// BEFORE:
zone.setCurrentCarZoneConfig(zoneConfig);
newAudioPolicy = changeAudioPolicyForConfigChangeLocked();

// AFTER: explicit 3-step sequence (fixes latent bug where user device affinities
//        were not updated for core-audio-routing devices on config switch)
zone.setCurrentCarZoneConfig(zoneConfig);
newAudioPolicy = setupRoutingAudioPolicyLocked();
setAllUserIdDeviceAffinitiesToNewPolicyLocked(newAudioPolicy);
swapRoutingAudioPolicyLocked(newAudioPolicy);
```

---

### `CarAudioVolumeGroup.java` — MODIFIED

```java
// Trivial javadoc fix only:
// android.car.media.CarAudioManager#getVolumeGroupCount()
//   -> CarAudioManager#getVolumeGroupCount()
```

---

### `CarAudioZone.java` — MODIFIED

```java
// updateVolumeDevices() call sites now read flag from CarAudioContext directly
// BEFORE: updateVolumeDevices(externalUseCoreAudioRouting)
// AFTER:  updateVolumeDevices(mCarAudioContext.useCoreAudioRouting())
```

---

### `CarAudioZoneConfig.java` — MODIFIED

```java
// RESTORED: useCoreAudioRouting param (was removed in A16)
// BEFORE (A16): void updateVolumeDevices()
// AFTER (A17):  void updateVolumeDevices(boolean useCoreAudioRouting)
//   delegates flag to each group's updateDevices(useCoreAudioRouting)
```

---

### `CarVolumeGroup.java` — MODIFIED

```java
// REMOVED: event logger (added A15→A16, removed here)
- private static final int EVENT_LOGGER_QUEUE_SIZE = 10;
- private final LocalLog mEventLogger = new LocalLog(EVENT_LOGGER_QUEUE_SIZE);
- private void logEvent(String event) { mEventLogger.log(event); }
// handleActivationVolume() no longer calls logEvent()
// dump() EventLogger section removed

// RESTORED: useCoreAudioRouting param (was no-param in A16)
void updateDevices(boolean useCoreAudioRouting) { ... }
```

---

### `ContentObserverFactory.java` — MODIFIED

```java
// BEFORE (A16): explicit Handler from caller
public ContentObserver createObserver(ContentChangeCallback wrapper, Handler handler) {
    return new ContentObserver(handler) { ... };
}

// AFTER (A17): no Handler param; uses main looper internally
public ContentObserver createObserver(ContentChangeCallback wrapper) {
    return new ContentObserver(new Handler(Looper.getMainLooper())) { ... };
}
```

---

### `CoreAudioVolumeGroup.java` — MODIFIED

```java
// BEFORE (A16): always ran device update (no param)
void updateDevices() {
    for (int i = 0; i < mContextToStrategies.size(); i++) {
        setPreferredDeviceForStrategy(...);
    }
}

// AFTER (A17): skip when core audio routing is active
void updateDevices(boolean useCoreAudioRouting) {
    if (useCoreAudioRouting) {
        return;  // core routing manages device selection itself
    }
    for (int i = 0; i < mContextToStrategies.size(); i++) {
        setPreferredDeviceForStrategy(...);
    }
}
```

---

### `CoreAudioVolumeGroupCallback.java` — MODIFIED

```java
// BEFORE (A16): executor stored in constructor
CoreAudioVolumeGroupCallback(CarVolumeInfoWrapper wrapper, AudioManagerWrapper am,
        Executor executor) { mExecutor = executor; }
void init() { mAudioManager.registerAudioVolumeGroupCallback(mExecutor, this); }

// AFTER (A17): executor injected at init() time (reverts A16)
CoreAudioVolumeGroupCallback(CarVolumeInfoWrapper wrapper, AudioManagerWrapper am) { }
void init(Executor executor) {
    mAudioManager.registerAudioVolumeGroupCallback(executor, this);
}
```

---

### `FocusInteraction.java` — MODIFIED

```java
// ContentObserver now uses main looper (aligns with ContentObserverFactory change)
// BEFORE (A16):
Handler handler = new Handler(CarServiceUtils.getHandlerThread(...).getLooper());
mContentObserver = mContentObserverFactory.createObserver(callback, handler);

// AFTER (A17):
mContentObserver = mContentObserverFactory.createObserver(callback);
// (factory injects Looper.getMainLooper() internally)
```

---

## Cross-Version Oscillation Summary

Several APIs changed direction multiple times — a sign of active in-flight refactoring:

| API | A14→A15 | A15→A16 | A16→A17 |
|-----|---------|---------|---------|
| `ContentObserverFactory.createObserver` | no Handler | explicit Handler | no Handler (main looper) |
| `CoreAudioVolumeGroupCallback.init()` | takes Executor | no-param (executor in ctor) | takes Executor |
| `CarVolumeGroup.updateDevices()` | `(boolean)` | no-param | `(boolean)` |
| `CoreAudioVolumeGroup.updateDevices()` | `(boolean)` | no-param | `(boolean useCoreAudioRouting)` with guard |
| `TimingsTraceLog` in `CarAudioFocus` | absent | added | removed |
| `mPersistFadeBalanceLevels` | absent | added | removed |
| `AudioPolicy Looper` | handler thread | main looper | main looper (kept) |
| `mEventLogger` in `CarVolumeGroup` | absent | added | removed |
