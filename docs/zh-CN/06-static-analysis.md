# 内核静态分析笔记

支撑已公开阶段的静态发现选编。这些是关于内核二进制(5.10.233,MTK 厂商树,本设备构建)的事实,每条都按方法论要求验证:反汇编是数值权威,反编译只用于定位代码,每个承重偏移都以内核镜像文件直读二次确认。

## rt_mutex_waiter 布局(5.10.233,MTK 厂商树)

futex PI 机制走过的 waiter 结构。以 requeue 路径反汇编核验的偏移:

```text
struct rt_mutex_waiter {
  +0x00  rb_node  tree_entry        // waiter 树成员
  +0x18  rb_node  pi_tree_entry     // PI 传播树成员
  +0x30  task_struct *task          // 所属任务
  +0x38  rt_mutex   *lock           // 该 waiter 阻塞的锁
  +0x40  int        prio            // 有效优先级
  +0x48  u64        deadline        // deadline 调度键
  ...
};
```

注意:

- prio 在 +0x40 为普通 int 经专项确认(其他 5.10 厂商树有差异)。
- 双 rb_node 成员(普通树与 PI 树)是 walk 成为树遍历而非链表走的原因——经 [2.3] stamp 的几何必须对两个消费者都合法。

## compat setsockopt 拷贝(MCAST_JOIN_SOURCE_GROUP)

setsockopt(fd, IPPROTO_IPV6, MCAST_JOIN_SOURCE_GROUP, buf, len) 且 len = 260,把调用方的 group-source 过滤请求结构拷入内核待验证。对 [2.3] 承重的结构要点:

- 结构固定 260 字节——拷贝长度由调用方声明,内核在验证语义字段前恰好拷 len,故 260 字节写是研究者定尺寸的。
- compat 路径(32 位调用方,64 位内核)上,入口链经过 compat syscall 框架,栈深与原生路径不同;目的槽深度在给定内核构建与入口路径下固定。
- 原生(64 位)pselect/sendmmsg 深栈替代路径已检查,因栈几何原因在本内核不可行;32 位 compat socket 路径是目的几何稳定的那条。这是公开探针构建为 armeabi-v7a 的原因。

## futex 哈希桶定位

内核把 futex key(地址、大小)映射到哈希桶,方式是对页与偏移做乘法哈希。推演一条链落在哪个 waiter 树,需要在用户空间复现该哈希。exploit/src/kernelsnitch/futex_hash.h 就是那个移植:同常数、同移位算术、可离线运行。它让 harness 在制造桶碰撞(进而 waiter 堆积)之前预测碰撞。

## perf_event 采样面

本设备允许非特权 perf_event_open(perf_event_paranoid ≤ 1)。[1.1] 的采样侧用内核侧活动的时序直方图推导 text 基址;kernelsnitch.h 头文件记录了批次节奏循环(有界 waiter 批、批间互斥唤醒的遍历时间探针、遍历时间爆表提前退出),防止采样器失稳被测系统。

## CFI

本内核构建启用了控制流完整性:间接调用目标经跳转表桩。对已公开阶段这只作为环境事实存在(记录于 [01-observation.md](01-observation.md));无任何公开阶段依赖击破它。

## 静态分析方法规则(短版)

1. 反汇编是数值权威。反编译输出用于定位代码,永不用于数值。
2. 运行时命题依赖的每个偏移,以直读内核镜像文件确认(ELF 程序头 → 文件偏移 → 字节)。
3. 交叉引用核验用反汇编器的 xref 通道,不做即席文本扫描——手扫调用点有漏检要害的文档化历史。
4. 静态发现永不把研究节点推过 A2 级,直到运行时判据轮观测到它。(见 [05-methodology.md](05-methodology.md)。)
