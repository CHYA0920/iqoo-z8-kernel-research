# 02. perf KASLR probe

KASLR 解析在 `rc_perf_leak_text_base()` 中完成。它不依赖硬编码 live base，而是从 perf ring buffer 里的 kernel IP sample 反推当前启动的 image base。

## perf_event_open 参数

核心配置：

| 字段 | 值 | 目的 |
| --- | --- | --- |
| `type` | `PERF_TYPE_SOFTWARE` | 使用软件事件 |
| `config` | `PERF_COUNT_SW_CPU_CLOCK` | 让 CPU clock 周期性产生 sample |
| `sample_period` | `1` | 提高短窗口内 sample 密度 |
| `sample_type` | `PERF_SAMPLE_IP` | ring 里只需要 IP |
| `exclude_user` | `1` | 过滤用户态 IP |
| `disabled` | `1` | mmap ring 后再显式 enable |

默认采样窗口是 `PERF_PROBE_MSEC=200` ms，可用环境变量调整。低于 10 ms 或高于约 2 s 的输入会被拒绝并回退到默认值。

## ring buffer 解析

`rc_perf_collect()` 按 `perf_event_header` 逐条推进：

- `type == PERF_RECORD_SAMPLE` 时读取 `tail + 8` 的 IP。
- `misc & 7 == 1` 时认为样本来自 kernel。
- IP 小于 `0xFFFFFFC008000000` 的样本不计入 kernel text 候选。
- IP 按 2 MiB 对齐页归桶：`ip & 0xFFFFFFFFFFE00000`。
- malformed record、lost record、bucket overflow 都会进入日志，作为候选质量判断依据。

## base 候选选择

`rc_perf_select_text_page()` 使用两个特征选 base：

1. 同时存在 `pg` 与 `pg + 0x1c00000` 两组样本。
2. 以 `pg` 开始的 `0x2000000` 窗口覆盖绝大多数 kernel sample。

如果候选页 2 MiB 对齐、落在 arm64 kernel text 合理范围内，并且窗口覆盖率足够，函数返回该页作为 `kaslr_base`：

```text
[*] perf probe candidate page=ffffffc0... window=... near=... far=...
[+] perf probe text-base=ffffffc0... min_kip=... samples=...
[+] slide-kaslr-perf-ok pid=... base=ffffffc0... slide=...
```

失败时会看到 `perf probe no usable kernel samples`、`perf probe rejected histogram` 或 `slide kaslr leak failed`。

## 输出如何被后续使用

`kaslr_base` 设置后，`text_addr(x)` 会把 `KIMAGE_TEXT_BASE` 上的静态符号转换成当前启动的 live 地址：

```c
return kaslr_base + (image_addr - KIMAGE_TEXT_BASE);
```

后续 fake fops 表、`init_task`、调度 task group 和 restore 原始 `ashmem_fops` 都通过这个函数换算。
