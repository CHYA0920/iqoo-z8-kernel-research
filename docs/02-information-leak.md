# 02. perf KASLR probe

KASLR recovery is implemented in `rc_perf_leak_text_base()`. It does not assume a live base; it derives the current image base from kernel IP samples in the perf ring buffer.

## perf_event_open configuration

| Field | Value | Purpose |
| --- | --- | --- |
| `type` | `PERF_TYPE_SOFTWARE` | software event source |
| `config` | `PERF_COUNT_SW_CPU_CLOCK` | periodic CPU clock samples |
| `sample_period` | `1` | dense samples in a short window |
| `sample_type` | `PERF_SAMPLE_IP` | ring records only need IP |
| `exclude_user` | `1` | ignore user-mode IPs |
| `disabled` | `1` | enable only after mmap setup |

The default sample window is `PERF_PROBE_MSEC=200` ms. Out-of-range input falls back to the default.

## Ring parsing

`rc_perf_collect()` advances through `perf_event_header` records:

- `PERF_RECORD_SAMPLE` records read IP at `tail + 8`.
- `misc & 7 == 1` marks a kernel sample.
- IPs below `0xFFFFFFC008000000` are ignored.
- IPs are bucketed by 2 MiB text page: `ip & 0xFFFFFFFFFFE00000`.
- malformed records, lost records, and bucket overflow are logged and affect candidate quality.

## Base candidate selection

`rc_perf_select_text_page()` selects a base using two properties:

1. Both `pg` and `pg + 0x1c00000` have samples.
2. The `0x2000000` window beginning at `pg` covers most kernel samples.

If the candidate is 2 MiB aligned and within the expected arm64 kernel text range, it becomes `kaslr_base`:

```text
[*] perf probe candidate page=ffffffc0... window=... near=... far=...
[+] perf probe text-base=ffffffc0... min_kip=... samples=...
[+] slide-kaslr-perf-ok pid=... base=ffffffc0... slide=...
```

Failures show as `perf probe no usable kernel samples`, `perf probe rejected histogram`, or `slide kaslr leak failed`.

## How later stages use it

After `kaslr_base` is set, `text_addr(x)` translates static `KIMAGE_TEXT_BASE` symbols into live boot addresses:

```c
return kaslr_base + (image_addr - KIMAGE_TEXT_BASE);
```

The fake fops table, `init_task`, scheduler task-group pointer, and original `ashmem_fops` restore value all use this translation.
