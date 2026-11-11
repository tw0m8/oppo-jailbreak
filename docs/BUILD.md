# Building and verifying the payload

The payload is a static aarch64 binary that must be built with the same recipe every time, and
verified before it goes near the phone.

## Toolchain

NDK 27.2.12479018:

```
$NDK/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android26-clang
```

## Build line

```bash
aarch64-linux-android26-clang -Oz -s -static -pthread -Wl,--gc-sections -o timer timer.c
```

| Build | Size |
|---|---|
| shipped, known-good | 309,304 B |
| `-Oz -s -static -pthread -Wl,--gc-sections` | 303,792 B |
| `-O2 -s -static -pthread -Wl,--gc-sections` | 307,888 B |

Size is a smoke test, not proof: within ~2% of the shipped binary is the expected range. An
`-O2` build exists as a fallback if a size-optimised build measures slower on device.

## Verify before shipping

1. **Log strings survive the rebuild.** Every exploit message the operators rely on must still
   be present — 33 of them on this build: `plant%d`, `stage `, `parking`, `free_seen`, `burst`,
   `fake cred`, `selinux`.
2. **Digest matches what the app expects.** The app verifies the payload against a pinned
   SHA-256; updating the binary without updating that constant makes every run fail preflight.
3. **Log-tail evidence.** Keep the timing lines visible (`zombie ... after Nms`) — a `tail -N`
   that drops them makes the run undiagnosable.

## Constants for this kernel

Verified against the matching `vmlinux`:

| Symbol | Address |
|---|---|
| `selinux_state` | `0xffffffc08239c620` |
| `panic_on_warn` | `0xffffffc08214b2cc` |
| `root_user` | `0xffffffc08216f6e0` |
| `init_user_ns` | `0xffffffc08216f450` |
| `init_ucounts` | `0xffffffc0821711b0` |
| `init_groups` | `0xffffffc082170c40` |
| `STATIC_TIMER_SITE` | `0xffffffc0801a4dfc` |

`STATIC_TIMER_SITE` is a call site, not a symbol: it cannot be looked up, only captured from a
live trace. After a kernel update, re-derive the symbols from that build's `vmlinux` and the
site from one instrumented run.

## Tuning, one variable at a time

| Knob | Current | Effect |
|---|---|---|
| zombie wait window | 10 s | a successful exec returns on the first poll; the window only bounds failure |
| `SPRAY_THREADS` | 8, pinned one per CPU | spray throughput vs CPU available to the exec thread |
| `SPRAY_PRE` / `SPRAY_POST` | 2000 / 10000 | how long the allocation burst runs |
| per-write sleep | 50 µs | pacing that keeps the trace scan in step |

Change one, run the phone five times, compare against the previous five. Tuning two at once
makes the result unreadable.
