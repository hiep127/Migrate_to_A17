# Audio Config Moving to the HAL — Explained in Plain Terms

*A companion to `config_on_hal_deep_dive.md`, written to be read start-to-finish without needing the
AIDL details. It answers three questions: what is actually changing, where does Nissan stand today,
and what should we do for A17.*

---

## The short version

When people say "Android is moving audio configuration into the HAL," they are really talking about
**two different changes** that happen to sound the same. They live in different places, replace
different files, and have nothing to do with each other. It is easy to mix them up — the migration
notes we were given do exactly that.

1. **The car's zone layout** (which speakers exist, how they group into volume controls, which audio
   goes where) can now be handed to Android **by the AudioControl HAL at runtime**, instead of being
   read from the `car_audio_configuration.xml` file baked into the system image.

2. **The low-level audio routing engine** (the "configurable audio policy," or CAP — the rules that
   decide which physical output a given sound uses) can now get its rules **from the core Audio HAL**,
   instead of from the static XML + parameter-framework files on the vendor partition.

Both changes arrived in Android 16. Both are **optional** — Google deliberately kept the old file-based
way working. Neither is something we are forced to adopt just to get to A17.

My recommendation, explained below, is simple: **for A17, keep our configuration in the XML files where
it is today. Don't move it into either HAL.** We can always revisit later.

---

## Change #1 — The car zone layout can come from the AudioControl HAL

### What this is

Today, when the car boots, `CarAudioService` opens a file — `car_audio_configuration.xml` — and reads
the whole audio layout from it: the zones, the volume groups, which bus each sound routes to. That file
ships inside the system image.

Starting in Android 16, Google added a second option. The AudioControl HAL (the vendor service that
already handles things like fade, balance, and ducking) can now be asked a question at boot:
"*do you want to describe the audio zones yourself?*" If the HAL says yes, it returns the entire zone
layout as live data, and Android uses that instead of the XML file. If the HAL says no, Android falls
back to reading the XML file exactly like before.

The point of this feature is flexibility: a manufacturer could ship **one** software build and let the
HAL describe a different speaker layout per vehicle trim, or push a layout update without shipping a new
system image.

### Where Nissan is today

We are firmly on the old file-based path, and — importantly — **our hardware couldn't do the new way
even if we wanted it to right now.** The reason is that the "describe your own zones" capability is part
of a **newer version of the AudioControl HAL interface** than the one in our tree. Our tree only has up
to **version 3** of that interface, and the zone-description commands (`getAudioZones` and friends)
simply **do not exist in version 3** — they were added later, in the version that ships with Android 16.
Our Alliance vendor HAL is even built against version 2 and implements none of it.

So adopting this feature isn't just a Java change in `CarAudioService`. It would mean **upgrading the
whole AudioControl HAL interface to the newer version, and then writing new vendor HAL code** to
actually produce the zone description. That is a real chunk of work in the vendor HAL, not a small
tweak.

### Is it worth it for us?

For a Nissan head unit with a **single, fixed audio zone**, the benefit is small — the layout never
changes at runtime, so describing it live buys us little. Meanwhile one of our targets (da2) already
routes through the Alliance audio-control plugin plus the XML file, which works fine. So the cost is
high and the payoff is low.

**Recommendation: keep the XML file. Skip this feature for A17.**

There is one piece of the A16 refactor we *do* have to take, but it costs nothing here: Google turned
`CarAudioZonesHelper` (the class that reads the config) into a small interface with two
implementations — one that reads XML, one that reads from the HAL. We take that refactor because it
comes bundled with the A16 code, but we simply keep the "does the HAL describe zones?" answer set to
**no**, so the XML reader stays in charge. Nothing about our configuration changes.

---

## Change #2 — The routing engine's rules can come from the core Audio HAL

### What this is

Underneath `CarAudioService` there is a lower-level piece of Android called the **audio policy engine**.
It decides, for every kind of sound (media, navigation, phone, alarm, and so on), which physical output
path it should take and how volume is applied. We use the "**configurable**" version of this engine,
which means those decisions are driven by **configuration files** rather than hard-coded logic — a set
of XML files plus some "parameter-framework" rule files on the vendor partition.

In Android 16, Google added the ability for the **core Audio HAL** to hand these engine rules to Android
directly, instead of Android reading them from those vendor files. And to keep older devices working,
they also shipped a **converter** that automatically turns the existing XML rules into the new format —
so even on the new path, your old files can keep driving the engine.

### The catch, and where Nissan stands

This new "engine rules from the HAL" path only works if the device is using the **new-style (AIDL) core
Audio HAL**. Our device is still on the **old-style (HIDL) core Audio HAL**. So we literally cannot use
this new config path until the core Audio HAL itself is migrated from the old style to the new one — and
**that migration is one of the biggest, most involved pieces of the entire A17 effort**, living well
outside the car-audio code we've been analyzing.

There's also a smaller, more concrete wrinkle. In Android 16 Google **renamed all the routing
strategies** to require a `STRATEGY_` prefix — for example `music` became `STRATEGY_MEDIA`,
`voice_call` became `STRATEGY_PHONE`. Our configuration has **21 strategies, all still using the old
lowercase names** (including OEM-specific ones like `oem_ipa`). If we ever move to the new AIDL engine
path, every one of those names has to be renamed, in the strategy file and everywhere else that refers
to them.

One caution worth repeating: the details of this second change come from **release-note summaries, not
from the actual source diffs** we were able to verify. Before anyone spends effort on it, we should
confirm it against the real Android source.

### Recommendation

- **Keep our existing engine configuration files.** If A17 eventually forces us onto the new core Audio
  HAL, use Google's **converter** so those files keep working — don't rewrite the rules by hand.
- **Treat the `STRATEGY_` rename as the one edit we probably can't avoid** if/when the engine goes to
  the new path. It's mechanical but touches many files, so plan it as its own careful, well-tested step.
- **Verify the whole thing against real source first**, since our information here is second-hand.

---

## The one thing to fix now, regardless of everything above

Separate from both big changes, there's a small housekeeping problem worth fixing on its own: our
AudioControl HAL **declares three different version numbers in three different places** — the device
manifest says version 1, another config fragment says version 3, and the build file compiles against
version 2. These should all agree. Settling them on **version 3** is cheap, removes confusion, and is
needed for a callback feature we already rely on. This is low-risk and can be done independently.

---

## Putting it together — what to actually do for A17

| The change | What we should do | Why |
|---|---|---|
| Car zone layout from the HAL | **Skip it.** Keep the XML file; take the code refactor but leave the HAL path off. | Needs a HAL interface upgrade + new vendor code; little benefit for a single-zone unit. |
| Engine rules from the HAL | **Keep our config files.** If forced onto the new core HAL, use the converter. | The new path depends on a huge separate HAL migration; our files can keep working. |
| Renaming strategies to `STRATEGY_` | **Only if/when the engine goes to the new path.** Plan it as an isolated step. | Breaking but mechanical; touches many files. |
| Core Audio HAL old→new style | **Its own large project**, tracked separately. | Prerequisite for the engine-config change; biggest single effort. |
| AudioControl HAL version mismatch | **Fix now.** Settle on version 3. | Cheap, removes a real inconsistency. |

**Bottom line:** the "config moves to the HAL" story sounds like a mandatory rewrite, but in practice
it's opt-in and backward-compatible. For A17, Nissan should keep its audio configuration in the XML
files it has today, adopt only the code refactors that come along for free, and treat the actual HAL
config migrations as optional future work — after verifying the details against real Android source.

---

*For the exact interface names, file paths, and the evidence behind each statement here, see the
technical companion: `config_on_hal_deep_dive.md`.*
