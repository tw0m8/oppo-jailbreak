# App-side notes

The exploit is roughly 60% per attempt on a marginal boot. Most of the engineering that made it
feel reliable lives in the app around it, not in the payload.

## The boot guard

A parked attempt holds a kernel object for the life of the boot, so a retry cannot succeed.
The app stores the boot marker (`elapsedRealtime`) when it sees `parking without reap`, and
refuses further runs until that value goes backwards — i.e. until the phone really rebooted.

Without this, a user taps Start again, waits 17 s, gets the same failure, and concludes the
exploit is broken.

## Independent verification

The script's own check is a false negative machine: it samples `/proc/modules` and `su`
immediately after the plant, while ksud is still coming up. Successful runs printed
`su verification FAILED` and then verified fine a second later.

The app therefore re-verifies after the script ends and trusts only `su -c id`:
`uid=0(root) ... context=u:r:ksu:s0`. When it overrules the script it says so in the log.

## Preflight that prevents wasted boots

- Battery floor **30%** — a miss costs a reboot, and low charge usually means power-save
- Battery saver blocked outright — it caps CPU, which is what the spray needs
- Kernel banner on every run, with a warning when `uname -r` changes

## Log integrity

Two bugs worth not repeating:

1. **Double delivery.** A runner that both streams lines to a callback *and* returns the full
   output makes every caller print everything twice — visible as doubled diagnostics and
   `loaded: 00`. Gate callback delivery to the streaming path only.
2. **Helpful tails that hide the answer.** `tail -6` dropped the one line (exec timing) added to
   diagnose failures. Excerpts must include the lines you grep for, not just the last N.

## Telemetry

One CSV row per run — verdict, failure signature, duration, battery, power-save flag, kernel —
and a running `N/M rooted (P%)` line. After a week this answers "is it getting worse?" with a
number instead of a feeling, and separates the two failure classes.
