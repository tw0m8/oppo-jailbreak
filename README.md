# oppo-timer-root

Field notes and a skill for rooting a locked-bootloader OPPO/ColorOS phone with the kernel
timer-page exploit, plus the operational discipline that makes a ~60% exploit behave like a
reliable tool.

Tested on: **OPPO Find X8s (PKT110, MT6991)**, ColorOS 17 CN, Android 17, bootloader locked,
kernel `6.6.118-android15-8-g9bc34d5b0c79-abogki537459655-4k`.

## What is here

- `SKILL.md` — the Hermes skill: triggers, procedure, failure signatures, pitfalls
- `docs/RUNBOOK.md` — the per-attempt checklist, including what to do when a run is lost
- `docs/BUILD.md` — how the payload is rebuilt and verified against the shipped binary
- `docs/APP-NOTES.md` — the app-side engineering that turns a 60% exploit into a usable tool

Not here: the payload source (`timer.c`), the KernelSU daemon source, and the Android app source.
The APK in `dist/` does bundle the compiled payload and `ksud` as assets, since the app cannot
run without them.

## Install

`dist/RootNow-v4.8.apk` (versionCode 38, signed with the dev keystore) is the app that drove all
the runs in this repo's notes. Verify before installing:

```bash
sha256sum -c dist/SHA256SUMS
```

Two caveats: the app contains the pinned payload digest, so a different `timer` binary requires
a rebuild, and it is debug-signed — install it as a sideload, not a store app.

## The short version

1. Root is **temporary**. It lives until reboot, and each run deliberately parks a kernel page.
2. **One attempt per boot.** After a parked attempt a retry cannot succeed; a boot marker in the
   app refuses it rather than wasting the boot.
3. The payload's own verdict is **not** the verdict. It checks `/proc/modules` and `su` a moment
   before ksud finishes loading and reports failure on runs that worked. Confirm with `su -c id`.
4. Two distinct failure modes exist and need different fixes:
   - `no zombie after Nms` — the exec thread never completed inside the wait window
   - `burst hit=0` — the page was freed but the spray lost it to a system allocation

## Measured behaviour

| Metric | Value |
|---|---|
| Success on first plant | 6 of 7 runs |
| Time to root | 3–5 s (`kernelsu loaded after 3-5s`) |
| Failure modes seen | `no zombie` ×2, `burst hit=0` ×2 across the session |
| Cost of a lost run | one reboot |

## Safety and limits

Root dies at reboot. Play Integrity trips — banking and some payment apps may refuse to run.
The exploit is pinned to one kernel build: a system update breaks it until the constants are
re-derived, and one of them (`STATIC_TIMER_SITE`) can only come from a live trace on the
matching image.

Only run this on hardware you own. A lost race can panic the device.
