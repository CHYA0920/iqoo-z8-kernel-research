# 07 — KernelSU 内核模块授权

## 概述

提权成功后，设备已具备 root 权限（uid=0, SELinux Permissive）。但
KernelSU APP 无法直接使用：APP 内嵌的 `libksud.so` 在 fork 子进程时
继承了 Android APP 的 seccomp 过滤器，syscall 142 被 seccomp 阻断
（`SIGSYS, SYS_SECCOMP`），导致 ksud daemon 崩溃。

解决方案：在提权成功后、root daemon 仍然存活的时间窗口内，使用 root
权限加载 KernelSU 内核模块（`kernelsu.ko`）。内核模块加载后，KernelSU
的内核 hook 拦截 seccomp 检查并放行，APP 内的 ksud 即可正常运行。

## 前置条件

- 提权成功，root daemon 存活（`/data/local/tmp/temp_su.sock` 可连接）
- SELinux 已被设为 Permissive（提权器自动完成）
- 设备 KMI: `android12-5.10`（内核 `5.10.233-android12-9`）

## 文件准备

### kernelsu.ko

从 GitHub 下载，无需修补 vermagic：

```
来源: https://github.com/tiann/KernelSU/releases/download/v3.2.5/android12-5.10_kernelsu.ko
大小: 344,504 bytes
KMI:  android12-5.10
```

ksud 的 `insmod` 命令使用 kallsyms 访问加载模块，绕过 vermagic 检查。
无需匹配设备内核的 vermagic 字符串。

### ksud

从 KernelSU APP 的 APK 中提取 `libksud.so`，作为独立二进制使用：

```
APK 路径: /data/app/~~*/me.weishu.kernelsu-*/base.apk
提取路径: lib/arm64-v8a/libksud.so
提取后重命名为 ksud
大小: 4,892,712 bytes
```

提取方法（PC 端）：

```python
import zipfile
z = zipfile.ZipFile('kernelsu_base.apk')
z.extract('lib/arm64-v8a/libksud.so', '.')
# 重命名为 ksud，push 到设备
```

## 授权流程

### 步骤 1：推送文件

```bash
adb push kernelsu.ko /data/local/tmp/kernelsu.ko
adb push ksud /data/local/tmp/ksud
```

### 步骤 2：提权

```bash
adb shell "LD_PRELOAD=/data/local/tmp/preload_master.so \
  Z_REFCLONE=1 EXPLOIT_ARM=1 RC_ALLOW_DIRTY=1 \
  RC_INSTALL_PERSIST=1 /system/bin/toybox id"
```

提权成功标志：

```
[*] root cred patched uid=0/0 sid=1/1
[+] embedded su daemon ready pid=XXXX socket=/data/local/tmp/temp_su.sock
uid=0(root) gid=0(root) groups=0(root) context=u:r:kernel:s0
```

### 步骤 3：加载内核模块

在 root daemon 存活期间（同一 shell 会话内紧接着执行）：

```bash
adb shell "/data/local/tmp/su -c 'chmod 755 /data/local/tmp/ksud; \
  /data/local/tmp/ksud insmod /data/local/tmp/kernelsu.ko allow_shell=1'"
```

成功输出：

```
Loaded kernel module: /data/local/tmp/kernelsu.ko
```

### 步骤 4：验证

```bash
# 内核模块已加载
adb shell "cat /proc/modules | grep kernelsu"
# 输出: kernelsu 196608 1 - Live 0x0000000000000000 (O)

# su 命令可用，SELinux context 为 ksu
adb shell "su -c 'id'"
# 输出: uid=0(root) gid=0(root) groups=0(root) context=u:r:ksu:s0

# SELinux 恢复 Enforcing（KernelSU 内核层绕过）
adb shell "getenforce"
# 输出: Enforcing
```

### 步骤 5：启动 KernelSU APP

```bash
adb shell "am start -n me.weishu.kernelsu/.ui.MainActivity"
```

APP 检测到内核模块后自动进入工作状态，可授予其他 APP root 权限。

## 关键技术点

### 为什么 APP 内嵌的 ksud 会崩溃

KernelSU APP 通过 `forkDontCareAndExecKsud` fork 子进程执行
`libksud.so`。子进程继承了 APP 的 seccomp 过滤器。ksud 在执行过程中
调用了被 seccomp 阻断的系统调用（syscall 142），触发 SIGSYS 信号被杀死。

日志证据：

```
Fatal signal 31 (SIGSYS), code 1 (SYS_SECCOMP), syscall 142
  in tid XXXX (libksud.so)
Cause: seccomp prevented call to disallowed arm64 system call 142
```

### 为什么加载内核模块后 APP 就正常了

KernelSU 内核模块（`kernelsu.ko`）注册了内核 hook，拦截了 seccomp
检查路径。内核模块加载后，APP fork 出的 ksud 子进程的 seccomp 检查
被内核 hook 放行，不再触发 SIGSYS。

### 为什么用 ksud insmod 而不是 insmod

直接 `insmod` 会检查模块的 vermagic 字符串。设备内核版本为
`5.10.233-android12-9-g44ec642832da-dirty`，而 ko 的 vermagic 为
`5.10.252-dirty SMP preempt mod_unload modversions aarch64`。两者不
匹配，`insmod` 返回 `Exec format error`。

`ksud insmod` 使用 kallsyms 符号访问来加载模块，绕过了 vermagic
检查。

### 为什么不修补 vermagic

设备内核的 vermagic 字符串 `5.10.233-android12-9-g44ec642832da-dirty`
比 ko 原始的 `5.10.252-dirty` 长 26 字节，超出 ko 中 vermagic 字段的
固定空间，无法原地替换。使用 `ksud insmod` 绕过此限制更为可靠。

### 时序约束

root daemon 是临时的——提权器进程退出后 daemon 随之终止。因此步骤 3
必须在步骤 2 成功后的同一 shell 会话中紧接着执行。如果 daemon 已死
（`su: connect daemon: Connection refused`），需要重新提权。

## 持久化

当前流程为一次性授权，重启后内核模块不会自动加载。持久化需要：

1. 将 kernelsu.ko 和 ksud 放到持久化路径（如 `/data/adb/`）
2. 通过 boot service 脚本在开机时自动加载

提权器的 `RC_INSTALL_PERSIST=1` 参数会安装 boot service 到
`/data/adb/service.d/10-neo11-su.sh`，但该脚本不包含 kernelsu.ko 的
自动加载逻辑。持久化的完整实现需要在 boot service 脚本中添加
`ksud insmod` 调用。

## 设备信息

- 型号: vivo PD2314 (iQOO Z8)
- SoC: MediaTek Dimensity 8200 (mt6896)
- 内核: 5.10.233-android12-9-g44ec642832da-dirty
- Android: 15 (AP3A.240905.015.A2)
- KMI: android12-5.10
- KernelSU: v3.2.5
- 提权器: preload_master.so (rework-refclone-ip 分支, r16n)
