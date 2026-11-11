# RootNow

**Temporary root for locked-bootloader OPPO / ColorOS phones.** One tap, about five seconds, no
unlock, no fastboot, no data wipe.

![status](https://img.shields.io/badge/status-field--tested-3DDC84?style=flat-square)
![tested](https://img.shields.io/badge/tested-Android%2017%20%C2%B7%20ColorOS%2017-3DDC84?style=flat-square)
![kernel](https://img.shields.io/badge/kernel-6.6.118--android15--8-blue?style=flat-square)

OPPO stopped handing out bootloader unlocks, so the usual route — unlock, flash a patched boot
image, install KernelSU — is closed. This repo is the other route: a kernel timer-page exploit
that wins root on a locked device, the Android app that runs it, and the operational discipline
that turns a ~60% race into a tool you can actually use.

Tested on **OPPO Find X8s (PKT110, MT6991)**, ColorOS 17 CN, Android 17, bootloader locked,
kernel `6.6.118-android15-8-g9bc34d5b0c79-abogki537459655-4k`.

## What you get

- `uid=0(root)` with SELinux context `u:r:ksu:s0` — real root, not a fake shell
- KernelSU late-loaded (`ksud --kmi android15-6.6`), so `su` and its module path exist
- Survives app restarts, **dies at reboot** — reboot is also the cleanup
- Reversible by design: each run parks one kernel page, and a reboot releases it

## Quick start

```bash
sha256sum -c dist/SHA256SUMS      # 1. verify
adb install -r dist/RootNow-v4.8.apk   # 2. sideload (debug-signed)
```

1. **Reboot** the phone before the first run — a parked page from an earlier attempt blocks every
   retry in that boot.
2. Charge above **30%** and turn **battery saver off**. The spray is CPU-bound; a throttled phone
   loses the page race.
3. Press **Start** once. Root lands in 3–5 s.
4. Confirm independently: `su -c id` → `uid=0(root) … context=u:r:ksu:s0`.

If it fails, **reboot before retrying**. The app enforces this with a boot marker instead of
letting you burn another boot.

## How it works

1. **Plant** — a freed timer page is reused so a controlled object lands where the kernel will use
   it (page ids `277,275`, free set `275,276,279` on this build).
2. **Fake credentials** — that object is shaped into a credential struct, so a child process
   reports `uid=0` while the spray holds the page.
3. **Late-load KernelSU** — with the primitives in place, `ksud` loads the KernelSU module and `su`
   becomes available.

Every constant is derived from the matching `vmlinux`; the recipe and the address table are in
[`docs/BUILD.md`](docs/BUILD.md).

## Measured, not promised

| Metric | Result |
|---|---|
| Success on first plant | **6 of 7 runs** |
| Time to root | **3–5 s** (`kernelsu loaded after 3-5s`) |
| Failure classes seen | `no zombie` ×2, `burst hit=0` ×2 |
| Cost of a lost run | one reboot (~1 min) |
| Payload rebuild | reproducible, byte-verified strings |

The exploit is a race, not a switch. The two failure modes need different fixes:

| Log line | Meaning | What changed |
|---|---|---|
| `no zombie after Nms` | exec thread didn't finish inside the wait window | window widened 400 ms → 10 s |
| `burst hit=0 after 15000ms` | page freed, spray lost it to a system allocation | preflight blocks low charge / battery saver |
| `stage selinux failed` | SELinux-state write missed | treat as a lost race; reboot |
| `su verification FAILED` | usually **not** a failure — the check ran before ksud finished | app re-verifies and overrules it |

Per-attempt checklist: [`docs/RUNBOOK.md`](docs/RUNBOOK.md).

## Limits — read before installing

- **Root dies at reboot.** Every boot needs a new run.
- **Play Integrity trips.** Banking and some payment apps may refuse to run with root present.
- **Pinned to one kernel build.** A system update breaks the exploit until the constants are
  re-derived; `STATIC_TIMER_SITE` can only come from a live trace, never from symbols.
- **A lost race can panic the device.** It happened once here, on a first-boot attempt.
- **One device tested.** This is not a claim about other OPPO, OnePlus or realme models.

**Not for you if:** your bootloader can be unlocked (use fastboot and a proper KernelSU install),
you need root across reboots, or you are not willing to reboot to recover.

## Repo layout

```
README.md           this file
SKILL.md            the Hermes skill: triggers, procedure, pitfalls
docs/RUNBOOK.md     per-attempt checklist and the failure → action table
docs/BUILD.md       payload rebuild recipe, constants, tuning knobs
docs/APP-NOTES.md   boot guard, independent verification, log integrity
dist/               the APK and its checksum
```

What is **not** here: the payload source, the KernelSU daemon source, and the Android app source.
The APK in `dist/` does bundle the compiled payload and `ksud` as assets — it cannot run without
them.

## Credits

- [KernelSU](https://github.com/tiann/KernelSU) — the in-kernel `su` this exploit late-loads
- [Magisk](https://github.com/topjohnwu/Magisk) — the prior art that made Android root legible

## License

MIT — see [LICENSE](LICENSE). Use it only on hardware you own.
