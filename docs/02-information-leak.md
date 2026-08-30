# 02. perf KASLR probe

`rc_perf_leak_text_base()` derives the live kernel text base from perf IP samples collected during the current boot. It does not treat an address copied from another log as fixed truth, and it does not use cross-device constants; it accepts only a candidate page that satisfies the clustering checks inside the same sampling window.

## perf_event_open setup

| Field | Current value | Purpose |
| --- | --- | --- |
| `type` | `PERF_TYPE_SOFTWARE` | software event source |
| `config` | `PERF_COUNT_SW_CPU_CLOCK` | dense CPU-clock samples |
| `sample_period` | `1` | short interval inside the 200 ms window |
| `sample_type` | `PERF_SAMPLE_IP` | only IP is needed from each record |
| `exclude_user` | `1` | filter user-space IPs |
| `disabled` | `1` | mmap first, then reset/enable |
| mmap data pages | `RC_PERF_PAGES=8` | one metadata page plus eight data pages |

The default collection window is `PERF_PROBE_MSEC=200`. Inputs below 10 ms or above the accepted range fall back to the default and log `duration out of range`.

## Ring parsing

`rc_perf_collect()` reads `data_head` / `data_tail` from the metadata page and walks the data ring by `perf_event_header`:

- `PERF_RECORD_SAMPLE`: read the 8-byte IP after the header.
- `PERF_RECORD_LOST`: accumulate lost records as sample-quality evidence.
- `misc & 7 == 1`: count the record as a kernel sample.
- IP below `RC_PERF_MIN_BASE=0xffffffc008000000` is excluded from text-base candidates.
- IP is bucketed with `RC_PERF_PAGE_MASK=0xffffffffffe00000`, giving 2 MiB-aligned buckets.
- malformed records, out-of-range records, and bucket overflow are counted and excluded from selection.

Useful logs include ring state, sample range, and selected candidate:

```text
[*] perf probe ring head=... tail=... records=... samples=... kernel=... lost=... malformed=...
[*] perf probe sampled duration_ms=200 disable=0/0 min=... max=...
[*] perf probe candidate page=... window=.../... near=... far=... buckets=...
[+] perf probe text-base=... min_kip=... samples=...
[+] slide-kaslr-perf-ok pid=... base=... slide=...
```

In one reference run, `records=2047 samples=2047 kernel=2047 lost=0 malformed=0` indicated a clean ring parse, and `window=2044/2047` showed that the selected window covered almost all kernel samples.

## Candidate selection

The selector uses two constraints:

1. Candidate page `pg` must have a visible bucket, and `pg + RC_PERF_PAIR_DELTA` must also have nearby samples. Current `RC_PERF_PAIR_DELTA=0x1c00000`.
2. The `RC_PERF_WINDOW=0x2000000` range starting at `pg` must cover the large majority of kernel samples. The current acceptance condition is approximately `best_window >= total - total / 4`.

After acceptance, `pg` becomes `kaslr_base`, and `kaslr_slide = kaslr_base - KIMAGE_TEXT_BASE`. Common failure logs:

```text
[-] perf probe no usable kernel samples
[-] perf probe rejected histogram best=... window=.../... buckets=...
[-] perf probe text base out of range=...
```

## Address-conversion boundary

perf only solves the live text base. Later source uses:

```c
return kaslr_base + (image_addr - KIMAGE_TEXT_BASE);
```

to convert static image addresses in `target.h` into current-boot addresses. This affects the CFI jump-table entries in the fake fops table, `init_task`, scheduler-related static objects, and the original `ashmem_fops` pointer used during restore.

It does not prove mm/slab collision success, payload placement, or fops routing. KASLR acceptance is only the first segment of the 3.1 evidence chain.

## Diagnostic points

| Log | Reading |
| --- | --- |
| `paranoid=-1` | perf policy allows shell sampling; other values need `open errno` context |
| `open retry ... attr_size=...` | kernel rejected the initial attr size, and the source retried with a smaller one |
| `lost > 0` | sampling pressure or ring size may have dropped data; candidate quality is weaker |
| `malformed > 0` | bad ring records were seen; handle the run cautiously |
| small `buckets` with concentrated `window` | candidate quality is usually good |
| both `near` and `far` present | pair-delta feature matched |
| non-zero `disable` errno | shutdown path was abnormal; preserve full logs, not just `base` |

Research notes should keep the full stdout, not just `base` and `slide`. Address values change across boots; the reproducible material is the sample count, coverage ratio, candidate-selection process, and whether later stages remain consistent with that base.
