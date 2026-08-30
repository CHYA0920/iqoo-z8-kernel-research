# 02. perf KASLR probe

`rc_perf_leak_text_base()` 的任务是从当前启动的 perf IP sample 中推导 live kernel text base。它不把日志里的某个地址当成固定事实，也不使用跨设备常量；它只接受同一次采样窗口内满足聚类条件的候选页。

## perf_event_open 配置

| 字段 | 当前值 | 作用 |
| --- | --- | --- |
| `type` | `PERF_TYPE_SOFTWARE` | 使用软件事件源 |
| `config` | `PERF_COUNT_SW_CPU_CLOCK` | 以 CPU clock 产生高密度 sample |
| `sample_period` | `1` | 尽量缩短 200 ms 窗口内的采样间隔 |
| `sample_type` | `PERF_SAMPLE_IP` | ring 中只需要 IP |
| `exclude_user` | `1` | 过滤用户态 IP |
| `disabled` | `1` | mmap ring 后再 reset/enable |
| mmap data pages | `RC_PERF_PAGES=8` | 1 个 metadata page + 8 个 data pages |

默认采样窗口是 `PERF_PROBE_MSEC=200`。输入值小于 10 ms 或过大时会回退默认，并在日志里打印 `duration out of range`。

## ring buffer 解析

`rc_perf_collect()` 读取 metadata page 的 `data_head` / `data_tail`，按 `perf_event_header` 逐条推进 data ring：

- `PERF_RECORD_SAMPLE`：读取 header 后的 8 字节 IP。
- `PERF_RECORD_LOST`：累计 lost 计数，用来判断样本质量。
- `misc & 7 == 1`：该 sample 计入 kernel sample。
- IP 小于 `RC_PERF_MIN_BASE=0xffffffc008000000` 的样本不进入 text base 候选。
- IP 用 `RC_PERF_PAGE_MASK=0xffffffffffe00000` 做 2 MiB 对齐归桶。
- malformed record、越界 record、bucket 溢出都进入统计，不参与候选。

有效日志会同时给出 ring 状态、样本范围和候选页：

```text
[*] perf probe ring head=... tail=... records=... samples=... kernel=... lost=... malformed=...
[*] perf probe sampled duration_ms=200 disable=0/0 min=... max=...
[*] perf probe candidate page=... window=.../... near=... far=... buckets=...
[+] perf probe text-base=... min_kip=... samples=...
[+] slide-kaslr-perf-ok pid=... base=... slide=...
```

参考日志里 `records=2047 samples=2047 kernel=2047 lost=0 malformed=0` 说明该次 ring 没有明显丢失或坏记录；`window=2044/2047` 表示候选窗口覆盖了绝大多数 kernel sample。

## 候选页选择

选择函数使用两层约束：

1. 候选页 `pg` 必须存在明显样本桶，并且 `pg + RC_PERF_PAIR_DELTA` 也有相邻样本。当前 `RC_PERF_PAIR_DELTA=0x1c00000`。
2. 从 `pg` 开始的 `RC_PERF_WINDOW=0x2000000` 覆盖窗口必须包含绝大多数 kernel samples；当前接受条件约等于 `best_window >= total - total / 4`。

通过后，`pg` 被记为 `kaslr_base`，`kaslr_slide = kaslr_base - KIMAGE_TEXT_BASE`。失败时常见日志是：

```text
[-] perf probe no usable kernel samples
[-] perf probe rejected histogram best=... window=.../... buckets=...
[-] perf probe text base out of range=...
```

## 地址换算边界

perf probe 只解决 live text base。后续源码通过：

```c
return kaslr_base + (image_addr - KIMAGE_TEXT_BASE);
```

把 `target.h` 中的静态 image 地址换算成当前启动地址。它会影响 fake fops 表中的 CFI jump-table 入口、`init_task`、`sched` 相关静态对象和 restore 时的原始 `ashmem_fops` 地址。

它不证明 mm/slab 碰撞成功，不证明 payload 已经落进目标页，也不证明 fops route 已经发生。KASLR 有效只是 3.1 证据链的第一段。

## 诊断要点

| 日志 | 判断 |
| --- | --- |
| `paranoid=-1` | perf 策略允许当前 shell 采样；其他值需结合 `open errno` 判断 |
| `open retry ... attr_size=...` | 设备内核不接受初始 attr size，源码会降级重试 |
| `lost > 0` | 采样压力过高或 ring 太小；候选仍可能有效，但证据质量下降 |
| `malformed > 0` | ring 解析遇到坏记录；该次应谨慎处理 |
| `buckets` 很少且 `window` 很集中 | 候选质量较好 |
| `near/far` 都存在 | 符合 pair-delta 特征 |
| `disable` errno 非 0 | 采样关闭阶段异常；保留日志，不要只看最后的 base |

研究记录里建议保存完整 stdout，而不是只摘 `base` 和 `slide`。地址值会随启动变化；可复核的是采样数量、覆盖比例、候选选择过程和后续阶段是否与该 base 一致。
