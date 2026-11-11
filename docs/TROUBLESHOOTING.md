# Troubleshooting

Symptom first, cause second, action last.

## The app refuses to start

Every blocker is printed with its code. These are the codes:

| Code | Means | Fix |
|---|---|---|
| `BATTERY_LOW` | charge under 30% | charge to 50%, it costs one reboot per miss |
| `POWER_SAVE_ON` | battery saver is on | turn it off — it caps the CPU the spray needs |
| `THERMAL_HOT` | device is throttling | let it cool; a hot phone loses the race |
| `STORAGE_LOW` | not enough free space for the payload | free space, payload is ~7 MB |
| `PAYLOAD_MISSING` | `timer`, `ksud` or `diag.sh` are not staged | reinstall the app; it copies them on first run |
| `PAYLOAD_SHORT` | a staged file is truncated | reinstall — the copy was interrupted |
| `PAYLOAD_CORRUPT` | staged file's SHA-256 does not match the build | reinstall, or the app is from a different build |
| `SHIZUKU_DOWN` | the runner service is not bound | install and start Shizuku, grant RootNow permission |

A battery check that passes at 32% is not a promise: if battery saver turns itself on at 20%, the
run you start now may miss. Charge first, then run.

## "Reboot required" when pressing Start

Working as designed. A previous attempt parked a kernel page; a retry in the same boot cannot
succeed. Reboot — the marker clears itself because it compares against `elapsedRealtime()`.

## `no zombie after 10000ms`

The exec thread never finished. On this device the measured exec latency was 296 ms, so anything
near the old 400 ms window was luck. If you see this with the 10 s window, the exec thread is
being starved: the spray runs eight threads pinned one per CPU. That is a payload tuning problem,
not something a retry fixes — reboot and retry once, and if it repeats, report it with the log.

## `burst hit=0 after 15000ms`

The page was freed and the spray lost it to another allocation. Check, in this order: charge,
battery saver, thermal state, background activity. All three were responsible for real failures
here. Reboot before retrying.

## `su: inaccessible or not found` / `SU_RC=127`

ksud never loaded, so there is no `su` binary. This follows a failed stage or a parked process —
the fix is a reboot and another run, not a reinstall.

## The app says ROOTED but another app says no root

Check, in order:

1. `su -c id` from a terminal — if it prints `uid=0 … u:r:ksu:s0`, root exists
2. Open the KernelSU manager — if it says **Working**, the kernel side is fine
3. In the manager, the requesting app must be granted superuser access; the first request
   usually shows a prompt

Root is live only until reboot. If the phone restarted since the run, there is no root — that is
the design, not a fault.

## Banking / payment app refuses to run

Play Integrity fails while root is present. Nothing in this repo fixes that; reboot to drop root
before using those apps.

## Diagnose output

Expected on a freshly rooted phone:

```
PASS: kernelsu in /proc/modules
SKIP: zygiskd not installed
SKIP: zygisk module files not installed
SKIP: vectord not installed
SKIP: vector-cli not installed
SKIP: LSPosed not installed
PASS: zygote/system_server running
INFO: timer process parked (expected after a run; reboot cleans it)
=== diag summary: PASS=2 FAIL=0 SKIP=5 ===
```

`SKIP` is not a failure: those layers are optional and simply not installed. `INFO: timer process
parked` is the *expected* state after a run — the parked process is what holds the fake
credentials page.

A `FAIL: kernelsu in /proc/modules` after a run the app called successful means the module never
loaded; trust the diagnosis over the green card and check `su -c id`.

## Root worked yesterday, not today

Nothing about the exploit is persistent. Two things change between boots:

- **KASLR slide** — a new random slide every boot, which is normal and handled
- **The kernel** — if the phone installed an OTA, `uname -r` will no longer match
  `6.6.118-android15-8-g9bc34d5b0c79-abogki537459655-4k`, and the payload's constants no longer
  apply. The app warns about this at the start of every run.

A kernel change is the one failure that a reboot cannot fix. It needs the constants re-derived
(see [BUILD.md](BUILD.md)).
