---
name: oppo-timer-root
description: Root OPPO ColorOS phones with the timer exploit.
version: 1.0.0
author: Tw0m8, Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [android, exploit, oppo, colorsu, kernel, root]
    related_skills: [android-phone-exploit-investigation]
---

# OPPO Timer-Exploit Root

Temporary root on OPPO/ColorOS phones through the kernel timer-page exploit, then KernelSU
late-load for a real `su`. Root lives until reboot; every run parks one kernel page, so the
cycle is reboot -> exploit -> use root -> reboot. The exploit is pinned to one kernel build
and silently fails on any other.

## When to Use

- A locked-bootloader OPPO/OnePlus/realme phone needs root without unlocking
- An existing RootNow-style app reports `race lost`, `no zombie`, or `burst hit=0`
- Re-deriving constants after a system update changed the kernel
- Don't use for: devices with an unlockable bootloader (use fastboot + KernelSU boot image),
or kernels other than the one listed in Prerequisites

## Prerequisites

- **Kernel must match**: `6.6.118-android15-8-g9bc34d5b0c79-abogki537459655-4k`
  (`uname -r` on device). A different build means every constant is wrong - re-derive first.
- Payload `timer` and daemon `ksud` pushed to `/data/local/tmp`, both executable.
- Exploit constants for this build: `TIMER_PAGE_IDS=277,275`, `TIMER_FREE_IDS=275,276,279`.
- Verified kernel addresses (from the matching `vmlinux`): `selinux_state 0xffffffc08239c620`,
  `panic_on_warn 0xffffffc08214b2cc`, `root_user 0xffffffc08216f6e0`,
  `init_user_ns 0xffffffc08216f450`, `init_ucounts 0xffffffc0821711b0`,
  `init_groups 0xffffffc082170c40`, `STATIC_TIMER_SITE 0xffffffc0801a4dfc`.

## Procedure

1. **Reboot before the first run.** A parked page from an earlier attempt holds a kernel
   object for the life of the boot, so a retry cannot succeed. Done when the app reports
   "Reboot detected since the last parked attempt".
2. **Check conditions**: charge >= 30%, battery saver off. The spray is CPU-bound; a throttled
   phone loses the page race. Done when preflight reports all checks passed.
3. **Run once.** The script plants the timer page, fakes the credential structs, then loads
   ksud with `late-load --kmi android15-6.6 --allow-shell`. Done when `/proc/modules`
   contains `kernelsu` (typically 3-5 s).
4. **Verify independently**: `su -c id` must print `uid=0(root) ... context=u:r:ksu:s0`.
   Trust this over the script's own verdict - see Pitfalls.
5. **Use root, then reboot** to release the parked page.

## Quick Reference

```bash
su -c id                       # uid=0, context=u:r:ksu:s0
grep -c kernelsu /proc/modules # 1 when loaded
uname -r                       # guard against a kernel change
```

## Failure Signatures

| Log line | Meaning | Action |
|---|---|---|
| `no zombie after Nms` | exec thread never finished inside the wait window | widen the wait, then check CPU contention |
| `burst hit=0 after 15000ms` | page freed, spray lost it to a system allocation | longer/faster spray; check throttling |
| `stage selinux failed` | SELinux-state write missed | treat as a lost race; reboot |
| `su verification FAILED` right after a real success | check ran before ksud finished loading | poll for `su` itself |

## Pitfalls

1. **The payload's own verdict is not the verdict.** It samples `/proc/modules` and `su`
   immediately after the plant; ksud can land a second later. Always confirm with your own
   `su -c id` before telling the user the run failed.
2. **Never retry in the same boot** after any park (`parking without reap`). Store the boot
   marker (`elapsedRealtime`) and refuse until it goes backwards.
3. **One payload change at a time.** The two failure modes (exec window, spray race) need
   different fixes; tuning both together teaches nothing.
4. **A rebuild is a different binary.** Rebuild with
   `aarch64-linux-android26-clang -Oz -s -static -pthread -Wl,--gc-sections` (NDK 27) and
   confirm the exploit's log strings are still present before shipping. Compare size against
   the known-good build as a smoke test.
5. **KASLR slide changes every boot.** Different `V=`, `A=` and `slide=` values between runs
   are expected, not a defect.
6. **Battery saver and low charge cause `burst hit=0`.** Block both in preflight instead of
   blaming the payload.
7. **Streaming callbacks double-print.** A runner that both delivers lines to a callback and
   returns the whole output makes every caller print the output twice; gate callback delivery
   to the streaming path only.
8. **`tail`-based log excerpts hide the line you need.** Include matching lines
   (`plant`, `zombie`, `burst`) in addition to the last N.

## Verification

- `uname -r` matches the pinned release
- `grep -c kernelsu /proc/modules` returns 1
- `su -c id` returns uid=0 with context `u:r:ksu:s0`
- The run's own evidence line records verdict, duration, kernel and failure signature

## Limits To State Out Loud

Root dies at reboot. Play Integrity still passes on the reference device (basic, device and strong
verdicts, measured with root active) because verified boot stays intact; apps with their own root
detection can still refuse.
A system update changes the kernel and kills the exploit until constants are re-derived -
`STATIC_TIMER_SITE` can only come from a live trace, not from symbols.
