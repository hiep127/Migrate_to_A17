# Step 5 — BT voice-call double-focus-request block

[← back to index](00_README.md) · Depends on: [Step 0](step0_resync_source.md)

## What it is

`CarAudioFocus`'s focus-request evaluation path rejects `AUDIOFOCUS_REQUEST` for
`USAGE_VOICE_COMMUNICATION` originating from the Bluetooth package (`BT_PACKAGE`), to avoid double focus
requests for voice calls. This was cherry-picked in the A14 fork from a Nissan A12 change — comment in the
A14 source cites "Cherry-pick from Nissan A12 commit
http://gerritaivi.lge.com/.../39383" and ticket `C2BST-26777`. Found incidentally during the
[Step 0](step0_resync_source.md) research pass while investigating Audio Off, not one of the originally
requested features, but audio-focus-adjacent and worth carrying forward since it fixes a real double-request
bug.

## Exact source, read directly from `a14full` `CarAudioFocus.java`

```java
private static final String BT_PACKAGE = "com.android.bluetooth";
...
/**
 * Fixed (b/C2BST-26777): Prevent request audio focus for voice call from BT package
 * Added: Reject request audio focus for USAGE_VOICE_COMMUNICATION from com.android.bluetooth
 * Reason: Cherry-pick from Nissan A12 commit
 * http://gerritaivi.lge.com/c/platform/vendor/alliance/services/car/audiocontrol/+/39383
 */
if (afi.getPackageName().equals(BT_PACKAGE)
        && afi.getAttributes().getUsage() == AudioAttributes.USAGE_VOICE_COMMUNICATION) {
    // https://android-review.googlesource.com/c/platform/packages/apps/Bluetooth/+/676254
    // introduced a double request of focus for voice call in automotive products.
    // focus request is already handled by com.android.server.telecom.
    // This WA is intended to be removed until @todo (b/XXX) is solved
    return AudioManager.AUDIOFOCUS_REQUEST_FAILED;
}
```

This sits right at the top of `evaluateFocusRequestLocked(AudioFocusInfo afi)`, before any of the normal
focus-arbitration logic runs — a hard early return, not a modification to the arbitration matrix. The A14
comment itself documents the reason precisely: a known AOSP Bluetooth-stack change
(`platform/packages/apps/Bluetooth/+/676254`) introduced a double focus request for BT voice calls in
automotive builds, and this is a workaround pending a real upstream fix (`@todo (b/XXX)`, never filled in).
Worth checking whether that AOSP Bluetooth-side bug is still present in the A17-era Bluetooth stack before
assuming the workaround is still needed — but porting it as-is is the safe default.

## Confirmed integration point in `a17full`

`CarAudioFocus.java:243` — `private int evaluateFocusRequestLocked(AudioFocusInfo afi)` still exists under
this name in current A17 stock, confirmed by direct read. No BT-package guard currently present anywhere in
the file (confirmed — no `BT_PACKAGE`/`bluetooth` references exist in `a17full`'s `CarAudioFocus.java`
today).

Note the signature drift already covered by the earlier verification pass: A17's broader
`evaluateRequest()` method (the one `FocusInteraction` calls into) now takes `int requestedUsage` instead of
`@AudioContext int` — confirm which of the several `evaluateFocusRequestLocked`/`evaluateRequest` overloads
in the current file is the right insertion point before porting (there are multiple overloads at lines 243,
508, and call sites at 366, 759, 855, 1004 — read the whole request-evaluation chain first, don't guess from
a single grep hit).

## Files to change in `a17full`

`CarAudioFocus.java` — add the same BT-package + `USAGE_VOICE_COMMUNICATION` rejection guard at the
equivalent point in the current `evaluateFocusRequestLocked`/request-evaluation chain. Preserve the original
ticket reference (`C2BST-26777`) in a short comment, since this is exactly the kind of non-obvious
"why" that's worth keeping (a plain read of the code wouldn't explain why BT voice-call requests are
special-cased).

## Risk

Low — small, isolated, no dependency on the other steps, no known interaction with Google's A15+ fade-manager
dispatch (this guard fires during request evaluation, before any focus-loss dispatch logic runs).

## Verify

- Place a BT-initiated voice call; confirm only one focus request is granted/dispatched for the call (no
  duplicate/competing focus entries attributable to the BT package).
- Place a non-BT `USAGE_VOICE_COMMUNICATION` request (e.g. a native telephony call); confirm it is
  **not** affected by this guard — the block must be scoped to the BT package specifically, not to the
  usage type in general.
