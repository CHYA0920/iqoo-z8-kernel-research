English mirror: [README.md](README.md).

# iQOO Z8 kernel refclone probe

这是一个面向 iQOO Z8 / vivo PD2314 指定固件的 3.1 边界探测仓库。它研究的是一条可观测的内核路径：perf 采样解析 KASLR、KernelSnitch/mm 碰撞定位、order-3 slab payload 落点、futex/rt_mutex 与 IP multicast side effect、ashmem fops 临时路由、还原以及 `route-summary`。

这里的 `route` 是源码里的内部观测标签，不表示推荐或构造后续利用链。当前包不包含提权阶段、持久化阶段、二进制投放逻辑、跨设备适配，也不声明任何第二阶段效果。

## 范围与材料事实

- 当前源码只保留 3.1 以及 3.1 之前的探测路径；后续阶段常量、无关注释和旧日志残留已经清掉。
- 包内不附带预编译 `probe.so`，也不附带设备 dump、上下文副本、dmesg、tombstone、完整 logcat 或外部采集日志文件。
- 文档中的日志片段用于说明字段和判定方式；真正结论必须来自执行者在目标设备上采集到的 stdout/logcat/dmesg。
- worker 会检查 build fingerprint 和 kernel release；不匹配时直接退出，不继续探测。

## 安全前提

不要在设备上运行来源未知、不可复核、不可重建的二进制。建议从本仓库本地构建，并在 host 与 device 两侧核对哈希。不要把本代码用于其他机型、其他固件或第三方设备；它只针对下表这一个目标组合。

执行测试可能导致设备重启、kernel panic、看门狗挂死、短暂卡死或测试期数据丢失，执行者自行承担后果。若设备无响应，通常可长按“音量下 + 电源键”强制重启以恢复设备通行；如果机型组合键不同，以官方恢复方式为准。合法拥有者在精确目标固件上运行时，本仓库没有持久写入或提权驻留设计，通常不应造成永久性损伤，但这不是保证。

## 目标固件

| 项目 | 值 |
| --- | --- |
| 设备 | iQOO Z8 / vivo PD2314 |
| Build fingerprint | `vivo/PD2314/PD2314:15/AP3A.240905.015.A2/compiler260617110852:user/release-keys` |
| Kernel release | `5.10.233-android12-9-g44ec642832da-dirty` |
| 构建产物 | `exploit/build/PD2314-AP3A.240905.015.A2/bin/probe.so` |

## 构建与哈希核对

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35
sha256sum build/PD2314-AP3A.240905.015.A2/bin/probe.so
```

如需同时构建两个辅助测试程序：

```bash
make PROJECT=PD2314-AP3A.240905.015.A2 API=35
```

推送后建议在设备侧复核：

```bash
adb push build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell sha256sum /data/local/tmp/probe.so
```

## 日志优先的运行方式

探测结论依赖日志相关性。建议一个终端抓 stdout，另一个终端提前抓 logcat：

```bash
adb logcat -b all -v threadtime > logcat-live.txt
```

运行探测：

```bash
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id' 2>&1 | tee probe-console.txt
```

如果发生重启或卡死，恢复后补采：

```bash
adb shell getprop ro.boot.bootreason
adb shell dmesg > dmesg-after.txt
```

`pr_*` 日志默认写到 stdout，supervisor 会设置 unbuffered stdout；logcat 主要用来记录系统侧并发信息和崩溃线索，不一定包含完整 probe stdout。终端中可能出现 ANSI 颜色控制字符，归档时保留原始文本即可。

## 技术观测面

| 阶段 | 关键日志 | 说明 | 不能单独证明 |
| --- | --- | --- | --- |
| 目标门禁 | `probe profile applied`、`probe context`、`probe build` | 确认 worker 在目标 build 上进入 3.1 profile | 漏洞存在 |
| perf KASLR | `perf probe ring`、`candidate page`、`slide-kaslr-perf-ok` | 从 kernel IP sample 反推 live text base | 后续内存落点成功 |
| mm/slab | `prepare_kernel_page geom`、`kernelsnitch collision-scan leaked mm` | 通过 mm 碰撞拿到 order-3 slab 邻近地址 | payload 已命中目标对象 |
| payload | `tcp payload geometry`、`final payload invariant ok` | 验证用户态 payload 两个 32 KiB chunk 的内部字段 | kernel 中已经按该布局落位 |
| spray | `af_unix order3 staged pairs`、`spray sent=4096` | 记录 AF_UNIX/sk_buff 喷洒数量和首个失败点 | fops 路由已经发生 |
| futex/rt_mutex | `FUTEX_CMP_REQUEUE_PI`、`slide ip side effect sched_ok=1` | 记录 PI requeue 与 consumer 侧 `sched_setattr` 副作用 | 写入目标槽位成功 |
| restore | `misc.fops restore ... ret=8 errno=0` | 用 configfs write window 写回原始 `ashmem_fops` | 之前所有阶段均可靠 |
| 汇总 | `route-summary ... kaslr=1 ... route_done=1` | worker 到达 3.1 收尾 | `route_done=1` 不是独立漏洞证明 |

当前最强证据不是某一行日志，而是同一次运行内的相关链：目标门禁通过、perf base 被接受、KernelSnitch 找到碰撞并泄露 `mm`、spray 数量完整、futex/sched 副作用出现、restore 返回 `ret=8`、最后打印 `route-summary`。如果其中任一段缺失，只能说明该段之前的观测成立。

## 运行参数

| 环境变量 | 默认值 | 用途 |
| --- | ---: | --- |
| `PERF_PROBE_MSEC` | `200` | perf 采样窗口；过短会降低候选质量，异常值回退默认 |
| `PAGE_PREP_SLABS` | `16` | mm/slab 预铺规模；第二次 supervisor retry 起自动调到 `32` |
| `PROCESS_VM_CONSUMER_MAX_CALLS` | `1` | consumer 每轮 `sched_setattr` 调用次数 |
| `SLIDE_IP_ROUTE_ATTEMPTS` | `1` | IP multicast side-effect route 尝试轮数 |
| `SLIDE_IP_ROUTE_ARM_SEQ` | `1` | 从第几轮开始放行 consumer |
| `SLIDE_IP_MAIN_CONSUMER_DELAY_USEC` | `0` | consumer 调用前延迟 |
| `SLIDE_IP_POST_SETSOCKOPT_HOLD` | `20000` | 未放行 consumer 时的 yield 保持窗口 |

## 深入文档

- [01-observation](docs/zh-CN/01-observation.md)：日志采集、字段解释、证据阶梯和失败定位。
- [02-information-leak](docs/zh-CN/02-information-leak.md)：perf ring 解析、KASLR base 候选选择和日志边界。
- [03-stack-write](docs/zh-CN/03-stack-write.md)：mm/slab、payload、futex/rt_mutex、IP multicast side effect 与 restore 闭环。
