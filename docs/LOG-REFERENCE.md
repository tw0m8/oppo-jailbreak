# Reading the exploit log

Every line the payload prints, and what it means when you are staring at a failed run at 1 a.m.

```
=== attempt 1/3
=== parked a zombie after 17s
=== timer log tail (attempt 1)
...
```

The wrapper makes up to three attempts in one script run. In practice the first plant either
works or parks the process — a parked process cannot be retried in the same boot.

## The plant

```
[+] plant1 V=0xffffff80ff910630 A=0xffffff80ff9106a8 pfn=0x17f910 slab=0x17f910
    slide=0x1642400000 recs=4096 slab_own=31/32 base=0xffffff7f80000000
```

| Field | Meaning |
|---|---|
| `V` | victim object — the timer object the exploit will act on |
| `A` | the address the payload writes to (derived from `V`) |
| `pfn` / `slab` | physical frame number of the victim page and its slab page |
| `slide` | KASLR slide for **this boot** — it changes on every reboot, so addresses never match between runs |
| `recs` | trace records scanned to find the victim |
| `slab_own` | how many of the 32 objects on that slab page belong to the payload; `31/32` is the healthy shape |
| `base` | linear-map base the payload resolved |

A different `slide`, `V` or `A` on every boot is expected. It is not a defect and not a
regression.

## The reaping

```
[+] plant1 target=15420 etid=15425 zombie code=2 status=9 after 296ms
```

The child was reaped as a zombie. `code=2` is `CLD_KILLED`, `status=9` is the SIGKILL the
exploit sends once the exec thread is gone. **`after` is the exec latency** — the number that
matters. On a healthy run it is tens to a few hundred milliseconds; 296 ms was measured on the
reference device, which is why a 400 ms wait window was marginal.

Absence of this line, followed by:

```
[-] plant1 no zombie after 10000ms; holding child
```

means the exec never completed inside the window. Widen the window first; if it persists, the
exec thread is being starved by the spray (eight threads, one CPU each).

## The free and the spray

```
[+] plant1 free_seen=1 burst hit=1 after 1237ms
[+] plant1 free_seen=1 burst hit=0 after 15000ms
```

`free_seen=1` means the victim page reached the allocator. `hit=1` means the spray reclaimed it,
`after` is how long that took — 1.2 s on the reference device. `hit=0` after the full window means
something else took the page first: charge level, battery saver and thermals are the usual causes,
because the spray is CPU-bound.

## The stages

```
[-] stage fake_uid failed
[-] stage fake_cred failed
```

Which primitive failed. `fake_uid` comes first (the credential struct reports `uid=0`), then
`fake_cred`. A failure at either stage parks the process.

Success looks like this instead:

```
[fake] child uid=0 euid=0; forking worker
[+] child report uid=0 euid=0 setuid(0)=0 errno=0 resuid=0/0/0
[fake] worker exit=0
[+] child exit status=0
[!] fake cred page held; reboot to clean -- parking without reap (zombie held)
```

That last line reads like an error and is not: on success the exploit deliberately keeps the page
so the fake credentials stay valid. A reboot is the clean way to release it.

## KernelSU

```
=== kernelsu loaded after 4s
=== verification 1
uid=0(root) gid=0(root) groups=0(root) context=u:r:ksu:s0
```

`kernelsu loaded after Ns` is the real time-to-root. `verification 1` is the module count from
`/proc/modules`. The `uid=0` line is the payload's own `su` check.

The wrapper's own verdict can be wrong in the pessimistic direction: it samples `/proc/modules`
and `su` the instant the plant returns, and ksud sometimes lands a second later. A line reading
`su verification FAILED` directly after `verification 1` is that race, not a failed run. Confirm
with `su -c id` yourself.

## Parked vs. failed

| Line | State | What to do |
|---|---|---|
| `parking without reap (zombie held)` | process parked, page held | **reboot** |
| `fake cred page held; reboot to clean` | parked *after success* | use root, reboot when done |
| `no zombie after Nms` | plant failed | reboot, retry |
| `burst hit=0 after Nms` | spray lost the page | reboot, check charge/saver/thermals |

Either way, the boot is over for exploit purposes.
