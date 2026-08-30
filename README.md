# iQOO Z8 kernel refclone probe

This repository is a probe for the specified iQOO Z8 / vivo PD2314 kernel build. The current package stops at the 3.1 boundary: KASLR recovery, mm/slab positioning, ashmem fops routing, rt_mutex/futex trigger, ashmem fops restoration, and `route-summary`.

Post-3.1 residues have been removed. The current tree focuses only on the probe loop.

## Target build

| Item | Value |
| --- | --- |
| Device | iQOO Z8 / vivo PD2314 |
| Build fingerprint | `vivo/PD2314/PD2314:15/AP3A.240905.015.A2/compiler260617110852:user/release-keys` |
| Kernel release | `5.10.233-android12-9-g44ec642832da-dirty` |
| Output | `exploit/build/PD2314-AP3A.240905.015.A2/bin/probe.so` |

The worker checks the fingerprint and kernel release before running the probe.

## Build

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35
```

To build the two auxiliary test binaries as well:

```bash
make PROJECT=PD2314-AP3A.240905.015.A2 API=35
```

## Run

```bash
adb push exploit/build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id'
```

The entry point still uses an `LD_PRELOAD` constructor to start the worker, but the artifact name, source entry name, and logs now use probe terminology.

## Execution path

1. `preload.c` constructor enters `run_probe()`.
2. `main.c` checks `Z_REFCLONE=1` and enters `refclone_load()`.
3. The supervisor forks up to three worker attempts, each hosted by `/system/bin/toybox id`.
4. The worker validates the target build and sets only the defaults used by this route.
5. `rc_perf_leak_text_base()` derives the live KASLR image base from perf IP samples.
6. `rc_prepare_fops_page()` uses the KernelSnitch/mm reclaim window to locate an order-3 slab and spray two 32 KiB payload chunks.
7. `rc_run_main_route_threads()` coordinates waiter/owner/consumer threads and triggers the fops route via futex PI plus the IP multicast side effect.
8. The worker uses the configfs write window to restore `dashmem_misc.fops` to the original `ashmem_fops`.
9. `route-summary` is printed, then toybox `id` prints the current process identity.

## Runtime knobs

| Variable | Default | Purpose |
| --- | ---: | --- |
| `PERF_PROBE_MSEC` | `200` | perf sampling window for KASLR base selection |
| `PAGE_PREP_SLABS` | `16` | mm/slab preparation scale; retry attempts raise it to `32` |
| `PROCESS_VM_CONSUMER_MAX_CALLS` | `1` | `sched_setattr` calls per consumer round |
| `SLIDE_IP_ROUTE_ATTEMPTS` | `1` | IP multicast route attempts |
| `SLIDE_IP_ROUTE_ARM_SEQ` | `1` | sequence that releases the consumer |
| `SLIDE_IP_MAIN_CONSUMER_DELAY_USEC` | `0` | consumer delay before `sched_setattr` |
| `SLIDE_IP_POST_SETSOCKOPT_HOLD` | `20000` | yield window before the consumer is released |

## Docs

- [01-observation](docs/01-observation.md): entry sequence, log fields, and the 3.1 stop boundary.
- [02-information-leak](docs/02-information-leak.md): perf KASLR probe mechanics.
- [03-stack-write](docs/03-stack-write.md): mm/slab placement, payload layout, futex/rt_mutex route, and restore loop.

中文说明：[README.zh-CN.md](README.zh-CN.md).
