# 03 — 根因：futex PI 代理 waiter 生命周期

[English](../03-stack-write.md) · [项目首页](../../README.zh-CN.md)

> 历史文件名仅为保持链接兼容而保留。本文讨论生命周期缺陷与修复，不描述写原语。

## 结论

必须修复的原因是 `remove_waiter()` 同时服务两种语义环境：

1. 普通 rtmutex 慢速加锁清理，执行任务通常就是 waiter 所属任务；
2. futex requeue-PI 代理加锁回滚，一个任务可能代替另一个任务操作 waiter。

旧实现使用 `current`，只是在第一种环境中碰巧成立；到第二种环境就发生任务身份错配。权威身份应当是 `waiter->task`。

## 为什么需要代理加锁

PI futex 把用户态 futex 字与内核 rtmutex 结合。requeue-PI 可以把等待任务从非 PI 等待队列迁移到 PI futex，而不先唤醒它制造无控制竞争。因此内核执行一次**代理**操作：当前 requeue 执行者代替真实等待任务修改锁状态。

三种角色必须分清：

| 角色 | 含义 |
| --- | --- |
| `current` | 正在执行 requeue/回滚代码的任务 |
| `waiter->task` | rtmutex waiter 所代表的任务 |
| lock owner | 当前因持锁而接受优先级捐赠的任务 |

三者可以分别是不同任务。普通路径和代理路径复用的 helper 不能从“谁正在执行”推断“waiter 属于谁”。

## 正常 PI 状态

每个任务具有：

- `pi_blocked_on`：指向该任务当前阻塞的存活 waiter；
- `pi_waiters`：该任务作为锁 owner 时收到优先级捐赠的 cached RB 树；
- `pi_lock`：串行化该任务 PI 状态的修改。

关键生命周期规则是：

```text
task->pi_blocked_on != NULL
    蕴含 waiter 仍存活、属于 task，且 task 确实正在阻塞。
```

因回滚、超时、信号或成功获得锁而删除 waiter 时，必须在 waiter 存储失效前清空该指针。

## 失效的清理

受影响代理回滚中 `waiter->task != current`。按 `current` 清理会造成上游修复说明中的三类相关错误：

1. 删除 waiter 树节点时没有持有真实 waiter 任务的 `pi_lock`；
2. 真实 waiter 任务的 `pi_blocked_on` 未清零，栈对象生命周期结束后留下悬空指针；
3. `rt_mutex_adjust_prio_chain()` 收到错误的 top-task 身份。

所以根因不是“调度器写得太多”，而是清理阶段制造的**所有权与生命周期违反**。后续调度代码只是在信任本应始终成立的内部状态。

## 语义修复

必须按具体厂商分支/API 适配，但三个任务相关操作要统一使用同一个 `waiter_task`：

```c
/* 仅表示语义；实际应采用上游/厂商审核过的精确回移。 */
struct task_struct *waiter_task = waiter->task;

raw_spin_lock(&waiter_task->pi_lock);
rt_mutex_dequeue(lock, waiter);
waiter_task->pi_blocked_on = NULL;
raw_spin_unlock(&waiter_task->pi_lock);

rt_mutex_adjust_prio_chain(/* 分支相关参数 */,
                           waiter_task);
```

不要直接复制这段示意签名；stable 与 Android 厂商分支在 `rt_mutex_base` 类型、参数顺序、锁 helper 和 guard 风格上可能不同，应审查真实上游提交。

## 后续回归：部分初始化

清理改为解引用 `waiter->task` 后，另一个不变量变得必须显式保证：到达 `remove_waiter()` 的所有路径都已经初始化该字段。非 top requeue waiter 的自死锁错误可能在赋值前提前返回。CVE-2026-53166 的后续修复用于阻止这种路径携 NULL waiter task 进入清理。

因此完整回移集必须包含两个概念：

- CVE-2026-43499：选择正确任务；
- CVE-2026-53166：在该任务尚不存在时，不进入正确任务清理。

## 源码审查判定表

| 厂商树观察 | 判定 |
| --- | --- |
| 共享 `remove_waiter()` 中出现 `current->pi_lock` / `current->pi_blocked_on` | 存在受影响语义模式 |
| 局部 `waiter_task = waiter->task`，并统一用于加锁、清零和链调整 | 主修复已存在 |
| 代理清理前检查 requeue 自死锁，或有等价的初始化证明 | 后续保护已存在 |
| 只有内核版本号 | 证据不足 |

## 验证矩阵

在带检测器的工程内核上覆盖普通加解锁、requeue 成功、回滚、超时、信号中断、owner 退出、top/non-top waiter 与自死锁。每个退出点都检查正确 `pi_lock`、`pi_blocked_on` 清理、任务引用平衡、RB 树缓存一致，并确认无 lockdep/KASAN/KCSAN 报告。

## 参考资料

- [上游提交 3bfdc63936dd](https://git.kernel.org/linus/3bfdc63936dd4773109b7b8c280c0f3b5ae7d349)
- [CVE-2026-43499 描述与修复版本跟踪](https://security-tracker.debian.org/tracker/CVE-2026-43499)
- [CVE-2026-53166 后续问题](https://access.redhat.com/security/cve/cve-2026-53166)
