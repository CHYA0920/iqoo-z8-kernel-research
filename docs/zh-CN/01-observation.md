# 阶段 0 —— 观测基础设施

本阶段的对象不是内核,而是测量装置:没有它,任何关于生产 Android 设备内核行为的命题都无法证伪。

## 问题所在

在本设备(vivo iQOO Z8,内核 5.10.233,Android 15,SELinux Enforcing,shell uid 2000)上,非特权 shell 的每一条标准内核诊断通道都被封死:

- pstore / console / kmsg:试过的全部访问路径均权限拒绝。
- kptr_restrict:符号地址不可读。
- 内核 panic 直接终结轮次;未落盘的瞬态全部丢失。

所以研究计划先建了自己的通道。

## [0.1] 自动化构建与部署流水线

完整回路——改源码、commit、云端 CI 构建(Android NDK r29)、带 SHA256 身份核对的产物下载、adb 部署、每轮设备 reboot 取得干净 boot 状态、测试发射——全程无人工步骤。每轮 preflight 打印产物哈希,"构建的是什么"与"跑的是什么"不一致在轮次计数前即可察觉。

两个公开探针走同一条流水线(CI 产物 mcast-test-z8、sched-test-z8)。

## [0.2] TCP 微秒级探针通道

设备侧 harness 经 adb reverse TCP(端口 18080)连回主机,每里程碑发射一行标记:

- CONN —— reverse 隧道建立
- EARLY —— 几何前里程碑
- HELLO —— payload 就绪,到达 fire 前状态
- FIRE —— 触发已发出

每条标记带时间戳,主机侧以微秒分辨率获得整轮相对时序。决定性属性:该通道是用户态进程持有的普通 TCP 连接,在内核侧出事后仍存活足够久,足以上报——内核在 fire 段后死掉的轮次仍把最终标记送进了主机日志。

## [0.3] O_DSYNC 诊断落盘

harness 的诊断日志文件以 O_DSYNC 打开:每次写在下一条语句执行前刷入存储。不存在随内核一起死掉的缓冲日志。panic 引发的重启后,日志被拉取(fire 后 T+120s 与 T+180s 双拉),包含死前写下的每一行——包括最后一行。

[0.2] 与 [0.3] 一起闭合回路:主机侧探针日志以微秒分辨率告诉你事件"何时"发生,设备侧 O_DSYNC 日志告诉你"发生了什么",直到最后一个幸存语句。

## 环境事实(实测)

| 项 | 值 |
|---|---|
| 设备 | vivo iQOO Z8 (PD2314 / V2314A) |
| SoC | MediaTek Dimensity 8200 (MT6895) |
| 内核 | 5.10.233-android12-9 (MTK 厂商树) |
| Android | 15 (SDK 35) |
| SELinux | Enforcing |
| 运行身份 | uid=2000 (shell) |
| VA_BITS | 39 |
| perf_event_paranoid | ≤ 1(允许非特权采样) |
| 32 位支持 | abilist 含 armeabi-v7a(compat syscall 路径可达) |
| 内核配置 | CFI 启用;SLAB freelist 硬化 + 随机化 |

这些是已公开阶段所依赖的前提。承重最大的两条:非特权 perf 采样(喂 [1.1])与 32 位 compat syscall 可达性(喂 [2.3])。
