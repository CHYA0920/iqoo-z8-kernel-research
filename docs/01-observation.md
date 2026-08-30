# Stage 0 — Observation Infrastructure

The subject of this stage is not the kernel. It is the measurement
apparatus: without it, no claim about kernel behavior on a production
Android device can be falsified.

## The problem

On this device (vivo iQOO Z8, kernel 5.10.233, Android 15, SELinux
Enforcing, shell uid 2000), every standard kernel diagnostic channel is
closed to the unprivileged shell:

- `pstore` / console / kmsg: permission-denied across every access path
  tried.
- `kptr_restrict`: no symbol address reads.
- Kernel panics end the round; whatever was in flight is lost unless it
  was already on disk.

So the research program built its own channels first.

## [0.1] Automated build & deploy pipeline

The full loop — source edit, commit, cloud CI build (Android NDK r29),
artifact download with SHA256 identity check, device deployment via
adb, per-round device reboot for a clean boot state, test launch — runs
with no manual step. Each round's preflight prints the artifact hash so
that a mismatch between "what was built" and "what ran" is detectable
before the round counts.

The two published probes build through the same pipeline
(`mcast-test-z8`, `sched-test-z8` CI artifacts).

## [0.2] TCP microsecond probe channel

The on-device harness connects back to the host over an adb-reversed
TCP socket (port 18080) and emits a marker line per milestone:

- `CONN` — reverse tunnel established
- `EARLY` — pre-geometry milestones
- `HELLO` — payload armed, pre-fire state reached
- `FIRE` — trigger issued

Each marker carries a timestamp, giving microsecond-scale relative
timing of the whole round from the host side. The decisive property:
the channel is a plain TCP connection owned by a userspace process, so
it survives kernel-side trouble long enough to report that trouble —
rounds in which the kernel died shortly after the fire stage still
delivered their final markers to the host log.

## [0.3] O_DSYNC diagnostic persistence

The harness's diagnostic log file is opened with `O_DSYNC`: every write
is flushed to storage before the next statement executes. There is no
buffered log that dies with the kernel. After a panic-induced reboot,
the log is pulled (dual pull at T+120s and T+180s after fire) and
contains every line written before death — including the last one.

Together [0.2] and [0.3] close the loop: the host-side probe log tells
you *when* things happened at microsecond resolution, and the
device-side O_DSYNC log tells you *what* happened up to the last
surviving statement.

## Environment facts (measured)

| Item | Value |
|---|---|
| Device | vivo iQOO Z8 (PD2314 / V2314A) |
| SoC | MediaTek Dimensity 8200 (MT6895) |
| Kernel | 5.10.233-android12-9 (MTK vendor tree) |
| Android | 15 (SDK 35) |
| SELinux | Enforcing |
| Runtime identity | uid=2000 (shell) |
| VA_BITS | 39 |
| perf_event_paranoid | ≤ 1 (unprivileged sampling allowed) |
| 32-bit support | armeabi-v7a in abilist (compat syscall paths reachable) |
| Kernel config | CFI enabled; SLAB freelist hardening + randomization |

These are the preconditions the published stages rely on. The two most
load-bearing ones: unprivileged perf sampling (feeds [1.1]) and 32-bit
compat syscall reachability (feeds [2.3]).
