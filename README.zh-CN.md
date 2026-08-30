# iQOO Z8 内核研究

[English documentation](README.md)

## 研究声明

本仓库公开的是在研究者**自有设备**(vivo iQOO Z8,PD2314,内核 5.10.233,Android 15)上完成的**防御性安全研究**。全部实验均在受控实验环境中、针对研究者本人持有的设备进行,目的是理解内核攻击面、改进检测与加固方案。

公开材料只覆盖整个研究计划中的**信息泄漏与内核交互阶段**。公开边界之后的全部内容——包括优先级继承(priority-inheritance)walk 之后的任何环节——均已刻意排除在本仓库之外。本仓库没有任何完整利用链,也不应被期待能拼出一条。

复现本仓库的观测结论需要:同型号实体设备、相同内核构建、受控测试环境。对不属于自己的设备使用相关技术,在绝大多数司法辖区属于违法行为。

## 公开边界

已公开:

- 0 → 3.1 全部研究节点的地图(见下),含每节点闭环条件、观测通道与实测轮次。
- 两个自包含的 syscall 路径探针(`mcast_test`、`sched_test`),用于验证本研究依赖的两个内核入口的可达性。
- 纯头文件的 KASLR 侧信道采样工具族(`kernelsnitch/`),即整个计划使用的基于 perf 的 text 基址披露实现。
- 研究文档:方法论、观测基础设施、各阶段背后的内核静态分析。

刻意不公开:3.1 之后的全部阶段。本仓库在"PI walk 执行并返回"这一里程碑处截断。无 walk 后产物、无提权材料、无持久化组件。

## 研究地图

整个计划以"最小闭环语义节点"树跟踪。一个节点被标为 **STABLE(已敲定)**,当且仅当:①闭环条件在实验前书面写定;②运行时判据连续 ≥3 轮通过;③复现路径文档化到换一个会话即可重跑。静态分析永远不能单独把节点标为 STABLE。

```text
[0] 基础设施与观测通道
 ├─[0.1] 设备 / adb / CI 构建部署流水线 ................. 已敲定
 ├─[0.2] TCP 微秒级探针通道 ............................. 已敲定
 └─[0.3] O_DSYNC 诊断落盘通道 ........................... 已敲定

[1] 信息泄漏域(纯读,系统健康)
 ├─[1.1] KASLR text 基址(perf_event 采样)............... 已敲定
 ├─[1.2] 内核符号锚族(差值式推导) ...................... 已敲定
 ├─[1.3] 物理页披露与受控 staging ....................... 已敲定
 └─[1.4] 喷射驻留(sk_buff 16/16 全回收)................ 已敲定

[2] 内核交互原语域
 ├─[2.1] futex PI chain-1 EDEADLK ........................ 已敲定
 ├─[2.2] chain-2 / chain-3 环闭合 ........................ 已敲定
 └─[2.3] 树 stamp 原语(32-bit compat setsockopt
          深栈写,260 字节,逐字节可验).................. 已敲定 ★

[3.1] fire → PI walk 执行并返回用户态
        (sched_setattr 触发,walk 消费已布置的
         几何并安全返回,ret2 = 0)...................... 已敲定
```

带每节点细节的完整版地图见 [docs/research-map.md](docs/research-map.md)。

## 研究价值

**1. 一套可运转的微秒级观测系统。** 在生产 Android 设备上做内核利用研究,通常会安静地死掉:内核 panic 后,shell 级诊断基本不可用(本设备的 pstore、console、kmsg 全部被权限封死)。本计划从结构上解决了这个问题。TCP 探针通道以微秒分辨率上报每阶段时间戳(连接、payload 就绪、触发),并在内核死亡后仍存活足够长时间以被抓取;O_DSYNC 直写日志通道保证 panic 前最后一行诊断在设备重启前已经落盘。连续十八轮以上双通道全活。后续每一个结论之所以可证伪,靠的都是这套基础设施。

**2. 三级证据纪律应用于内核研究。** 本计划每个结论都携带显式证据等级:A1(直接运行时观测,判据自证行在场)、A2(间接推断,永不单独支撑设计)、B(未探测)。结论从 B 升到 A1 的唯一路径是专用判据轮。这套纪律记录于 [docs/05-methodology.md](docs/05-methodology.md),在我们看来是整个计划最具复用性的产物:它是"你如何知道你的利用研究结论是真的"这一问题的可运转答案。

**3. 基于 perf_event 时序侧信道的 KASLR 披露,以投票机制加固。** text 基址从非特权 perf_event_open 采样恢复。一个关键工程发现:单锚推导在可测比例的启动上概率性出错(错误基址可以偏差兆字节级却仍然"看起来合理")。修复方案是多采样投票——加上投票后,验证批次中的披露通过率达到 100%。实现它的工具族公开在 `exploit/src/kernelsnitch/`。

**4. 通过 32-bit compat setsockopt 路径的逐字节精确 260 字节深栈写。** 本计划的核心内核交互成果。在本内核上,由 32 位任务发起的 `setsockopt(IPPROTO_IPV6, MCAST_JOIN_SOURCE_GROUP)` 会把 260 字节的调用方缓冲拷贝到调用任务内核栈深处的一个固定槽位。研究以逐字节回读验证(32 qword dump)证明:该拷贝落在可预测的栈偏移上,且可携带任意内容。即地图中的 [2.3]——原语域的明星资产。

**5. futex PI 链几何作为确定性的 walk 前状态机。** 研究构造了三锁 futex PI 链(EDEADLK 回滚路径),其中每个中间状态都可观测:chain-1 每轮返回 EDEADLK(errno 35),chain-2/3 以 WAITERS 位握手闭合环路。最终的 walk——由 sched_setattr 重定优先级触发——消费已布置的几何并干净返回(ret2 = 0),即本仓库的截断点 3.1。

## 内核静态分析背景

支撑已公开阶段的静态发现选编在 [docs/06-static-analysis.md](docs/06-static-analysis.md)。要点:

- **rt_waiter 几何(5.10.233,MTK 厂商树)。** 本内核的 `rt_mutex_waiter` 布局:tree_entry 在 +0x00,pi_tree_entry 在 +0x18,task 指针 +0x30,lock 指针 +0x38,prio +0x40,deadline +0x48。以 futex requeue 路径的反汇编核验,不以反编译为准。
- **compat setsockopt 拷贝路径。** MCAST_JOIN_SOURCE_GROUP 触发的 260 字节 group-source 过滤结构拷贝,以及为什么 compat(32 位调用方)路径才是本 64 位内核上栈位置问题的关键。
- **futex 哈希桶定位。** futex key 的地址哈希(`futex_hash.h` 把内核内哈希移植到用户态)——推演一条链落在哪个 hb 桶所必需。
- **方法规则。** 数值以反汇编为权威;反编译只用于定位有趣代码。所有关键偏移均在 ELF 文件中直接二次确认。

## 仓库结构

```text
exploit/
  Makefile                 构建两个探针(32 位 ARM,NDK r29)
  src/
    mcast_test.c           setsockopt(MCAST_JOIN_SOURCE_GROUP) 探针
    sched_test.c           sched_setattr 可达性探针
    kernelsnitch/          KASLR 侧信道采样工具族(头文件)
docs/
  research-map.md          完整节点地图(含闭环条件)
  01-observation.md        阶段 0:观测基础设施
  02-information-leak.md   阶段 1:KASLR / 符号锚 / 页披露
  03-stack-write.md        阶段 2:compat 深栈写原语
  04-fire-walk.md          阶段 3.1:PI walk 执行
  05-methodology.md        证据分级 / 判据先行 / 单变量轮
  06-static-analysis.md    内核静态分析笔记
.github/workflows/build.yml   CI:构建并上传两个探针
```

## 构建

需要 Android NDK(r29,含 32 位 ARM 工具链),或带 Android sysroot 的本机 clang:

```bash
cd exploit
make mcast-test sched-test
```

CI 在每次 push 到 main 后构建两个探针并上传产物(mcast-test-z8、sched-test-z8)。

两个探针均为静态 PIE armeabi-v7a 二进制,零依赖。各自只发起一次 syscall 并打印带定界符的结果块——它们是可达性探针,不是利用程序。

## 许可

为研究与防御教育目的公开。无担保。遵守所在辖区法律的责任由使用者自行承担。
