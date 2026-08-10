# Step 3 — OEM gain-callback wiring via the Alliance plugin

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

## What it is

A14 runs gain events through **two parallel paths**:

1. **AOSP's stock HAL-direct path** — already fully present, unmodified, in `a17full`:
   `CarAudioService.setupHalAudioGainCallbackLocked()` (`CarAudioService.java:2532`) registers
   `mCarAudioGainMonitor.registerAudioGainListener(mHalAudioGainCallback)` directly against the AudioControl
   HAL, gated on `audioControlWrapper.supportsFeature(AUDIOCONTROL_FEATURE_AUDIO_GAIN_CALLBACK)`.
2. **A Nissan-added second path**, through the standard AOSP `CarOemProxyService` /
   `OemCarAudioVolumeService.addAudioGainListener()` API — this API itself is confirmed present and
   unmodified in `a17full` (generic AOSP plumbing; `a17full`'s `CarAudioService.java:1196` already
   references `proxy.isOemServiceEnabled()` for the focus/volume/ducking OEM service check). What's missing
   is Nissan's *use* of it for gain events specifically.

Vendor side (unaffected by this step, already present):
`vendor/alliance/services/car/audiocontrol/oemcaraudioservices/.../OemCarAudioVolumeServiceImpl
.addAudioGainListener()` → `VolumeInteractions.addAudioGainListener()`, which runs the plugin's own
`com.alliance.car.oem.audiogain.AudioGainMonitor` (a separate AIDL AudioControl HAL listener living inside
the OEM plugin process) and re-dispatches through the standard `OemCarAudioGainChangedListener` interface
back into `CarAudioService`.

Both callbacks converge on the same downstream logic (`CarAudioZone.onAudioGainChanged(reasons, gains)`) —
they're two independent entry points feeding one sink, not competing implementations.

## `CarAudioGainCallbackHandler.java` — full content, read directly from `a14full` (not paraphrased)

```java
class CarAudioGainCallbackHandler {
    static final String TAG = TAG_AUDIO + ".CarAudioGainCallbackHandler";
    private static final int INVALID_INFO = -1;
    @NonNull private final CarAudioService mCarAudioService;
    private final CarVolumeInfoWrapper mCarVolumeInfoWrapper;
    @NonNull private final SparseArray<CarAudioZone> mCarAudioZones;

    @VisibleForTesting
    CarAudioGainCallbackHandler(CarAudioService carAudioService,
            CarVolumeInfoWrapper carVolumeInfoWrapper, SparseArray<CarAudioZone> carAudioZones) {
        mCarAudioService = Objects.requireNonNull(carAudioService, ...);
        mCarVolumeInfoWrapper = Objects.requireNonNull(carVolumeInfoWrapper, ...);
        mCarAudioZones = Objects.requireNonNull(carAudioZones, ...);
    }

    public void reset() {
        // TODO (b/224885748): handle specific logic on IAudioControl service died event
    }

    private final OemCarAudioGainChangedListener mOemCarAudioGainChangedListener =
        new OemCarAudioGainChangedListener() {
            @Override
            public void onAudioGainChanged(List<Integer> halReasons,
                    List<OemCarAudioGainConfigInfo> gains) {
                // converts OemCarAudioGainConfigInfo -> CarAudioGainConfigInfo, then:
                mCarAudioService.onAudioGainChanged(halReasons, carGainInfos);
            }
        };

    void startListeningForAudioGainChanges() {
        CarLocalServices.getService(CarOemProxyService.class)
                .getCarOemAudioVolumeService()
                .addAudioGainListener(mOemCarAudioGainChangedListener);
    }

    // Called from CarAudioService.onAudioGainChanged(); groups gains by zone, calls
    // CarAudioZone.onAudioGainChanged(reasons, gains) per zone, collects CarVolumeGroupEvents,
    // then mCarVolumeInfoWrapper.onVolumeGroupEvent(events).
    void handleAudioGainChanged(List<Integer> reasons, List<CarAudioGainConfigInfo> gains) { ... }
}
```

The full chain is: OEM plugin → `OemCarAudioGainChangedListener.onAudioGainChanged()` → converts to
`CarAudioGainConfigInfo` → calls `CarAudioService.onAudioGainChanged(reasons, gains)` → (inside
`CarAudioService`) dispatches to `mCarAudioGainCallbackHandler.handleAudioGainChanged(...)` → groups by
zone → `CarAudioZone.onAudioGainChanged(reasons, gainsForZone)` per zone → collects
`CarVolumeGroupEvent`s → `mCarVolumeInfoWrapper.onVolumeGroupEvent(events)`.

## Files to change in `a17full`

- **Add** `CarAudioGainCallbackHandler.java` (new class, content above) to
  `packages/services/Car/service/src/com/android/car/audio/`. The `CarAudioService`/
  `CarVolumeInfoWrapper`/`SparseArray<CarAudioZone>` constructor parameters are already the exact types
  `a17full`'s `CarAudioService.java` already constructs (e.g. `new CarVolumeInfoWrapper(this)` is used
  verbatim in the existing `setupHalAudioGainCallbackLocked()` at `CarAudioService.java:2532` — reuse that
  same pattern here).
- **Modify** `CarAudioService.java`:
  - Add `setupOemAudioGainListener()` (constructs `CarAudioGainCallbackHandler` and calls
    `startListeningForAudioGainChanges()`), activated once `mCarOemProxyServiceCallback.onOemServiceReady()`
    fires and `proxy.isOemServiceEnabled()` is true. Reuse the exact readiness gate already referenced at
    `CarAudioService.java:1196` for the focus/volume/ducking OEM service check — don't invent a new
    readiness signal.
  - Add an `onAudioGainChanged(List<Integer> reasons, List<CarAudioGainConfigInfo> gains)` entry point that
    forwards to `mCarAudioGainCallbackHandler.handleAudioGainChanged(reasons, gains)` — this is the method
    the OEM listener calls into, separate from the existing HAL-direct gain callback path.
- **No change needed** to `CarAudioZone.onAudioGainChanged(reasons, gains)` — both the HAL-direct path
  (via `CarAudioGainMonitor`) and this new OEM path converge there already in current A17 stock code.
- `CarAudioGainMonitor.java` (stock AOSP, already present, unmodified) also defines
  `shouldBlockVolumeRequest`/`shouldLimitVolume`/`shouldDuckGain`/`shouldMuteVolumeGroup`/
  `REASONS_TO_EXTRA_INFO` policy tables consumed elsewhere in `CarAudioService` — no change needed there,
  just confirming they're shared correctly by both gain-event entry points.

## Risk

Low — pure addition, no AOSP code to reconcile against, since Google never touched this area in A15-A17.
The only failure mode is a duplicate/conflicting readiness check if `setupOemAudioGainListener()` doesn't
reuse the existing `onOemServiceReady()`/`isOemServiceEnabled()` gate correctly.

## Verify

- `adb logcat` filtered on the Alliance plugin's gain-callback path; confirm gain-change events arrive at
  `CarAudioZone.onAudioGainChanged` via the **OEM path** (not just the pre-existing HAL-direct path) once
  wiring is in place.
- Confirm no duplicate/double-application of a single gain change (i.e. the HAL-direct and OEM paths aren't
  both firing for the same underlying event in a way that double-applies a gain adjustment).
- Confirm the listener registers only after `onOemServiceReady()` — no crash/NPE if the OEM proxy service
  isn't ready yet at `CarAudioService` construction time.
