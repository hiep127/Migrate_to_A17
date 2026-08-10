# Nissan `nissan_u_ccs2_release` vs. Google AOSP `android14-release`

Diffs generated from the **`a14full`** checkout, comparing each repo's local
`nissan_u_ccs2_release` branch (Nissan/Alliance customizations on Android 14)
against the upstream Google AOSP `android14-release` branch (fetched fresh
from `android.googlesource.com`, since this checkout has no local ref for it).

Scope is limited to the paths tracked in
`/home/worker/doc/audio-media-workspace-a17.code-workspace` (the audio/media
subsystem: `server/audio`, `media`, `core/jni`, av `services`/`media`, Car
`service/audio` + `car-lib/car`).

## Repo list (from the A17 workspace file)

| Workspace folder | Backing git repo | Google AOSP branch | Diff file |
|---|---|---|---|
| base/services/audio, base/media, base/media/jni | `frameworks/base` | `android14-release` (`6e47c7075b9`) | `frameworks_base.diff` |
| av/services, av/media | `frameworks/av` | `android14-release` (`2c377d34a8a`) | `frameworks_av.diff` |
| car/audio, car/car-lib/car | `packages/services/Car` | `android14-release` (`f9a252f4a3b`) | `packages_services_Car.diff` |
| device/nissan | `device/nissan/aivi2_n_full`, `aasp_n`, `aasp/confighub` | **none** | not generated |
| Alliance audio config (OEM XML) | `vendor/alliance/services/car/audiocontrol` | **none** | not generated |
| Alliance Car Audio Service Plugin | `vendor/alliance/services/car/plugins/audio` | **none** | not generated |

`device/nissan/*` and `vendor/alliance/*` are Nissan/Alliance-proprietary repos
that were never part of AOSP — confirmed via `git ls-remote` against
`android.googlesource.com` (repository not found for all three). There is no
"google branch" to diff them against.

## Command used per repo

```bash
git fetch --depth=1 https://android.googlesource.com/<project> refs/heads/android14-release
git update-ref refs/google/android14-release FETCH_HEAD
git diff nissan_u_ccs2_release..refs/google/android14-release -- <scoped paths>
```

## Diff stats

| File | Files changed | Size |
|---|---|---|
| `frameworks_base.diff` | 47 | 348 KB |
| `frameworks_av.diff` | 331 | 1.14 MB |
| `packages_services_Car.diff` | 60 | 259 KB |

Diff direction: `nissan_u_ccs2_release..android14-release`, i.e. lines shown as
added (`+`) exist in Google AOSP but not in Nissan's branch, and lines removed
(`-`) are Nissan/Alliance-only customizations not in AOSP.
