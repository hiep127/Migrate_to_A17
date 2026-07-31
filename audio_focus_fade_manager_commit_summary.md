# Audio Focus & Fade Manager — Deep Dive (22 commits)

**Repo:** `packages/services/Car`, path `service/src/com/android/car/audio/...`
**Where this lives today:** these 22 commits are **not** on `nissan_u_ccs2_release`
(verified: `git merge-base --is-ancestor <commit> HEAD` fails for all of them).
They *are* on `remotes/origin/alliance_u_migration_a17_260703` — i.e. this is
part of the AOSP/LAMP baseline that rides in with the **A17 migration**, not
something already running on the current A14 Nissan build.

Everything below was read from the actual diffs and the resulting file
contents at the tip of this set (`f5f80fd01e`), not from commit messages.

---

## 1. The architecture change: one audio policy becomes four

Before this set, `CarAudioService` built its `AudioPolicy` objects in a
handful of places. After it, `CarAudioService.java` has four clearly
separated setup methods, each registering its **own** `AudioPolicy` with
`AudioManager`:

- **`setupRoutingAudioPolicyLocked()`** — dynamic audio routing + mirror
  device policy (only when `mUseDynamicRouting` is on). This can be torn
  down and rebuilt on its own when zone configuration changes, without
  touching focus or volume.
- **`setupVolumeControlAudioPolicyLocked()`** — registers
  `CarAudioPolicyVolumeCallback` for group volume changes and mute.
- **`setupFocusControlAudioPolicyLocked()`** — builds `CarZonesAudioFocus`
  (via `AudioManagerWrapper`), sets it as the policy's
  `AudioPolicyFocusListener`, and calls `builder.setIsAudioFocusPolicy(true)`.
- **`setupFadeManagerConfigAudioPolicyLocked()`** — a *fourth*, otherwise
  empty `AudioPolicy` whose only job is to carry the **default fade
  manager configuration** via
  `AudioPolicy.setFadeManagerConfigurationForFocusLoss(...)`.

This is the `f274f92e0e` ("Separate audio policy for audio focus, volume,
and routing") change, later extended by the fade-manager commits to add
the fourth policy. Splitting these out means a zone-configuration reload
only has to recreate the routing policy — focus and volume keep running
uninterrupted.

## 2. End-to-end flow: what happens when an app requests audio focus

1. **App calls `AudioManager.requestAudioFocus(...)`.** Because
   `CarZonesAudioFocus` registered itself as `setIsAudioFocusPolicy(true)`,
   the platform routes the request to
   `CarZonesAudioFocus.onAudioFocusRequest(AudioFocusInfo, int)` instead of
   the default `MediaFocusControl` engine.

2. **Null-service guard (fix from `3d717d8185`).** `CarZonesAudioFocus` now
   keeps `mCarAudioService` / `mAudioPolicy` behind a `mLock` and, on every
   entry point (`onAudioFocusRequest`, `onAudioFocusAbandon`,
   `reevaluateAndRegainAudioFocus[List]`), snapshots the service reference
   and bails out with a logged error if it's `null`. Before this fix, a
   focus message already sitting on the looper queue when
   `CarZonesAudioFocus` was unregistered (service/policy set to `null`)
   would NPE; now it's a clean no-op.

3. **Zone dispatch.** `CarZonesAudioFocus` resolves the audio zone for the
   request and forwards to the per-zone `CarAudioFocus.onAudioFocusRequest`.

4. **`CarAudioFocus.evaluateFocusRequestLocked(afi)`** does the actual
   work:
   - Checks for duplicate-client / restricted-focus / notification-vs-
     exclusive edge cases (unchanged pre-existing logic).
   - Calls `evaluateFocusRequestLocked(replacedCurrentEntry, afi)`, which
     internally picks one of two paths:
     - **`evaluateFocusRequestExternallyLocked`** — if an **OEM audio
       focus plugin** is registered, the evaluation (who wins/loses, and
       any fade configs) is delegated to it.
     - **`evaluateFocusRequestInternallyLocked`** — the built-in engine,
       which calls into `FocusInteraction.evaluateRequest(...)` for every
       existing holder/loser.

5. **`FocusInteraction.evaluateRequest(requestedUsage, focusHolder, ...)`**
   (fixed by `01d855c04a`) — see §6 below for what changed here.

6. **Fade config resolution**, only when
   `isFadeManagerSupported()` is true (i.e. the OEM/config declared
   `AUDIO_FEATURE_FADE_MANAGER_CONFIGS` via `CarAudioFeaturesInfo` —
   see §5):
   - `defaultCarAudioFadeConfigFromXml` = the zone config's default fade
     config (`CarAudioZoneConfig.getDefaultCarAudioFadeConfiguration()`).
   - `transientCarAudioFadeConfigFromXml` = the fade config, if any,
     registered against the **new/gaining** request's audio attributes
     (`getCarAudioFadeConfigurationForAudioAttributes(newEntryAfi.getAttributes())`)
     — this is exactly the `b7a9da6581` change: the transient lookup key
     is the **incoming/gaining** request, not the app about to lose focus.
   - `transientCarAudioFadeConfigsFromOemService` = whatever
     `evaluationResults.getAudioAttributesToCarAudioFadeConfigurationMap()`
     returned. If the OEM plugin supplied entries here, they win outright
     (`getOptimalUsageBasedTransientFadeConfig`: OEM map first, XML
     transient as fallback — nothing from the config file is used once the
     OEM map is non-empty, per `d5d4b1269e`).
   - `getTransientFadeManagerConfig(...)` then picks, in order: the
     resolved transient config → else the zone's default config (skipped
     entirely for the **primary zone**, since the core framework already
     applies the primary zone's default via the fourth `AudioPolicy` from
     §1) → else `null` (falls back to old abrupt behavior).

7. **Fadability gate (`005104f3ec`).** Even with a config in hand, a fade
   is only requested when the loss is a genuine, permanent
   `AUDIOFOCUS_LOSS` **and** the losing entry wasn't already ducked
   (`!entry.isDucked()`). Every transient loss
   (`AUDIOFOCUS_LOSS_TRANSIENT[_CAN_DUCK]`) and every delayed-focus
   abandonment passes `shouldFade = false` explicitly — fading is never
   applied outside a real, permanent loss.

8. **Dispatch.** `sendFocusLossLocked(loser, lossType, winner, shouldFade,
   transientFadeManagerConfig)`:
   - if fade manager is supported **and** `shouldFade` → builds the list
     of other currently-active `AudioFocusInfo`s (`otherActiveAfis`,
     minus the loser, plus the winner) and calls
     `AudioManagerWrapper.dispatchAudioFocusChangeWithFade(loser, lossType,
     mAudioPolicy, otherActiveAfis, transientFadeManagerConfig)` — the new
     platform API (`AudioManager.dispatchAudioFocusChangeWithFade`,
     introduced in `307260fac3`) that fades the loser out instead of an
     abrupt stop.
   - otherwise → the old `dispatchAudioFocusChange(loser, lossType,
     mAudioPolicy)`.

## 3. Fade config file format (new — `76f3db958d`, `8f0eaf6487`)

Two independent XML surfaces were added:

**a) `car_audio_fade_configuration.xml`** (new overlay, parsed by the new
class `CarAudioFadeConfigurationHelper.java`):

```xml
<carAudioFadeConfiguration version="1">
  <configs>
    <config name="my_fade_config"
            defaultFadeOutDurationInMillis="1000"
            defaultFadeInDurationInMillis="500">
      <fadeState value="..."/>
      <fadeableUsages> <usage value="USAGE_MEDIA"/> ... </fadeableUsages>
      <unfadeableContentTypes> ... </unfadeableContentTypes>
      <unfadeableAudioAttributes> <audioAttributes .../> </unfadeableAudioAttributes>
      <fadeOutConfigurations>
        <fadeConfiguration fadeDurationMillis="2000">
          <audioAttributes> <usage value="USAGE_NAVIGATION_GUIDANCE"/> </audioAttributes>
        </fadeConfiguration>
      </fadeOutConfigurations>
      <fadeInConfigurations> ... same shape ... </fadeInConfigurations>
    </config>
  </configs>
</carAudioFadeConfiguration>
```

Each `<config name=...>` is parsed into an `android.car.oem.CarAudioFadeConfiguration`
wrapping a platform `FadeManagerConfiguration`, keyed by name in an
`ArrayMap<String, CarAudioFadeConfiguration>`. Volume shapers are
explicitly **not** supported (only durations/usages/content-types).

**b) `car_audio_configuration.xml` bumped to version 4** (parsed by
`CarAudioZonesHelper.java`): inside a zone-config block, a new
`<applyFadeConfigs>` element can reference fade configs **by name**:

```xml
<zoneConfig name="...">
  ...
  <applyFadeConfigs>
    <fadeConfig name="my_fade_config"/>                 <!-- no nested <audioAttributes> -> this zone config's DEFAULT -->
    <fadeConfig name="nav_transient_fade">                <!-- has nested <audioAttributes> -> TRANSIENT, scoped to those attrs -->
      <audioAttributes> <usage value="USAGE_NAVIGATION_GUIDANCE"/> </audioAttributes>
    </fadeConfig>
  </applyFadeConfigs>
</zoneConfig>
```

`CarAudioZonesHelper.parseFadeConfig()` looks the name up in the
`CarAudioFadeConfigurationHelper` loaded from file (a); an unknown name
throws (`Preconditions.checkArgument` in `validateFadeConfig`), and
`<applyFadeConfigs>` itself throws if used under schema version < 4. The
resolved configs are stored on `CarAudioZoneConfig` via
`setDefaultCarAudioFadeConfiguration()` / a per-`AudioAttributes` map
(`setCarAudioFadeConfigurationForAudioAttributes()`) — exactly the two
fields read back in step 6 above. If the feature flag is off,
`CarAudioZoneConfig.Builder.build()` clears both fields regardless of what
was parsed, so a disabled feature never leaks a stale config.

## 4. Primary-zone special case

`setupFadeManagerConfigAudioPolicyLocked()` registers a bare `AudioPolicy`
purely to hold the *default* fade config for the **primary zone** via
`AudioPolicy.setFadeManagerConfigurationForFocusLoss(...)` — this is set
once at startup (`updateFadeManagerConfigurationForPrimaryZoneLocked`) and
refreshed whenever the primary zone's active configuration changes. This
is why `CarAudioFocus.getTransientFadeManagerConfig()` explicitly skips
returning the zone default when `mCarAudioZone.isPrimaryZone()` — the core
audio framework is already applying it at the policy level; re-applying
it as a "transient" from `CarAudioFocus` would be redundant. Non-primary
zones don't get this treatment, so `CarAudioFocus` supplies their default
fade config itself, per-request.

## 5. Feature gating — three ANDed conditions

`mUseFadeManagerConfiguration` in `CarAudioService` is **only** true when
all of these hold (`257815f097`):
```
enableFadeManagerConfiguration()          // android.media.audiopolicy Aconfig flag
&& carAudioFadeManagerConfiguration()     // android.car.feature Aconfig flag
&& R.bool.audioUseFadeManagerConfiguration  // RRO overlay (default false)
```
Plus a hard guard: `Preconditions.checkArgument(!(runInLegacyMode() &&
mUseFadeManagerConfiguration), ...)` — the feature cannot be turned on for
a car service running in legacy (non-dynamic-routing) mode; enabling it
there throws at construction time.

A second, independent RRO — `R.bool.audioUseIsolatedAudioFocusForDynamicDevices`
(`f51b2b9f1a`) — was added in the same window to eventually isolate focus
per dynamic device, but in this commit set it only wires the flag through
`CarAudioFeaturesInfo` (`AUDIO_FEATURE_ISOLATED_DEVICE_FOCUS`) and adds it
to the service dump; no isolation behavior ships yet in these 22 commits.

## 6. `FocusInteraction` correctness fix (`01d855c04a`)

**Before:** the interaction matrix was consulted using whatever car-audio
context the *OEM's* audio context configuration produced for a usage. If
an OEM defined only one context (say, "media") covering several distinct
usages, all of those usages got treated as the same context — so two
completely different usages could incorrectly interact as "media vs.
media."

**After:** `FocusInteraction.evaluateRequest(int requestedUsage,
FocusEntry focusHolder, ...)` now takes the *raw* `AudioAttributes`
system usage for both sides, and only inside
`getFocusInteractionLocked(requestedUsage, holderUsage)` are they each
converted via `CarAudioContext.getLegacyContextForUsage(...)` before
indexing into `INTERACTION_MATRIX` (which is still keyed by the fixed set
of legacy contexts: `MUSIC`, `NAVIGATION`, `CALL`, `ALARM`, etc.). The OEM
context grouping never enters the interaction decision anymore — it's
purely a legacy-context lookup per usage, decoupled from however the OEM
chose to bucket usages for its own config.

## 7. Race-condition fix in `CarZonesAudioFocus` (`3d717d8185`)

Covered in step 2 above — `mCarAudioService` and `mAudioPolicy` moved
behind `@GuardedBy("mLock")`, and all four call sites
(`onAudioFocusRequest`, `onAudioFocusAbandon`,
`reevaluateAndRegainAudioFocus`, `reevaluateAndRegainAudioFocusList`) now
snapshot-and-null-check the service before touching zone state. This
closes a real crash: a focus message queued on the looper right as
`CarZonesAudioFocus` was being unregistered (service/policy nulled out)
would previously NPE.

## 8. HAL audio focus: activation volume (`fb15a31d8e`, `eba27c1c7e`)

In `hal/HalAudioFocus.java`, `makeAudioFocusRequestLocked()` — used when
the **HAL** (AudioControl HAL, i.e. an in-vehicle amp/DSP asking for
focus on behalf of a hardware source) requests focus — now calls
`handleNewlyActiveHalPlayback(attributes, zoneId)` whenever the request is
freshly **granted**:
```java
private void handleNewlyActiveHalPlayback(AudioAttributes attributes, int zoneId) {
    if (mCarAudioPlaybackMonitor == null) return;
    long identity = Binder.clearCallingIdentity();
    try {
        mCarAudioPlaybackMonitor.onActiveAudioPlaybackAttributesAdded(
            List.of(new Pair<>(attributes, Binder.getCallingUid())), zoneId);
    } finally {
        Binder.restoreCallingIdentity(identity);
    }
}
```
`onActiveAudioPlaybackAttributesAdded` is what actually applies the
min/max **activation volume** to the matching car volume group elsewhere
in `CarAudioPlaybackMonitor`/`CarVolumeGroup` (pre-existing machinery,
reused here for the HAL path). The `Binder.clearCallingIdentity()` /
`restoreCallingIdentity()` wrapper is the `eba27c1c7e` fix: without it,
this call runs under the AudioControl HAL binder thread's identity, which
doesn't hold `Car.PERMISSION_CAR_CONTROL_AUDIO_VOLUME` — so the
activation-volume callback would fail permission checks. Delayed HAL
focus grants explicitly skip this (`AUDIOFOCUS_REQUEST_DELAYED` branch
just marks the result as a loss; HAL doesn't support delayed focus).

## 9. `AudioManagerWrapper` refactor (`f5f80fd01e`, `53fd09fa85`)

`AudioManagerWrapper` already existed before these commits (used for
volume calls — `getMinVolumeIndexForAttributes`, `setVolumeGroupVolumeIndex`,
etc.). These two commits add five **focus**-related passthrough methods to
it —
`abandonAudioFocusRequest`, `requestAudioFocus`, `dispatchAudioFocusChange`,
`dispatchAudioFocusChangeWithFade`, `setFocusRequestResult` — and switch
`CarAudioFocus`, `CarZonesAudioFocus.createCarZonesAudioFocus(...)`, and
`hal/HalAudioFocus` to take an `AudioManagerWrapper` instead of a raw
`AudioManager`. Purely mechanical (flagged `EXEMPT refactor`), but it
means focus dispatch can now be unit-tested/mocked the same way volume
calls already were, instead of needing a real `AudioManager`.

## 10. Diagnostics (`6b2919721d`, `892cf976bd`)

Adds proto-based `dump()` support for focus/ducking/muting
(`audio_dump.proto` +85 lines, new `CarDuckingInfo.java`), later refined
so the HAL focus dump reports full `AudioAttributes` instead of just the
bare usage int. The fade-manager commits extend the same proto
(`CarAudioProtoUtils.java`, `CarAudioFadeConfigurationHelper.dumpProto()`,
`CarAudioZoneConfig.dumpProto()`) so `dumpsys car_service --proto` can
show which fade configs are loaded and assigned per zone config.

---

## Quick reference: file → concrete change

| File | What actually changed |
|---|---|
| `AudioManagerWrapper.java` | +5 focus methods (`requestAudioFocus`, `abandonAudioFocusRequest`, `dispatchAudioFocusChange`, `dispatchAudioFocusChangeWithFade`, `setFocusRequestResult`) added to the pre-existing volume wrapper |
| `CarAudioFadeConfigurationHelper.java` *(new)* | XML parser for `car_audio_fade_configuration.xml` → `Map<String, CarAudioFadeConfiguration>` |
| `CarAudioProtoUtils.java` *(new)* | proto dump helpers for `CarAudioFadeConfiguration` |
| `CarAudioFocus.java` | `mAudioManager` field retyped to `AudioManagerWrapper`; `sendFocusLossLocked` gains fade dispatch + fadability gate; `evaluateFocusRequestLocked` resolves default/transient (XML + OEM) fade configs per request; `getTransientFadeManagerConfig`/`getOptimalUsageBasedTransientFadeConfig` added |
| `CarZonesAudioFocus.java` | `createCarZonesAudioFocus(...)` takes `AudioManagerWrapper`; `mCarAudioService`/`mAudioPolicy` moved behind `mLock`; null-service guard added to 4 entry points |
| `FocusInteraction.java` | `evaluateRequest` takes raw usage ints; conversion to legacy `CarAudioContext` moved inside `getFocusInteractionLocked` (was: OEM context used directly) |
| `CarAudioContext.java` | supporting changes for `getLegacyContextForUsage` used by the `FocusInteraction` fix |
| `hal/HalAudioFocus.java` | `mAudioManager` retyped to `AudioManagerWrapper`; `handleNewlyActiveHalPlayback()` added (activation volume trigger), wrapped in `clearCallingIdentity`/`restoreCallingIdentity`; dump switched to full `AudioAttributes` |
| `CarAudioZoneConfig.java` | new fields `mIsFadeManagerConfigurationEnabled`, `mDefaultCarAudioFadeConfiguration`, `mAudioAttributesToCarAudioFadeConfiguration` + getters/builder setters; feature-flag-off path clears both |
| `CarAudioZonesHelper.java` | schema version bumped to 4; `parseApplyFadeConfigs`/`parseFadeConfig`/`parseTransientFadeConfigs`/`validateFadeConfig` added |
| `CarAudioService.java` | policy setup split into `setupRoutingAudioPolicyLocked` / `setupVolumeControlAudioPolicyLocked` / `setupFocusControlAudioPolicyLocked` / `setupFadeManagerConfigAudioPolicyLocked`; `mUseFadeManagerConfiguration` (3-way gate) and `mUseIsolatedFocusForDynamicDevices` added; `getAudioFeaturesInfo()` builds the `CarAudioFeaturesInfo` passed down to `CarAudioFocus`/`HalAudioFocus` |
| `res/values/config.xml`, `overlayable.xml` | new RRO bool `audioUseIsolatedAudioFocusForDynamicDevices` (`audioUseFadeManagerConfiguration` already existed, now actually read) |
| `service/proto/.../audio_dump.proto` | extended repeatedly: focus/ducking/muting dump, then fade-config dump, then `USE_FADE_MANAGER_CONFIGURATION` / isolated-focus fields |
| `CarDucking.java`, `CarDuckingInfo.java` *(new)*, `CarVolumeGroupMuting.java`, `FocusEntry.java` | proto dump plumbing only (`6b2919721d`) |

## New platform/OEM APIs this code depends on

- `AudioManager.dispatchAudioFocusChangeWithFade(AudioFocusInfo, int, AudioPolicy, List<AudioFocusInfo>, FadeManagerConfiguration)`
- `AudioPolicy.setFadeManagerConfigurationForFocusLoss(FadeManagerConfiguration)`
- `android.media.FadeManagerConfiguration` / `.Builder`
- `android.car.oem.CarAudioFadeConfiguration` / `.Builder`
- `android.car.oem.CarAudioFeaturesInfo` — `AUDIO_FEATURE_FADE_MANAGER_CONFIGS`, `AUDIO_FEATURE_ISOLATED_DEVICE_FOCUS`
- `android.car.oem.OemCarAudioFocusResult.getAudioAttributesToCarAudioFadeConfigurationMap()` (OEM plugin → transient fade configs)
- Aconfig flags: `android.media.audiopolicy.Flags.enableFadeManagerConfiguration`, `android.car.feature.Flags.carAudioFadeManagerConfiguration`

All of the above are new-to-this-codebase surface — if the A17 migration
brings these commits in, anything currently building against the A14
`CarAudioFocus`/`AudioManagerWrapper`/`HalAudioFocus` constructors needs to
be checked, since their constructor signatures changed (raw `AudioManager`
→ `AudioManagerWrapper`, plus the new `CarAudioFeaturesInfo` parameter).
