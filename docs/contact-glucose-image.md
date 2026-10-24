# Contact glucose image

This opt-in display feature is packaged as `patches/contact-glucose-image.patch`.
Both existing browser-build workflows apply `patches/*` after checking out
submodules. No submodule URL, dosing code, signing configuration, or workflow
is changed. Base tested: Loop `ca3ff783433c1be39a1d28665736da41e4dff9da`.

For a local Xcode build, initialize the workspace's submodules, then run once
from the workspace root:

```sh
git apply --check patches/contact-glucose-image.patch
git apply patches/contact-glucose-image.patch
```

## Use

Open **Settings → Contact Image**, allow Contacts access, choose an editable
contact, then enable **Update Contact Photo**. Prefer a dedicated contact:
this replaces its existing photo and may sync the glucose image through the
contact account (for example, iCloud). Disabling or changing contacts leaves
the last photo on the previous contact; it does not restore the original.

The image has a glucose value, reported trend (omitted if unavailable), and a
large localized reading time. Units and the calendar date are not drawn. LOW/HIGH sensor conditions
are preserved. It uses the glucose measurement timestamp, not the write time.
The first update follows a glucose-store notification (or enabling after one
has already arrived during this app session). Unit changes appear with the
next glucose update. No synthetic glucose values or estimated trends are used.

Only the selected contact's photo is changed. Writes are serialized off the
main actor, identical successful writes are skipped, and failed writes can be
retried with the next store update. Changing contacts turns updates off until
explicitly re-enabled. Disabling stops subsequent queued writes once the
configuration change reaches the serial writer; a write already in progress
may finish. Contact errors appear in this settings screen, not as therapy alerts.

Background execution and contact-photo propagation are controlled by iOS.
The image can remain stale indefinitely if readings stop or writes fail.
The time alone cannot identify a reading from a previous day; check freshness
in Loop. It is not a live glucose widget or a dosing input.

## xDrip layout parity

Matches the timestamp-enabled renderer in ATC-T1D/xdripswift at `5a5fa47`
(also present on master when checked). Uses the same SwiftUI Canvas and
ImageRenderer at scale 1: 256×256, centered glucose at y=80 with an arrow or
110 without, bold arrow 54pt at y=150, and monospaced-digit semibold time
34pt at y=204, uniformly scaled to a maximum 160pt width. Glucose uses xDrip's
unit/value-dependent bold font sizes capped at 86pt with an arrow or 110pt
without, including -0.025 tracking. Missing trends omit the arrow like xDrip.
Loop retains its white-on-black colors and LOW/HIGH text; this matches layout
and typography, not xDrip's range coloring or freshness policy. SwiftUI renders
on the main actor; contact fetch/save still runs on the serial utility queue.

## Verification

Completed on the Linux implementation host:

- Swift syntax parsed for both new files and both modified Swift files.
- Patch applied to a clean copy of the exact pinned Loop commit.
- Applied files compared byte-for-byte with the implementation.
- Info.plist XML and Contacts purpose string validated.
- Both new Swift files verified in the app's Xcode groups and Sources phase.
- `git diff --check` and reverse patch check passed.

Not yet performed: Xcode type checking/build, device rendering, Contacts
permission/write tests, and background/iCloud propagation. No signed build
or TestFlight release was triggered.

### Device acceptance checks

1. Fresh install: feature is off, no Contacts prompt before choosing a contact.
2. Denied/restricted access: clear settings message; no automatic prompt loop.
3. Full and iOS 18+ limited access: selected accessible editable contact updates;
   inaccessible/deleted/read-only contacts show an error without affecting Loop.
4. Verify mg/dL and mmol/L, all reported arrows, unknown trend, LOW/HIGH, 12/24-hour
   time, and circular cropping (including long localized time strings). Confirm
   that the units label and calendar date are absent.
5. Deliver older backfill and repeated samples: photo must not regress to an
   older reading; repeated identical content should not cause another save.
6. Disable, change contact, cancel selection, relaunch, revoke permission:
   verify opt-in persistence and that unrelated contacts are untouched.
7. Stop readings: photo must retain the original measurement time.
8. Receive readings in the background and check propagation to the intended
   device/contact-photo surface; do not infer this from a successful foreground save.

The build workflows attempt upstream synchronization before building when
`SCHEDULED_SYNC` is not `false`. A feature branch has no matching upstream
branch. Do not dispatch that workflow on the feature branch without accounting
for this existing sync behavior. This change does not modify that setting.
