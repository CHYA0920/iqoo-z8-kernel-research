# 04 — PI 优先级传播：后续消费者

[English](../04-fire-walk.md) · [项目首页](../../README.zh-CN.md)

> 历史文件名仅为保持链接兼容而保留。本文解释消费损坏状态的合法调度机制，不描述触发序列。

## 结论

优先级链 walk 不是第二个根漏洞；它是让嵌套锁优先级继承保持正确的正常机制。[03](03-stack-write.md) 中的缺陷之所以危险，是因为后续 walk 会把 `task->pi_blocked_on` 和 cached waiter 树当成权威内核状态。

## 为什么必须传播

设高优先级任务 H 等待低优先级任务 L 持有的锁。PI 临时提升 L，使其尽快运行并释放锁。若 L 又阻塞在 M 持有的另一把锁上，只提升 L 没用；捐赠必须继续传播给 M。这就是 `rt_mutex_adjust_prio_chain()` 的设计目的。

```mermaid
flowchart LR
    H["H 等待"] --> L["L 持锁<br/>接受捐赠"]
    L --> M["L 又等待 M<br/>捐赠继续传播"]
```

加锁/解锁、超时、信号清理、owner 变化以及任务优先级/策略变化，都可能要求重新计算。这些是合法事件，并非漏洞行为。

## 所给镜像的任务 PI 模型

| 偏移 | 字段含义 | 消费作用 |
| --- | --- | --- |
| `task+0x84` | 有效优先级 | 比较/更新捐赠顺序 |
| `task+0x360` | deadline 键 | deadline 实体同优先级比较 |
| `task+0x86c` | `pi_lock` | 保护任务 PI 状态 |
| `task+0x880` | `pi_waiters.rb_root.rb_node` | 空树/根检查 |
| `task+0x888` | `pi_waiters.rb_leftmost` | 缓存最高优先级 waiter |
| `task+0x890` | `pi_top_task` | 调度器当前选择的 top donor |
| `task+0x898` | `pi_blocked_on` | 沿下一把锁继续传播 |

解释这些树所需的结构布局见 [06](06-static-analysis.md)。

## 指令级消费

每个链步入口先读取当前任务阻塞的 waiter：

```asm
rt_mutex_adjust_prio_chain (rel 0x225C68)
+0x00d8  ldr  x28, [x19, #0x898]  // task->pi_blocked_on
```

随后检查 owner 的 cached PI waiter 树，并把缓存的最左 RB 节点还原为外层 waiter。本镜像减法常量明确为 `0x18`，因为 `pi_tree_entry` 位于 waiter `+0x18`：

```asm
rt_mutex_adjust_prio_chain
+0x0104  add  x8, x19, #0x880
+0x0108  ldar x8, [x8]             // cached tree root
+0x010c  cbz  x8, ...
+0x0110  ldr  x8, [x19, #0x888]   // rb_leftmost
+0x0114  sub  x8, x8, #0x18       // 外层 waiter
```

完成树维护后，函数取 top waiter 的任务并重新计算有效调度优先级：

```asm
rt_mutex_adjust_prio_chain
+0x115c  add  x8, x19, #0x880
+0x1160  ldar x8, [x8]
+0x1164  cbz  x8, ...              // 空树选择 NULL donor
+0x1168  ldr  x8, [x19, #0x888]
+0x116c  ldr  x1, [x8, #0x18]     // pi_node + 0x18 == waiter->task
+0x1170  mov  x0, x19
+0x1174  bl   rt_mutex_setprio
```

空树不会跳过调用；它以 NULL donor 调用，使 owner 向 normal priority 恢复。

## 陈旧状态如何被再次消费

在错误清理下，任务可能已经从代理等待路径返回，但 `pi_blocked_on` 仍指向已完成内核栈帧中的 waiter。以后任一次优先级重算都会把它当成“仍在阻塞”读取。walk 随后可能：

- 把复用后的字节解释成 waiter 字段；
- 跟随从未真实存在的锁或 owner 关系；
- 用不一致成员关系调整 RB 树；
- 把非预期 donor task 交给 `rt_mutex_setprio()`。

具体后果依赖内存复用和线程交错。因此修复应消除陈旧状态生产者，而不是给每个消费者增加零散指针范围检查。

## `rt_mutex_setprio()` 与 CFI

本镜像中，有效优先级选择 deadline（`w25 < 0`）、实时（`0..99`）或 fair（`>99`）调度类，并把三个静态 `sched_class` 对象之一写入 `task+0x98`。CFI 校验后续间接回调目标，却不能证明 task、waiter、RB 节点或调度类指针来自仍存活对象。错误结构指针甚至可能在到达 CFI 前先异常。

## 修复为什么有效

`remove_waiter()` 在真实 waiter 任务的 `pi_lock` 下清空真实任务的 `pi_blocked_on` 后，后续 walk 就看不到需要继续跟随的阻塞边。这样恢复了抽象边界：传播机制可以继续信任内核自有 PI 状态，无需在每次读取时做昂贵生命周期验证。

调试构建加入不变量断言仍有价值，但量产根修复应位于制造不变量违反的清理点。
