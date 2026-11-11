# Changelog

Version codes in parentheses are the Android `versionCode`.

## 4.9 (39)

- **Live log.** The run path called `run()`, which only returns the whole output once the script
  finishes, so the screen sat on "Running script (streaming)" for the entire run. It now uses
  `runStream()`. Same command, same result string, lines delivered as they arrive.
- **`kernelsu modules loaded: 00`.** `grep -c` exits 1 when the count is zero, so the `|| echo 0`
  fallback fired *in addition to* grep's own `0` and both were concatenated. Now
  `n=$(grep -c …); echo ${n:-0}`.

## 4.8 (38)

- The timer log excerpt shows the lines that matter (`plant`, `zombie code`, `burst hit`) in
  addition to the last six lines. `tail -6` had been hiding the exec timing that 4.7 introduced.

## 4.7 (37)

- **First payload rebuilt from source.** `wait_zombie` window raised from **400 ms to 10 s** and
  exec latency is now logged (`zombie … after 296ms`). A successful exec returns on the first
  poll, so the longer ceiling costs nothing on the happy path.
- Rebuild recipe: `-Oz -s -static -pthread -Wl,--gc-sections`, NDK 27, 304,048 B against the
  previously shipped 309,304 B, all 33 exploit log strings verified present.

## 4.6 (36)

- **Telemetry.** One CSV row per run — verdict, signature, duration, battery, power-save flag,
  kernel — plus a running `N/M rooted (P%)` line in the log.

## 4.5 (35)

- **Duplicate output fixed at the source.** `RunnerService.run()` delivered every line to the
  streaming callback *and* returned the full text, so callers printed everything twice. Callback
  delivery is now gated to `runStream()`.

## 4.4 (34)

- The verification block polls for `su` itself, not just for `/proc/modules`. Successful runs had
  been printing `su verification FAILED` because the check ran before the su binary appeared.

## 4.3 (33)

- Battery floor **15% → 30%**, and charging no longer exempts it — a miss costs a full reboot.
- New `POWER_SAVE_ON` blocker: battery saver caps the CPU the spray depends on.

## 4.2 (32)

- A park is no longer treated as failure on sight: after `parking without reap` the script polls
  `/proc/modules` for up to 6 s before declaring a lost run.
- The app states when it overrules the script: *"script reported a failure, but root is present"*.
- `diag` ran twice and printed twice because `(su -c …) || sh …` fell through on su's exit code;
  it now tests `su -c true` first and runs exactly once.

## 4.1 (31)

- Script output was printed twice (streamed, then appended again as a 500-char tail). The tail is
  now a fallback for when nothing streamed.
- `diag` reports `SKIP` for components that are not installed instead of `FAIL`, and the parked
  timer is `INFO` rather than a failure. The LSPosed check had been setting its flag inside a
  pipeline subshell, so it could never pass.
- Post-root note: root lives until reboot, and a reboot releases the held page.

## 4.0 (30)

- **Boot guard.** A parked attempt stores the boot marker and further runs are refused until a
  real reboot. Before this, a second tap in the same boot could never work.
- **Kernel banner.** Every run logs `uname -r` and warns when it is not
  `6.6.118-android15-8`, because offsets from a different image are how a phone panics.
