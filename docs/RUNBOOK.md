# Runbook

One page, per attempt.

## Before the run

- [ ] The phone was **rebooted** since the last attempt (not just the app restarted)
- [ ] Charge **>= 30%**
- [ ] **Battery saver off** — it caps CPU and the spray loses the page
- [ ] Nothing heavy running in the background
- [ ] `uname -r` still `6.6.118-android15-8-g9bc34d5b0c79-abogki537459655-4k`

A failed preflight on any of these is cheaper than a lost boot.

## The run

1. Press Start **once**.
2. Watch for `kernelsu loaded after Ns` — normally 3-5 s.
3. Confirm independently: `su -c id` must show `uid=0(root)` and `context=u:r:ksu:s0`.

Do not judge the run by the payload's own verification block. It samples `/proc/modules` and
`su` before ksud has finished loading and prints `su verification FAILED` on successful runs.

## If it rooted

Use root while it lasts. Reboot when done to release the parked page.

## If it failed

Read the timer tail and classify:

| Line | Class | Next attempt |
|---|---|---|
| `no zombie after Nms` | exec never completed | reboot, retry; if it repeats, widen the wait window |
| `burst hit=0 after 15000ms` | spray lost the page | reboot; check charge and battery saver first |
| `stage selinux failed` | SELinux write missed | reboot, retry |
| `parking without reap` | a page is held | **reboot** — a retry in this boot cannot succeed |

Then reboot. Every class ends the boot for this purpose.

## Never

- Never retry in the same boot after a park.
- Never run two attempts back to back hoping the second lands.
- Never change two payload parameters between attempts.
