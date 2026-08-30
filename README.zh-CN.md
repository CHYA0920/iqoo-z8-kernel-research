# iQOO Z8 kernel refclone probe

这是针对 iQOO Z8 / vivo PD2314 指定固件的内核漏洞存在性探测仓库。当前包的边界停在 3.1：完成 KASLR 解析、mm/slab 定位、ashmem fops 路由、rt_mutex/futex 触发、ashmem fops 还原和 `route-summary` 输出。

仓库里已经删掉 3.1 之后的旧阶段残留；当前内容只围绕探测闭环展开。

## 目标固件

| 项目 | 值 |
| --- | --- |
| 设备 | iQOO Z8 / vivo PD2314 |
| Build fingerprint | `vivo/PD2314/PD2314:15/AP3A.240905.015.A2/compiler260617110852:user/release-keys` |
| Kernel release | `5.10.233-android12-9-g44ec642832da-dirty` |
| 编译产物 | `exploit/build/PD2314-AP3A.240905.015.A2/bin/probe.so` |

`refclone.c` 的 worker 会先检查 fingerprint 和 kernel release；不匹配时直接退出，不继续探测。

## 构建

```bash
cd exploit
make probe PROJECT=PD2314-AP3A.240905.015.A2 API=35
```

如果需要同时构建两个辅助测试程序：

```bash
make PROJECT=PD2314-AP3A.240905.015.A2 API=35
```

## 运行

```bash
adb push exploit/build/PD2314-AP3A.240905.015.A2/bin/probe.so /data/local/tmp/probe.so
adb shell 'Z_REFCLONE=1 LD_PRELOAD=/data/local/tmp/probe.so /system/bin/toybox id'
```

运行入口仍然利用 `LD_PRELOAD` constructor 拉起 worker；产物名、入口名和日志已经改成 probe 语义。

## 运行链路

1. `preload.c` constructor 进入 `run_probe()`。
2. `main.c` 检查 `Z_REFCLONE=1`，进入 `refclone_load()`。
3. supervisor fork 三次以内的 worker，worker 用 `/system/bin/toybox id` 承载一次探测。
4. worker 检查目标固件身份，设置当前链路真实使用的三个默认参数。
5. `rc_perf_leak_text_base()` 通过 perf IP sample 还原 live KASLR base。
6. `rc_prepare_fops_page()` 用 KernelSnitch/mm 回收窗口定位 order-3 slab，并喷入两份 32 KiB payload。
7. `rc_run_main_route_threads()` 组织 waiter/owner/consumer 三线程，用 futex PI + IP multicast side effect 触发 fops 路由。
8. worker 用 configfs write window 把 `dashmem_misc.fops` 还原到原始 `ashmem_fops`。
9. 打印 `route-summary`，随后 toybox `id` 输出当前身份。

## 可调参数

| 环境变量 | 默认值 | 用途 |
| --- | ---: | --- |
| `PERF_PROBE_MSEC` | `200` | perf 采样窗口，影响 KASLR base 候选质量 |
| `PAGE_PREP_SLABS` | `16` | mm/slab 预铺规模；第二次 supervisor retry 起自动调到 `32` |
| `PROCESS_VM_CONSUMER_MAX_CALLS` | `1` | consumer 线程每轮 `sched_setattr` 调用数 |
| `SLIDE_IP_ROUTE_ATTEMPTS` | `1` | IP multicast side-effect route 尝试轮数 |
| `SLIDE_IP_ROUTE_ARM_SEQ` | `1` | 从第几轮开始放行 consumer |
| `SLIDE_IP_MAIN_CONSUMER_DELAY_USEC` | `0` | consumer 调用前延迟 |
| `SLIDE_IP_POST_SETSOCKOPT_HOLD` | `20000` | 未放行 consumer 时的 yield 保持窗口 |

## 文档

- [01-observation](docs/zh-CN/01-observation.md)：运行顺序、日志字段和 3.1 终止边界。
- [02-information-leak](docs/zh-CN/02-information-leak.md)：perf KASLR probe 的样本解析和候选选择。
- [03-stack-write](docs/zh-CN/03-stack-write.md)：mm/slab 回收、payload 布局、futex/rt_mutex route 和还原闭环。

English mirror: [README.md](README.md).
