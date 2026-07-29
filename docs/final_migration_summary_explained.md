# Migrating Car Audio from Android 14 to Android 17 — Explained in Plain Terms

*A readable walkthrough of what the A14→A17 upgrade means for the Nissan audio code. The detailed,
evidence-backed version with file tables and risk flags is in `final_migration_summary_a14_to_a17.md`;
this file tells the same story in prose.*

---

## The situation in one paragraph

Our car-audio code is sitting at the Android 14 baseline, with a stack of Nissan-specific fixes layered
on top. Between Android 14 and 17, Google rewrote large parts of that same code — adding new features,
splitting classes apart, and moving things around. None of that has been brought into our tree yet. So
the migration is essentially: **take three years of Google's changes and merge them into code we've
already customized.** The hard part isn't the volume of new code — it's that Google rewrote the exact
same files we've been patching.

---

## What actually changes in behavior (Android 14 → 17)

Setting aside the code churn, here is what changes in how the audio system *behaves*. These are the
things worth understanding, because a few of them overlap with features we already built ourselves.

- **The system can now fade apps out when they lose audio focus.** Before, if an app ignored the
  "you lost focus" signal, it just kept playing and Android couldn't easily silence it. Now Android can
  smoothly fade that app down on its own, without the app's cooperation. This is a **new, optional
  capability** — it's *not* related to our **Audio Off** (which just mutes entertainment); the two do
  different jobs and can coexist. The only wrinkle for us is that Google's fade code lives in the same
  focus files we've already patched, so it's a **merge** to manage — not a behavior clash.

  <details><summary><b>How it works in code (A15 implementation example)</b></summary>

  **1. The focus dispatch gains a fade path** (`CarAudioFocus.sendFocusLossLocked`). Today it just
  tells the app it lost focus; A15 adds a branch that fades the loser out itself when a fade config is
  available:
  ```java
  // BEFORE (A14) — advisory only: ask the app to stop
  private void sendFocusLossLocked(AudioFocusInfo loser, int lossType) {
      mAudioManager.dispatchAudioFocusChange(loser, lossType, mAudioPolicy);
  }

  // AFTER (A15) — enforce a fade-out on the loser without its cooperation
  private void sendFocusLossLocked(AudioFocusInfo loser, int lossType,
          AudioFocusInfo winner, boolean shouldFade,
          FadeManagerConfiguration transientFadeManagerConfig) {
      if (shouldFade && isFadeManagerSupported()) {
          mAudioManager.dispatchAudioFocusChangeWithFade(
                  loser, lossType, transientFadeManagerConfig, winner, List.of());
          return;                                   // faded to silence by the framework
      }
      mAudioManager.dispatchAudioFocusChange(loser, lossType, mAudioPolicy);  // fallback
  }
  ```

  **2. The fade behavior is described by a `FadeManagerConfiguration`** — which usages fade, how long,
  and what must never be faded (e.g. speech):
  ```java
  FadeManagerConfiguration fade = new FadeManagerConfiguration.Builder()
          .setFadeableUsages(List.of(USAGE_MEDIA, USAGE_GAME))
          .setFadeOutDurationForUsage(USAGE_MEDIA, /* ms */ 500)
          .setFadeInDurationForUsage(USAGE_MEDIA,  /* ms */ 2000)
          .setUnfadeableContentTypes(List.of(CONTENT_TYPE_SPEECH))  // don't chop nav/voice
          .build();
  ```

  **3. On the device it's turned on by a flag + an XML file.** The flag
  `audioUseFadeManagerConfiguration` enables it, and `CarAudioFadeConfigurationHelper` parses
  `/vendor/etc/car_audio_fade_configuration.xml`:
  ```xml
  <carAudioFadeConfiguration>
    <fadeConfiguration name="default_fade"
        defaultFadeOutDurationInMillis="500" defaultFadeInDurationInMillis="2000">
      <fadeableUsages>
        <usage value="AUDIO_USAGE_MEDIA"/>
        <usage value="AUDIO_USAGE_GAME"/>
      </fadeableUsages>
      <unfadeableContentTypes>
        <contentType value="AUDIO_CONTENT_TYPE_SPEECH"/>
      </unfadeableContentTypes>
    </fadeConfiguration>
  </carAudioFadeConfiguration>
  ```

  **What it does under the hood:** `dispatchAudioFocusChangeWithFade` applies a `VolumeShaper` (a volume
  envelope) to the losing app's player and ramps it to zero. The app's `AudioTrack` keeps running — it's
  just made inaudible — so a misbehaving app that ignores its focus-loss callback still goes quiet.

  *For us:* the only file we'd merge here is `CarAudioFocus` (plus the new `CarAudioFadeConfigurationHelper`
  is a brand-new file, no conflict). Whether we ship a `car_audio_fade_configuration.xml` at all is the
  on/off switch — leaving the flag `false` keeps today's behavior exactly.

  </details>

- **Volume can be clamped into a safe range when audio starts.** When a sound kicks in from silence,
  Android can now force the starting volume to sit between a minimum and maximum. This is essentially the
  same idea as our **MaxVolumeStartup** feature — so we'll have two mechanisms doing the same job unless
  we reconcile them.

  <details><summary><b>How it works in code (A15 implementation example)</b></summary>

  **1. A new `CarActivationVolumeConfig`** holds the range and *when* to apply it:
  ```java
  new CarActivationVolumeConfig(invocationType, /*min%*/ 10, /*max%*/ 80);
  // invocationType: ACTIVATION_VOLUME_ON_BOOT | ON_SOURCE_CHANGED | ON_PLAYBACK_CHANGED
  ```

  **2. `CarAudioPlaybackMonitor` notices playback starting** and asks the group to clamp itself:
  ```java
  void onActiveAudioPlaybackAttributesAdded(
          List<Pair<AudioAttributes,Integer>> active, int zoneId) {
      int groupId = mCarAudioZones.get(zoneId)
              .getVolumeGroupForAudioAttributes(pair.first);
      handleActivationVolumeForGroup(zoneId, groupId, ACTIVATION_VOLUME_ON_PLAYBACK_CHANGED);
  }
  // → CarVolumeGroup.handleActivationVolume() clamps the index between
  //   getMinActivationGainIndex() and getMaxActivationGainIndex()
  ```

  **3. Turned on by the flag `audioUseMinMaxActivationVolume` + XML v4:**
  ```xml
  <volumeGroup name="music_group">
    <activationVolumeConfig name="media_activation">
      <activationVolumeConfigEntry minActivationVolumePercentage="10"
          maxActivationVolumePercentage="80" invocationType="onPlaybackChanged"/>
    </activationVolumeConfig>
  </volumeGroup>
  ```

  *For us:* this **overlaps MaxVolumeStartup** (which lives in `CarVolumeGroup`/`CarAudioGainMonitor`).
  Pick one, or the boot volume gets clamped twice. Leaving the flag `false` keeps our version in charge.

  </details>

- **Routing is smarter about disconnected devices.** Only the currently-active audio layout gets set up,
  with the default layout always kept as a fallback, so audio recovers cleanly when a device (say USB or
  Bluetooth) drops.

  <details><summary><b>How it works in code (A15 implementation example)</b></summary>

  `CarAudioDynamicRouting.setupAudioDynamicRouting` used to register every zone config; A15 registers
  only the selected+active one and always keeps the default as a fallback:
  ```java
  // BEFORE (A14): route every config unconditionally
  for (CarAudioZoneConfig config : zoneConfigs) {
      setupAudioDynamicRoutingForZoneConfig(builder, config, ...);
  }

  // AFTER (A15): only the active config, default always registered last as fallback
  CarAudioZoneConfig defaultConfig = null;
  for (CarAudioZoneConfig config : zoneConfigs) {
      if (config.isDefault()) { defaultConfig = config; continue; }
      if (!config.isSelected() || !config.isActive()) { continue; }
      setupAudioDynamicRoutingForZoneConfig(builder, config, ...);
  }
  setupAudioDynamicRoutingForZoneConfig(builder, defaultConfig, ...);  // fallback path
  ```

  *For us:* low overlap — we're stock A14 here, so this applies cleanly. It mainly matters if we ever use
  multiple zone configs; with a single config the effect is just "default is always there."

  </details>

- **The service recovers if the audio server crashes**, and starts up more carefully if the audio server
  isn't ready yet at boot.

  <details><summary><b>How it works in code (A15/A16 implementation example)</b></summary>

  A new `CarAudioServerStateCallback` (implements `AudioManager.AudioServerStateCallback`) lets the
  service notice the audioserver going down/up, and `init()` no longer assumes it's already running:
  ```java
  // init() defers setup if the audioserver isn't up yet
  if (!mAudioManager.isAudioServerRunning()) {
      // don't build policies / volume groups yet — wait for the server
      return;
  }
  ...
  // On audioserver restart, tear down and re-initialize cleanly
  void releaseAudioCallbacks(boolean isAudioServerDown) { ... }
  // callback:
  public void onAudioServerDown()  { releaseAudioCallbacks(/*isAudioServerDown*/ true); }
  public void onAudioServerUp()    { setupControlAndRoutingAudioPoliciesLocked(); /* re-init */ }
  ```

  *For us:* stock A14, so it merges cleanly — but **re-check our boot/Audio-Off-persist init order**
  against the new "defer until server up" timing so our startup hooks still fire at the right moment.

  </details>

- **The car's zone layout can optionally come from the HAL** instead of the XML file. (This is the
  "config on HAL" topic — covered in its own document. Short answer: we should keep using the XML file.)

  <details><summary><b>How it works in code (A16 implementation example)</b></summary>

  `CarAudioZonesHelper` became an interface with two implementations, chosen at boot by asking the HAL:
  ```java
  boolean halSupportsZoneConfig = mAudioControlWrapper.supportsAudioZones();
  mCarAudioZonesHelper = halSupportsZoneConfig
          ? new CarAudioZonesHelperAudioControlHAL(mAudioControlWrapper, ...) // getAudioZones() from HAL
          : new CarAudioZonesHelperImpl(mContext, ...);                       // our car_audio_configuration.xml
  ```

  *For us:* take the interface refactor (it comes with A16), but keep `supportsAudioZones()` returning
  **false** so the XML implementation stays in charge. Note our AudioControl HAL is only at v3 and doesn't
  even define `getAudioZones()` yet — see the config-on-HAL docs for why we defer this.

  </details>

- **Threading was cleaned up** — audio work moved onto the main thread, and a background helper thread
  was removed. Mostly invisible, but we should re-check the timing of our callbacks.

  <details><summary><b>How it works in code (A16/A17 implementation example)</b></summary>

  Audio policies and callbacks moved from a dedicated `HandlerThread` to the main looper/executor:
  ```java
  // BEFORE (A16): dedicated background thread
  builder.setLooper(mHandlerThread.getLooper());
  var executor = new HandlerExecutor(mHandler);
  mAudioManagerWrapper.setAudioServerStateCallback(executor, callback);

  // AFTER (A17): main looper / main executor
  builder.setLooper(Looper.getMainLooper());
  mAudioManagerWrapper.setAudioServerStateCallback(mContext.getMainExecutor(), callback);
  ```

  *For us:* re-test the **timing** of our callbacks that run here — the OEM gain callback and the
  master-mute callback — since they now dispatch on the main thread instead of the old helper thread.

  </details>

- **Some things were added and then removed again** across the versions (debug logging, tracing, a
  fade-balance feature). Since we're jumping straight from 14 to 17, we should land on the final
  17 result and skip the back-and-forth in between.

  <details><summary><b>What these were (A16-added, A17-removed)</b></summary>

  Three examples of code that appears in A16 and is gone by A17 — so on a direct 14→17 jump we add
  **none** of them:
  ```java
  // 1. Per-group event logger in CarVolumeGroup — added A16, removed A17
  private static final int EVENT_LOGGER_QUEUE_SIZE = 10;
  private final LocalLog mEventLogger = new LocalLog(EVENT_LOGGER_QUEUE_SIZE);

  // 2. Persist-fade-balance feature in CarAudioService — added A16, removed A17
  private boolean mPersistFadeBalanceLevels;
  case AUDIO_FEATURE_PERSIST_FADE_BALANCE_VALUES: return mPersistFadeBalanceLevels;

  // 3. TimingsTraceLog around focus evaluation in CarAudioFocus — added A16, removed A17
  t.traceBegin("evaluate-focus-entry-build"); ... t.traceEnd();
  ```

  *For us:* if we merge version-by-version, don't waste effort porting these — they get reverted. Merge
  to the **A17 end-state** directly.

  </details>

Two more behavior changes come from the "config on HAL" and app-facing areas and are unverified from
our source, so they're tracked separately: a low-level routing-engine config change, background-audio
restrictions for apps, and a separate volume control for the voice assistant.

---

## Where Nissan stands, and why the merge is the real work

We confirmed by looking at the actual code and the commit history that our tree is **stock Android 14
plus Nissan patches** — none of Google's A15/A16/A17 car-audio changes are in yet.

The catch is *which* files we've patched. Our audio fixes over the past couple of years landed in the
busiest files — the main service, the volume-group classes, the focus logic. And those are exactly the
files Google rewrote the most. So the same handful of files are "hot" on both sides.

**The five biggest collision points** (Google rewrote them heavily *and* we've customized them heavily):

- **The main audio service** (`CarAudioService`) — Google restructured how it manages audio internally;
  we've added Audio Off, MaxVolumeStartup, the OEM gain callback, master-mute handling, and more.
- **The volume group base class** (`CarVolumeGroup`) — Google changed how it maps sounds to speakers and
  how muting works; we've patched startup volume and mute behavior in those very methods.
- **The focus logic** (`CarAudioFocus`) — Google added the fade-on-focus-loss behavior; we've added
  Audio Off mode there.
- **The focus interaction rules** (`FocusInteraction`) — Google changed the method signatures; we've
  added Audio Off behavior.
- **The core-audio volume group** (`CoreAudioVolumeGroup`) — Google rewrote its constructor and mute
  path; we've added the default-volume feature (ticket #222006) and Siri/E-call volume fixes there.

Two more files (`CarZonesAudioFocus`, `CarAudioZoneConfig`) are medium-risk for the same reason.

Everything else is easier: about ten files where we're still on stock Android 14, so Google's changes
can be taken almost as-is, plus roughly a dozen brand-new files from Google that we simply add.

---

## The volume-group classes, specifically

Because volume is where a lot of our customization lives, here's what happens to that family of classes.

Google changed the fundamental way a volume group tracks its speakers. Today our code maps each kind of
sound to a speaker **by address** (a text name). Google's new version maps **by device object** and adds
a companion "activation volume" config (the min/max-on-start feature mentioned above). It also splits the
mute operation into two clearer steps — one that mutes the hardware, one that saves the setting.

That mute split and the mapping change are precisely where our patches live — MaxVolumeStartup, the
mute/ducking tweaks, the #222006 default-volume logic, the Siri volume fix. So these classes are the
most delicate part of the merge. Our Nissan-only helper (`CoreAudioVolumeGroupHelper`, from #222006)
isn't something Google touches, but it plugs into the constructor Google rewrote, so it needs re-wiring
onto the new shape rather than dropping in unchanged.

The two decisions to make here:
- **MaxVolumeStartup vs. Google's new min/max-on-start** — same goal, two implementations. Pick one, or
  we risk clamping the volume twice.
- **The #222006 default-volume + persistence logic** has to be re-applied on top of Google's new
  constructor and mute steps, not pasted back verbatim.

---

## What we recommend doing, in order

1. **Bring in the easy Android 15 layer first** — add Google's new helper classes and apply the changes
  to the files we haven't customized. Large but low-risk, and it lays the foundation.
2. **Merge the five hot files one at a time**, each followed by a targeted test of the Nissan feature
  that lives in it (Audio Off, MaxVolumeStartup, Siri/E-call volume, master mute, ducking).
3. **Take the Android 16 structural change** that splits the config reader into an interface — but keep
  reading our XML config file, not the HAL.
4. **Apply the small Android 17 cleanups** (threading, and dropping the features Google added-then-removed).
5. **Get the audio team to decide the real overlap** — MaxVolumeStartup vs. Google's activation volume
  (same feature, two implementations) — before merging the volume files. Separately, decide whether to
  turn on Google's new focus-loss fade; that's an independent choice, not tied to Audio Off.
6. **Track the big separate pieces on their own** — the low-level audio HAL migration, the app-facing
  background-audio restrictions, and the assistant volume stream. These are outside the car-audio code
  and some are still unverified.

**How we'll know it worked:** the car boots on each variant, the audio layout loads from our XML file as
before, and — most importantly — our custom behaviors still work: Audio Off mode, MaxVolumeStartup,
mute, ducking, Siri/E-call volume, the guest-profile volume bar, and the OEM gain callback. Plus the
usual audio sanity checks (volume keys, focus, ADAS ducking, ANC).

---

## A note on trust

Everything about the car-audio code changes above is grounded in the **actual Google source diffs** and
**our actual code and commit history**, so it's reliable. A second set of topics — the low-level engine
config, the background-audio restrictions, the assistant stream — comes from **release-note summaries we
couldn't fully verify locally**, so those are flagged as needing confirmation against real source before
we plan work around them.

*For the file-by-file tables, the exact conflict matrix, and the volume-group class table, see
`final_migration_summary_a14_to_a17.md`.*
