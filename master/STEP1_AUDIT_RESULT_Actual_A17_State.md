# Step 1 Audit Result — Actual State of `/a17full` vs. This Doc's Model

> **Context.** `FINAL_Conclusion_What_To_Change_A14_to_A17.md` was written by diffing a clean A14
> baseline (`/a14full`, `nissan_u_ccs2_release`) against fixed AOSP A15/A16/A17 release tags. Its §8
> Step 1 said: *"Add the A15 foundation... nothing else compiles without it."*
>
> The tree actually used for this audit — `/a17full/android/packages/services/Car`, branch
> `nissan_c_join_aivi2_fc` — turned out to already have every file that instruction wanted added. This
> note records what was checked, what was confirmed, and one open item that needs a decision from
> whoever owns the Qualcomm branch sync and the Nissan audio feature roadmap.
>
> Every claim below was checked two ways: against this doc's stated requirement, and against the actual
> live content of `https://android.googlesource.com/platform/packages/services/Car` (`main` branch, as
> of 2026-08-05). Both checks are called out explicitly — do not extend either conclusion past what was
> actually tested.

---

## Part 1 — Confirmed complete, matches live AOSP `main` too

No action needed. These were re-verified against the doc's requirement **and** against the actual
current content of AOSP `main`, not just assumed from local file presence.

| Item | Status |
|---|---|
| 8 new A15 classes (`AudioManagerWrapper`, `CarActivationVolumeConfig`, `CarAudioDeviceCallback`, `CarAudioFadeConfigurationHelper`, `CarAudioParserUtils`, `CarAudioPlaybackMonitor`, `CarAudioProtoUtils`, `CarAudioServerStateCallback`) | Present, wired into real callers (not dead code), and match live AOSP `main` |
| `CarAudioZonesHelper.java` converted from `final class` → `interface` | Confirmed on live AOSP `main` too (`public interface CarAudioZonesHelper`) |
| `CarAudioDeviceInfo`'s constructor takes `(AudioManagerWrapper, AudioDeviceAttributes)` | Confirmed on live AOSP `main` too |
| `CarAudioService.java`'s single `AudioPolicy` → 4-way split (`mVolumeControlAudioPolicy`, `mFocusControlAudioPolicy`, `mRoutingAudioPolicy`, `mFadeManagerConfigAudioPolicy`) | Confirmed on live AOSP `main` too |
| `CarAudioContext.java`'s internal duck-context table (`sContextsToDuck` / `evaluateAudioAttributesToDuck`) | Present on live AOSP `main` too — this doc's §3a instruction to remove the public `getContextsToDuck()` accessor is correct; the *internal* table is a separate, legitimate, still-present mechanism, not local debt |
| `CarAudioDynamicRouting`, `CarAudioVolumeGroup`, `CarVolumeGroupFactory`, `CarAudioUtils`, `CarAudioZone` (§3a files) | Match this doc's §3a requirements |

---

## Part 2 — Present locally, absent from public AOSP `main` (needs a decision, not a doc fix)

This is the part that changed after checking against the real upstream repo instead of stopping at
local git history. All items below trace to a cluster of commits dated Feb–May 2025, with convincing
AOSP-style authorship (`oscarazu@google.com`, `xuweilin@google.com`, `rajgoparaju@google.com`, proper
`Bug:`/`Test:`/`Change-Id:` trailers). **When checked against the actual `main` branch on
`android.googlesource.com` today, none of them are reachable from it:**

| This tree has... | Public AOSP `main` actually has (verified 2026-08-05) | How it was checked |
|---|---|---|
| `CarAudioService`'s 4 policy builders run on `mHandlerThread.getLooper()` (a background thread) | `Looper.getMainLooper()` for the policy builders, `mContext.getMainExecutor()` for callbacks | Fetched `CarAudioService.java` from `main`, checked line-by-line |
| `mPersistFadeBalanceLevels` field, actively used (constructor init, dump, API getter) | Does not exist at all | Grepped the real file fetched from `main` — zero matches |
| `ContentObserverFactory.createObserver()` takes an explicit `Handler` parameter | No `Handler` parameter — binds the main looper internally | Fetched both the local and `main` versions and diffed them |
| `CoreAudioVolumeGroupCallback.init()` is no-arg; executor supplied via constructor | `init(Executor executor)` — takes the executor as a parameter | Fetched the real file from `main` |
| `car/audio/hal/` folder is gone; single concrete `AudioControlWrapper` class; no `V1`/`V2`/`Aidl` variants | `hal/` folder **still exists**; `AudioControlWrapper` is still an **interface**; `AudioControlWrapperV1.java`, `V2.java`, and `Aidl.java` **all still exist as separate files** | Listed the real `hal/` directory tree on `main` |
| `CarAudioFocusEnforcement.java` / `CarAudioParkedStateMonitor.java` / `CarAudioEffects.java` ("Relaxed Park Mode") — fully wired into `CarAudioService.java` | None of these three files exist | Listed the real `car/audio/` directory tree on `main` |

**How this was chased down:** the first pass classified all of these as "doc is stale, local code is a
legitimate newer AOSP state," based on reading local `git log`/`git show` output — the commits had real
Google authorship and proper trailers, so they looked authentic. That conclusion was not checked against
the actual public ref before being written down. One commit in the cluster (`57362aba25`, "Move car
audio service handlers to car audio service thread") was fetchable directly from
`android.googlesource.com` by its hash — git object storage keeps commits reachable even when they're
not on any branch tip — but a page-by-page scan of `main`'s real commit log around its date (Feb 2025)
shows a completely different set of commits landing in that window. It was never part of `main`'s
ancestry.

**Most likely explanation:** the branch this tree syncs from (`automotive-aosp-va.lnx.17.0` via a
Qualcomm sync, ultimately Google-authored) is pulling in genuine automotive code that hasn't been
published to the public `android.googlesource.com` mirror — plausibly an internal or partner-preview
branch used for early automotive-OEM access, which is common for AAOS features ahead of a public
release. That's a reasonable, benign explanation, but it is a guess, not something confirmed here.

**This is different from "the doc is outdated."** This doc's specific original claims for these items —
main looper, no `mPersistFadeBalanceLevels`, no-`Handler` `ContentObserverFactory`, `Executor`-param
`init()`, `hal/` folder with `V1`/`V2`/`Aidl` — turn out to still be **correct as of today's public AOSP
main**. What's actually true: this tree has moved past both this doc's A14 baseline and public AOSP
`main`, onto code that (as far as this audit could determine) only exists in this downstream branch.

### Decision needed

Not resolvable by reading source alone — needs input from whoever owns the Qualcomm branch sync and the
Nissan audio feature roadmap:

1. Is moving `CarAudioService`'s policy builders off the main car-service thread onto
   `mHandlerThread` intentional for this platform (e.g. a deliberate SoC-specific stability/perf choice)?
2. Is "Relaxed Park Mode" (`CarAudioFocusEnforcement` and friends) something Nissan is meant to be
   shipping, or did it arrive unintentionally via the branch sync?
3. Is the single-class `AudioControlWrapper` (vs. the public interface + `V1`/`V2`/`Aidl` split) required
   by the Nissan HAL integration, or could it be safely re-aligned with the public AOSP structure later?

**Note (resolved, no action needed):** the doc's own §11 worried this "Relaxed Park Mode" subsystem
might collide with Nissan's "Audio Off" feature. It doesn't — Audio Off lives entirely in the OEM plugin
(`vendor/alliance/.../CarVolumeGroups.java`, gated by a `Settings.Global` key, mutes at the volume-group
layer) while Relaxed Park Mode lives in AOSP's own code (gated by park-state, adjusts individual
`PlayerProxy` gain). Disjoint files, disjoint mechanisms, disjoint trigger conditions — confirmed
independently of the Part 2 question above.

---

## Lesson for future audits of this tree

Local git history in a downstream/synced branch is not sufficient evidence that something reflects
"real AOSP," even when commit messages look completely authentic. Anything claimed as a legitimate newer
upstream state should be spot-checked against the live public ref
(`https://android.googlesource.com/platform/packages/services/Car/+/refs/heads/main/...`) before being
treated as settled.
