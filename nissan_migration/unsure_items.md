# Unsure items — not included in the final-state plan

These were investigated during the Nissan-to-A17 migration research but **did not meet the bar** for
inclusion in the main plan (`nissan_migration_plan.html` / `step*.md`) as confirmed work items. Each entry
states exactly what was checked, what was found, and what would resolve the uncertainty. Nothing here
should be assumed true or acted on without further confirmation.

---

## 1. "Siri / E-call volume" fix — existence unconfirmed

**Status:** could not find any evidence this feature exists, anywhere.

**What was checked:**
- Exhaustive text search (`grep -rniE "siri|ecall|e-call|emergency"`) across the entire
  `packages/services/Car` project (not just `car/audio/`) in the `a14full` checkout — zero hits for "Siri"
  anywhere in the Car project.
- The authoritative `nissan_u_ccs2_release` vs `android14-release` diff
  (`diffs_nissan_vs_google/packages_services_Car.diff`) — no matching feature in the
  `CarAudioFocus.java`/`FocusInteraction.java`/`CoreAudioVolumeGroup.java` sections, the files most likely
  to hold this kind of logic.
- The only related code found anywhere is a stock AOSP javadoc constant,
  `EXTRA_INFO_MUTE_TOGGLED_BY_EMERGENCY`, in `car-lib/src/android/car/media/CarVolumeGroupEvent.java` —
  documented as covering "(example: European eCall) in progress." This is generic AOSP plumbing already
  present in `a17full`, not a Nissan feature.

**What would resolve this:** the actual ticket/CR number for "Siri/E-call volume." Without it, this cannot
be scoped — it may not exist in `car/audio` at all, or may have landed somewhere outside this component
(e.g. `frameworks/base` `AudioService`, or the Alliance OEM plugin rather than `car/audio` itself).

**Do not** implement a speculative fix for this. If the ticket surfaces a real change, re-run the same
verification method used for the rest of this plan (on-disk read + diff) before writing any code.

---

## 2. Occupant-zone *filter* — likely conflated with a different, already-present feature

**Status:** no evidence of a filter distinct from the occupant-zone/mirror-cleanup logic that's already
confirmed present in `a17full`.

**What was checked:**
- The original pre-migration plan listed "occupant-zone filter" as a suspected Nissan customization.
- Deep-dive research found a real Nissan A14 feature in this area —
  `CarAudioService.removePrimaryZoneRequestForOccupantLocked()` / `removeAudioMirrorForZoneId()`, called
  from `updateUserForOccupantZoneLocked()`, plus `MediaRequestHandler.getAssignedRequestIdForOccupantZoneId()`
  — confirmed via the authoritative diff, and confirmed **already present** in `a17full` today
  (`CarAudioService.java:4024-4026`, `MediaRequestHandler.java:244`).
- No second, separate "filter" mechanism (e.g. something that restricts which occupant zones can be
  assigned, or filters audio routing by occupant) was found anywhere — every other occupant-zone method in
  `CarAudioService.java` is line-for-line identical to stock AOSP.

**Most likely explanation:** "occupant-zone filter" in the original plan was probably just an inexact
description of the cleanup logic above, not a separate feature. Since that logic is already in `a17full`,
this is likely fully resolved with no further action — but is listed here rather than in "already done"
because the possibility of a genuinely separate filter feature elsewhere (outside `car/audio`, e.g. in
`CarOccupantZoneService.java`) was not exhaustively ruled out.

**What would resolve this:** confirm with whoever originally used the term "occupant-zone filter" what
specific behavior they meant, or search `CarOccupantZoneService.java` and related non-audio occupant-zone
code if the term resurfaces in testing.

---

## 3. `canSwapCallOrRingerClientRequest()` — confirmed real gap, but priority/impact unconfirmed

**Status:** the gap is real and directly confirmed by code inspection (not speculative), but whether it
causes any actual user-visible bug is unknown.

**What was checked:**
- The authoritative diff shows `CarAudioFocus.java`'s **duplicate-request matching for already-blocked
  requests** (the code path handling "this is a repeat of a request that is currently blocked") calls
  `canSwapCallOrRingerClientRequest(...)` in Nissan's version, but only checks
  `entry.getAudioContext() == requestedContext` (no swap tolerance) in Google's `android14-release`.
- Directly confirmed in `a17full` (not inferred): `CarAudioFocus.java` has exactly one call site for
  `canSwapCallOrRingerClientRequest()`, at line 307, inside the **focus-holders** matching block (the
  "Replacing accepted request from same client" path). The **blocked/pending-request** matching block
  (line 343, "Replacing pending request from same client id") does **not** call it — confirmed by reading
  that exact code region — matching Google's stock form, not Nissan's.
- The function itself (`canSwapCallOrRingerClientRequest`, defined at `CarAudioFocus.java:742` in
  `a17full`) already exists and is already used in the other path — so this is a one-call-site gap, not a
  missing function.

**What this means in practice, unconfirmed:** without this second call site, a call/ringer client request
that is currently *blocked* (not yet holding focus) may be evaluated more strictly than one that's already
a focus holder — e.g. a legitimate call-to-ringer (or ringer-to-call) swap might be rejected instead of
allowed, in the specific case where the prior request from that client is still pending/blocked rather than
already granted. Whether this is user-visible, and how often that exact sequence occurs, was not
determined.

**What would resolve this:** trace an actual call/ringer swap scenario through both code paths (or find a
bug report describing the symptom) to determine if the missing call site is load-bearing. If it is, it's a
one-line addition mirroring the existing call site at line 307.

---

## 4. `CarAudioContextInfo.equals()` / `hashCode()` — real addition, purpose unconfirmed

**Status:** confirmed real (present in the authoritative diff as a genuine Nissan-only addition), but no
consumer was identified, so its necessity for A17 is unknown.

**What was checked:**
- The diff shows Nissan's `CarAudioContextInfo.java` has a full `equals()`/`hashCode()` implementation
  (value-equality based on ID, name, and audio-attribute-set comparison via `ArraySet`) that Google's
  `android14-release` does not have.
- No caller of `CarAudioContextInfo.equals()`/`hashCode()` requiring this specific implementation (e.g. a
  `Set<CarAudioContextInfo>` or `Map` keyed by it, or a test asserting equality) was identified during this
  research pass — the search did not extend to the full test suite or to callers outside `car/audio/`.

**What would resolve this:** search the A14 fork's test suite and any `Set`/`Map` usage of
`CarAudioContextInfo` for a real dependency on value-equality. If none exists, this can likely be skipped
entirely rather than ported.
