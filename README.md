# iQOO Z8 kernel refclone probe

This repository is a 3.1-boundary probe for one specific iQOO Z8 / vivo PD2314 firmware build. It documents and exercises an observable kernel path: perf-based KASLR recovery, KernelSnitch/mm collision positioning, order-3 slab payload placement, futex/rt_mutex plus IP multicast side effect, temporary ashmem fops routing, restoration, and `route-summary`.

The word `route` is a source-level observation label. It is not a recommendation to build or continue a post-route exploitation chain. This package contains no privilege stage, no persistence stage, no binary deployment framework, no cross-device adaptation, and no second-stage claim.

## Scope and package facts

- The source tree keeps only the 3.1 path and earlier prerequisites. Later-stage constants, unrelated comments, and stale log residues have been removed.
- The package does not include a prebuilt `probe.so`, device dumps, external context copies, dmesg files, tombstones, full logcat captures, or captured console logs.
- Log snippets in the documentation explain fields and evidence boundaries. Real conclusions must come from stdout/logcat/dmesg collected on the exact target device.
- The worker checks the build fingerprint and kernel release before running. On mismatch it exits instead of probing.

## Safety baseline

Do not run unknown, unaudited, or unreproducible binaries on a device. Build locally from reviewed source and compare hashes on both host and device. Do not use this code on other models, other firmware builds, or third-party devices; it targets only the build below.

Running the probe can cause reboot, kernel panic, watchdog hang, temporary freeze, or test-time data loss; the operator accepts that risk. If the device becomes unresponsive, long-pressing volume-down plus power usually forces a restart and restores device access; if the model uses a different recovery key sequence, follow the official device instructions. On the exact target firmware and with a legitimate owner running it, this repository does not implement persistence or a privilege-resident stage, so permanent damage should not normally occur, but it cannot be guaranteed.

## Target build

| Item | Value |
| --- | --- |
| Device | iQOO Z8 / vivo PD2314 |
| Build fingerprint | `vivo/PD2314/PD2314:15/AP3A.240905.015.A2/compiler260617110852:user/release-keys` |
| Kernel release | `5.10.233-android12-9-g44ec642832da-dirty` |
| Output | `exploit/build/PD2314-AP3A.240905.015.A2/bin/probe.so` |

## Build and hash check

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35
sha256sum build/PD2314-AP3A.240905.015.A2/bin/probe.so
```

To build the two auxiliary test binaries as well:

```bash
make PROJECT=PD2314-AP3A.240905.015.A2 API=35
```

After pushing, verify the device-side file:

```bash
adb push build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell sha256sum /data/local/tmp/probe.so
```

## Logging-first run

The probe result depends on correlated logs. Use one terminal for stdout and another terminal for live logcat:

```bash
adb logcat -b all -v threadtime > logcat-live.txt
```

Run the probe:

```bash
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id' 2>&1 | tee probe-console.txt
```

If the device reboots or hangs, collect what remains after recovery:

```bash
adb shell getprop ro.boot.bootreason
adb shell dmesg > dmesg-after.txt
```

The `pr_*` logs write to stdout by default, and the supervisor makes stdout unbuffered. logcat is complementary system evidence; it is not guaranteed to contain the full probe stdout. ANSI color sequences may appear in terminal output and can be kept in the raw archive.

## Technical observation surface

| Stage | Key logs | What it shows | It does not prove alone |
| --- | --- | --- | --- |
| Target gate | `probe profile applied`, `probe context`, `probe build` | Worker entered the 3.1 profile on the expected build | Vulnerability presence |
| perf KASLR | `perf probe ring`, `candidate page`, `slide-kaslr-perf-ok` | Kernel IP samples selected a live text base | Later memory placement |
| mm/slab | `prepare_kernel_page geom`, `kernelsnitch collision-scan leaked mm` | mm collision produced an order-3 slab-near address | Payload hit the intended object |
| payload | `tcp payload geometry`, `final payload invariant ok` | The two user-space 32 KiB chunks are internally consistent | The kernel page now has that layout |
| spray | `af_unix order3 staged pairs`, `spray sent=4096` | AF_UNIX/sk_buff send count and first failure point | fops routing occurred |
| futex/rt_mutex | `FUTEX_CMP_REQUEUE_PI`, `slide ip side effect sched_ok=1` | PI requeue and consumer `sched_setattr` side effect occurred | The target slot was written |
| restore | `misc.fops restore ... ret=8 errno=0` | The original `ashmem_fops` pointer was written back | Every prior stage was stable |
| summary | `route-summary ... kaslr=1 ... route_done=1` | Worker reached the 3.1 end point | `route_done=1` is not a standalone proof |

The strongest current evidence is the correlation inside one run: target gate passed, perf base accepted, KernelSnitch collision and `mm` leak appeared, spray count completed, futex/sched side effect appeared, restore returned `ret=8`, and `route-summary` was printed. If a segment is missing, only the observations before that segment should be treated as supported.

## Runtime knobs

| Variable | Default | Purpose |
| --- | ---: | --- |
| `PERF_PROBE_MSEC` | `200` | perf sampling window; invalid values fall back to the default |
| `PAGE_PREP_SLABS` | `16` | mm/slab preparation scale; retry attempts raise it to `32` |
| `PROCESS_VM_CONSUMER_MAX_CALLS` | `1` | `sched_setattr` calls per consumer round |
| `SLIDE_IP_ROUTE_ATTEMPTS` | `1` | IP multicast side-effect route attempts |
| `SLIDE_IP_ROUTE_ARM_SEQ` | `1` | sequence that releases the consumer |
| `SLIDE_IP_MAIN_CONSUMER_DELAY_USEC` | `0` | consumer delay before `sched_setattr` |
| `SLIDE_IP_POST_SETSOCKOPT_HOLD` | `20000` | yield window before the consumer is released |

## Deep-dive docs

- [01-observation](docs/01-observation.md): log capture, fields, evidence ladder, and failure mapping.
- [02-information-leak](docs/02-information-leak.md): perf ring parsing, KASLR candidate selection, and limitations.
- [03-stack-write](docs/03-stack-write.md): mm/slab, payload, futex/rt_mutex, IP multicast side effect, and restore closure.

中文说明：[README.zh-CN.md](README.zh-CN.md).
