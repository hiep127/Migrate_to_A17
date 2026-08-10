# Step 0 — Re-sync source of truth

[← back to index](00_README.md)

## Goal

Confirm the `a14full` checkout is a valid porting source, and re-sync only the specific piece that turned
out to be unreliable, before writing any A17 code.

## Correction after re-verification

An earlier research pass characterized the whole `a14full` checkout (`nissan_u_ccs2_release`, HEAD
`dbe12bde8d8`) as stale, based on grepping the working tree for "guest"/"CP"/"audio_off" and finding
nothing, then reconstructing the real code via `git show <hash> -- <path>` against commits reachable from
`remotes/m/nissan_u_ccs2_release` (tip `df434821b9b`) and later unmerged refs (up to `429afe45ba0`).

A follow-up direct verification pass — running real `diff -u` between `a14full` and `a17full`, and reading
the actual on-disk files in full (not `git show` on scattered hashes) — found that **most of what matters is
actually present, committed, and clean** in the checked-out tree:

- `CoreAudioVolumeGroupHelper.java` ([Step 1](step1_max_volume_startup.md)) — present, full content
  confirmed on disk, matches exactly.
- `CarAudioSettings.java`'s `GROUP_ID_MEDIA`/`RING`/`CALL` → `Settings.Global` logic and the
  `ro.config.isPersistVolumeGroupMute` sysprop override ([Step 2](step2_settings_global_persistence.md)) —
  present, confirmed via direct `diff -u` against `a17full`, `git status` clean (no local uncommitted
  edits).
- `CarAudioGainCallbackHandler.java` ([Step 3](step3_oem_gain_callback_wiring.md)) — present, full content
  confirmed on disk.
- Audio Off's core mechanism — `CarAudioService.handleMuteChanged()`'s media-group branch,
  `CarZonesAudioFocus.applyUserMuteAudioFocusPolicy()`, and `CarAudioFocus`'s `mUserMuted`/
  `isEntertainmentSource()`/restoration-guard logic ([Step 6](step6_audio_off_mode.md)) — all present,
  confirmed on disk, and **simpler** than what the reconstruction described.
- The BT voice-call focus block ([Step 5](step5_bt_voice_call_focus_block.md)) — present, confirmed on
  disk with full original comments intact.

**What the reconstruction got wrong**, specifically: it described a `SettingsObserver.java` trigger file and
per-`FocusEntry` mute tracking for Audio Off that **do not exist anywhere in the checked-out tree**. Those
were most likely details from a genuinely later, unmerged commit reachable only via `git show` — i.e. a
real future iteration of Audio Off that hasn't landed on `nissan_u_ccs2_release` yet, not a fabrication, but
also not what's authoritative today. [Step 6](step6_audio_off_mode.md) has been corrected to describe the
on-disk mechanism, which is directly verified and simpler.

## What's still genuinely open — now precisely identified, not vague

Ran the actual diff rather than speculating further:

- `git diff HEAD remotes/m/nissan_u_ccs2_release -- service/src/com/android/car/audio/` shows the local
  `HEAD` (`dbe12bde8d8`, 2026-08-03) is **already newer** than `remotes/m/nissan_u_ccs2_release`
  (`df434821b9b`, 2026-04-29) — the remote ref is *missing* content HEAD already has (69 deleted lines
  across 7 files when diffing HEAD→remote, i.e. remote lacks them), not the other way around. **The local
  checkout was never behind `remotes/m` — that part of the original "stale" claim is fully retracted.**
- The [Step 4](step4_zone_playback_callback_hardening.md) ANR fix commit `429afe45ba0` genuinely exists as
  a git object, but `git merge-base --is-ancestor 429afe45ba0 HEAD` returns **false** — it is **not** on
  `nissan_u_ccs2_release` at all. `git branch -a --contains 429afe45ba0` shows it only on a **sibling branch
  family**: `alliance_u_26w30_260720`/`260722`/`260724` and `alliance_u_release`. Confirmed directly:
  `a14full`'s checked-out `CarAudioService.java:1711` still has
  `registerAudioPlaybackCallback(mCarAudioPlaybackCallback, null)` (the pre-fix `null` Handler), and
  `ZoneAudioPlaybackCallback.java` still calls the lock-suffixed `startTimersForContextThatBecameInactiveLocked`
  under `mLock`. **This is a real, unmerged, cross-branch fix — not present on `nissan_u_ccs2_release`
  (A14) or on `a17full` (A17) today.** It needs to be pulled from `alliance_u_release` specifically, not
  reconstructed or assumed absent.
- Audio Off's `SettingsObserver.java`/per-`FocusEntry` mute-tracking claim from the original research pass
  could not be corroborated anywhere in `nissan_u_ccs2_release`'s reachable history checked so far. Treat it
  as **unconfirmed** rather than "a real future iteration" until someone with Gerrit access finds the actual
  commit — don't block on it; [Step 6](step6_audio_off_mode.md)'s on-disk mechanism is directly verified and
  is the right thing to port unless proven otherwise.

## Actions

1. **For Step 4 specifically**: pull the ANR fix from `alliance_u_release` (or whichever of the
   `alliance_u_26w30_*` snapshots is authoritative) — `git show 429afe45ba0 -- service/src/com/android/car/audio/ZoneAudioPlaybackCallback.java service/src/com/android/car/audio/CarAudioService.java`
   against that branch, then port forward to `a17full`'s current method names/structure.
2. Confirm with whoever owns Gerrit whether `nissan_u_ccs2_release` is meant to pick up
   `alliance_u_release` fixes automatically (branch merge policy) or whether this is a genuine gap Nissan
   needs to cherry-pick manually — this affects whether Step 4's finding is a one-off or a sign that other
   `alliance_u_*`-only fixes are also missing from the Nissan line and worth a broader audit later.
3. Chase down the real Audio Off "final iteration" only if/when Gerrit access surfaces it — not a blocker
   for starting Steps 1-5.

## Verify

- `git merge-base --is-ancestor 429afe45ba0 HEAD` on the authoritative `nissan_u_ccs2_release` ref returns
  true once the ANR fix is actually merged/cherry-picked in, or the fix has been manually ported to
  `a17full` per [Step 4](step4_zone_playback_callback_hardening.md).
- No other `alliance_u_*`-only commits for `car/audio/` remain unaccounted for (spot-check a few more paths
  from that branch family against `nissan_u_ccs2_release` if time allows).

## Addendum: a materially better source became available

After the above was written, `doc/Migrate_to_A17/diffs_nissan_vs_google/packages_services_Car.diff` was
generated — a real `git diff nissan_u_ccs2_release..android14-release` (both Android 14), which is a
strictly cleaner signal than diffing across Android versions (`a14full` vs `a17full`) the way the rest of
this plan was researched, since it can't confuse "AOSP changed this between versions" with "Nissan changed
this." Reviewing it surfaced [Step 8](step8_gain_context_enrichment.md),
[Step 9](step9_suspend_wake_volume_limit.md), and [Step 10](step10_forced_mute_volume_blocking.md) — three
real Nissan features the original research (Steps 1-7) missed entirely — and corrected the
occupant-zone/mirror-handling "out of scope" claim in the [index](00_README.md) (real Nissan feature at
A14, but independently already present in `a17full` by A17).

**If more time is available, re-run the Steps 1-7 verification using this diff as the primary source**
instead of (or alongside) the `a14full`-vs-`a17full` comparison — it's very likely a few more small gaps
exist in files not yet reviewed this way. `frameworks_base.diff` and `frameworks_av.diff` in the same
folder are also unreviewed as of this writing and may contain platform-level Nissan customizations
(outside `car/audio` but potentially relevant to APIs `car/audio` depends on) — check the `README.md` in
that folder for scope and diff-direction details before reading them.
