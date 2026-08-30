# 06 — 指令级静态分析

[English](../06-static-analysis.md) · [项目首页](../../README.zh-CN.md)

## 样本与地址约定

目标为 AArch64 ELF，`5.10.233-android12-9`；链接基址 `0xffffffc008000000`。“rel”表示虚拟地址减链接基址。镜像无 DWARF/BTF，故布局由精确机器码中的生产者、消费者和栈对象边界恢复。结论只适用于根 README 所列 SHA-256 的 ELF。

## 1. `struct rt_mutex_waiter`

布局 A 与该镜像一致；布局 B 与指令不相容。

| 偏移 | 大小 | 字段 |
| --- | ---: | --- |
| `+0x00` | `0x18` | `rb_node tree_entry` |
| `+0x18` | `0x18` | `rb_node pi_tree_entry` |
| `+0x30` | `0x08` | `task_struct *task` |
| `+0x38` | `0x08` | `rt_mutex_base *lock` |
| `+0x40` | `0x04` | `int prio` |
| `+0x44` | `0x04` | 对齐填充 |
| `+0x48` | `0x08` | `u64 deadline` |
| `+0x50` | — | 结构末端 / `sizeof` |

直接初始化与赋值证据：

```asm
rt_mutex_init_waiter (rel 0x227AA0)
+0x0004  add  x8, x0, #0x18
+0x0008  str  x0, [x0]
+0x000c  str  x8, [x0, #0x18]
+0x0010  str  xzr, [x0, #0x30]

task_blocks_on_rt_mutex (rel 0x224E6C)
+0x0088  stp  x20, x19, [x22, #0x30]  // task, lock
+0x008c  ldr  w8, [x20, #0x84]
+0x0094  str  w8, [x22, #0x40]        // prio
+0x0098  ldr  x8, [x20, #0x360]
+0x009c  str  x8, [x22, #0x48]        // deadline
```

`rt_mutex_slowlock` 把 waiter 放在 `sp+0x08`，初始化至 `sp+0x57`，相邻栈保护值位于 `sp+0x58`，独立限定对象大小为 `0x50`。

要求的 leftmost 回推常量明确为 **`0x18`**：

```asm
rt_mutex_adjust_prio_chain (rel 0x225C68)
+0x0110  ldr  x8, [x19, #0x888]
+0x0114  sub  x8, x8, #0x18
```

## 2. PI 链的 task 字段读取顺序

一次循环按下表直接消费任务字段；后段读取依赖树与 top-waiter 是否变化：

| 顺序 | 函数偏移 | 指令 | 含义 |
| ---: | --- | --- | --- |
| 1 | `+0x00d8` | `ldr x28,[x19,#0x898]` | `pi_blocked_on` |
| 2 | `+0x0104/+0x0108` | `add ...,#0x880; ldar ...` | PI cached tree 根/空检查 |
| 3 | `+0x0110/+0x0114` | `ldr ...,#0x888; sub ...,#0x18` | leftmost 转 waiter |
| 4 | `+0x0134` | `ldr w8,[x19,#0x84]` | 有效优先级 |
| 5 | `+0x0150` | `ldr x8,[x19,#0x360]` | 条件性 deadline 键 |
| 6 | `+0x085c...` | 访问 `#0x86c` | owner 的 `pi_lock` |
| 7 | `+0x0a90/+0x1070` | `ldr ...,[x19,#0x888]` | 删除时的旧 cached leftmost |
| 8 | `+0x0afc/+0x10c0` | `ldr ...,[x19,#0x880]` | RB 插入根 |
| 9 | `+0x115c/+0x1160` | 再检查根 | 选择 donor 或 NULL |
| 10 | `+0x1168/+0x116c` | leftmost 后 `[node+0x18]` | top waiter 的 task |
| 11 | `+0x1174` | `bl rt_mutex_setprio` | 重算 owner 优先级 |
| 12 | `+0x11b8` | `ldr ...,[x19,#0x898]` | 是否继续链 |

`task+0x880` 是根指针，`task+0x888` 是 `rb_leftmost`。根为 NULL 会选择 NULL 第二参数，但不会仅因此跳过 `rt_mutex_setprio()`。当被重排 waiter 既不是旧 top 也不是新 top 时，调用才由 `+0x0a68..+0x0a78` 跳过。

## 3. `rt_mutex_setprio()` 决策树

存在 donor 时，候选优先级为 `min(p->normal_prio, pi_task->prio)`。快速返回要求 `p->pi_top_task == pi_task`，且候选非负并等于当前 `p->prio`。

```asm
rt_mutex_setprio (rel 0x1D4A70)
+0x0034  ldr  w25, [x20, #0x8c]  // normal_prio
+0x003c  ldr  w8, [x21, #0x84]   // donor prio（非 NULL 时）
+0x0048  ldr  x8, [x20, #0x890]  // pi_top_task
+0x030c  tbnz w25, #31, ...       // 负数不能走该快速返回
+0x0310  ldr  w8, [x20, #0x84]
+0x0318  b.eq ...                  // 未变化则返回
```

调度类选择与任务写入：

| 候选值 | 写入 `task+0x98` 的类 | 其他直接数据 |
| --- | --- | --- |
| `< 0` | `dl_sched_class` | 按分支读取 donor/task 的 `+0x84`、`+0x360`、`+0x36b`、`+0x400` |
| `0..99` | `rt_sched_class` | 离开 DL 时恢复自身 deadline 实体 |
| `>99` | `fair_sched_class` | 离开 RT 时清 timeout 状态 |

```asm
rt_mutex_setprio
+0x02f0  tbnz w25, #0x1f, +0x0374
+0x03e4  str  x8, [x20, #0x98]   // sched_class
+0x03e8  str  w25, [x20, #0x84]  // effective prio
```

CFI 校验的是加载后的 `sched_class` 间接回调，不校验 task 或 class 指针的生命周期/可读性。本函数不读取 `task->state`（`+0x30`）；把 state 设为零不能使部分初始化 task 安全。

## 4. `rb_erase()` 双子节点

左右子均存在时，successor `S` 是右子树最小值：若 `R->left == NULL` 则为 `R`，否则沿 left 链到末端。

```asm
rb_erase (rel 0xB25CC4)
+0x0004  ldp  x8, x9, [x0, #8]    // R, L
+0x0010  ldr  x11, [x8, #0x10]   // R->left
+0x0014  cbz  x11, +0x00b4        // 直接 successor S=R
+0x001c  mov  x10, x9             // 前一父节点 P
+0x0020  mov  x9, x11             // S
+0x0024  ldr  x11, [x11, #0x10]
+0x0028  cbnz x11, +0x001c        // 继续向左
```

深层 successor 的写集合依次重接 `P->left=S->right`、`S->right=R`、`R` 父元数据、`S->left=L`、`L` 父元数据、原父/根 child 槽、可选子节点 `C` 的父/颜色，最后让 `S` 继承被删节点 parent/color。这些是正常 RB 替换写，不构成独立漏洞证据。

只有替代子节点为 NULL 且 successor 原颜色为黑色时需要再平衡。本镜像从 `rb_erase+0x168` 进入内联 fix-up；双子分支不调用另有符号的 `__rb_erase_color`。

## 5. `try_to_wake_up()` 的 `state == 0`

不存在“解引用 task 前退出”。非自唤醒先定位/取得 `task+0x86c` 的 `p->pi_lock`，再读 `+0x30` 的 `p->state`。最早失败判决是 `+0x00b4`，最终返回在 `+0x062c`。

```asm
try_to_wake_up (rel 0x1C8DCC)
+0x0058  add   x22, x21, #0x86c
+0x00a8  ldr   x8, [x21, #0x30]
+0x00b0  tst   x8, x9
+0x00b4  b.eq  +0x05bc
+0x05c0  stlrb w8, [x22]
+0x062c  ret
```

自唤醒分支不取得 task `pi_lock`，但仍会先加载 `task+0x30`，才判断 state zero 不可唤醒。

## 边界

本分析证明机器码布局与可能的直接消费，不单独证明运行时可利用性、竞争可靠性或其他厂商构建适用性。缺失的源码级/内联边界按不确定处理，不用香草内核假设填充。
