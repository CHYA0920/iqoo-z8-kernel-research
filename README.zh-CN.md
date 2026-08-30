# iQOO Z8 内核研究——防御性技术文档

[English](README.md)

> [!CAUTION]
> **不要运行来自 Issue、评论区、聊天群、Fork 或非官方镜像的共享库、预加载对象、二进制、脚本和压缩包。** 通过 `LD_PRELOAD` 装入的 `.so` 会在启动进程内部执行；在所谓“内核测试”开始前，它就可能读取凭据、修改文件、联网或破坏已连接设备。除非源代码、构建方法、审查历史和密码学摘要均已独立核验，否则一律按恶意文件处理。实验应使用隔离、无个人账户、无令牌、无 SIM 卡、无重要数据的专用样机。

## 项目目的与公开边界

本仓库是纯文档、纯防御性的内核研究，研究对象为研究者自有的 vivo iQOO Z8（PD2314）样本，主题包括调度器优先级继承（PI）、futex requeue-PI 清理以及相关内核不变量。文档说明设计原意、二进制证据、失效语义、修复方法和安全验证方案。

本项目**不鼓励**未授权访问、提权、持久化或漏洞载荷部署。文档刻意不提供可运行利用链、堆回收配方、面向利用的伪对象模板、权限修改步骤或武器化参数。只有在审核所给镜像和复核修复所必需时，才保留内核字段偏移与指令位置。

目标为 AArch64 厂商内核，版本标识 `5.10.233-android12-9`。Android 用户空间/OTA 标签可能不同；厂商常做补丁回移，因此仅凭版本字符串既不能判定有漏洞，也不能判定已修复。

## 核心结论

相关失效是 **CVE-2026-43499** 所描述的 rtmutex 代理加锁回滚路径“任务身份错配”。受影响实现中的 `remove_waiter()` 按 `current` 清理，但 waiter 实际属于 `waiter->task`；futex requeue-PI 场景下两者可以不同。真实 waiter 任务因而可能保留指向已结束生命周期栈对象的 `pi_blocked_on`。随后完全合法的 PI 优先级传播会信任该内核内部指针并再次消费陈旧状态。

语义修复是：任务 `pi_lock`、`pi_blocked_on` 清零以及传给优先级链调整的 top-task 参数，均统一使用 `waiter->task`。应回移上游或厂商审核过的完整补丁，并同时包含 **CVE-2026-53166** 对自死锁/未初始化 waiter 的后续保护；不能只手工摘取一个差异片段而不做回归覆盖。

## 文档导航

| 简体中文 | English | 主题 |
| --- | --- | --- |
| [研究地图](docs/zh-CN/research-map.md) | [Research map](docs/research-map.md) | 信任边界、不变量失效、修复关卡 |
| [01 — 观测](docs/zh-CN/01-observation.md) | [01 — Observation](docs/01-observation.md) | 安全实验与证据采集 |
| [02 — 信息暴露](docs/zh-CN/02-information-leak.md) | [02 — Information exposure](docs/02-information-leak.md) | KASLR/perf/debug 暴露面与加固 |
| [03 — 根因](docs/zh-CN/03-stack-write.md) | [03 — Root cause](docs/03-stack-write.md) | futex PI 代理加锁生命周期与修复 |
| [04 — PI 传播](docs/zh-CN/04-fire-walk.md) | [04 — PI propagation](docs/04-fire-walk.md) | 后续调度活动为何会重消费陈旧状态 |
| [05 — 方法论](docs/zh-CN/05-methodology.md) | [05 — Methodology](docs/05-methodology.md) | 证据分级与安全验证 |
| [06 — 静态分析](docs/zh-CN/06-static-analysis.md) | [06 — Static analysis](docs/06-static-analysis.md) | AArch64 指令级结论 |

中英文文件使用相同章节结构并互相链接。历史文件名 `03-stack-write.md` 和 `04-fire-walk.md` 仅为保持仓库链接兼容而保留；内容已经改写为防御性的根因与传播分析。

## 修复优先级

1. **修正生命周期不变量：**厂商 `remove_waiter()` 路径必须使用 `waiter->task`，持有该任务的 `pi_lock`，并将同一任务身份传给链调整。
2. **纳入后续保护：**确认 requeue-PI 自死锁路径不会用未初始化或 NULL 的 `waiter->task` 调用清理逻辑（CVE-2026-53166）。
3. **压缩暴露面：**按产品需求限制生产环境 perf、debugfs/tracefs、厂商调试节点及包含内核地址的日志。
4. **按不变量验收：**任何回滚、超时或信号退出后，返回用户态的任务都不得仍保留 `pi_blocked_on`；RB 树成员关系与任务引用必须受正确的锁保护。

## 参与者产物规则

- 欢迎文档类 Pull Request。不要附带预编译 `.so`、APK、ELF、内核模块、加密压缩包或设备镜像。
- 不要要求审阅者运行不透明产物。若文档分支之外确需测试产物，应同时给出源码、精确构建方法、编译器身份及可复现摘要。
- 公开日志前删除设备标识、账户数据、令牌、boot ID 和运行时绝对内核地址。
- 对可能显著增加现实利用能力的问题，应先走设备/内核厂商的私密披露流程。

## 证据样本身份

本文档的静态结论来自所给 AArch64 ELF 与 kallsyms。记录的 SHA-256：

- ELF：`77cfbe299179360f54b5cb41f119766d3642a07208ce09d87b238072fbf19a52`
- ELF 容器压缩包：`149ba95260bf86806f98771bc86403ce2317d3b359b60b29d7865ebda3756dd0`
- kallsyms：`539dddd5bc02d86460b1fa9d6bc6808b0610395462fa271a8be261dd2ec518a2`

## 参考资料

- [Linux 上游修复 3bfdc63936dd：`rtmutex: Use waiter::task instead of current in remove_waiter()`](https://git.kernel.org/linus/3bfdc63936dd4773109b7b8c280c0f3b5ae7d349)
- [Debian：CVE-2026-43499](https://security-tracker.debian.org/tracker/CVE-2026-43499)
- [Red Hat：后续回归 CVE-2026-53166](https://access.redhat.com/security/cve/cve-2026-53166)

## 免责声明

本文档不提供任何担保。授权边界、实验安全、披露协调、出口规则与所在地法律合规均由使用者自行负责。
