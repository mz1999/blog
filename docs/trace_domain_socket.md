# 追踪 Java 进程创建的 domain socket

在使用 CRaC 创建 checkpoint 镜像时，需要 Java 应用能够恰当处理它持有的外部资源，例如打开的日志文件，监听的服务端口，对外创建的数据库连接池等。

对于 Java 应用打开的文件，可以通过定义[文件描述符策略](https://docs.azul.com/core/crac/fd-policies) 让 CRaC 自动处理。对于监听端口或连接池，一般建议应用实现 CRaC 的 [Resource](https://github.com/CRaC/org.crac/blob/master/src/main/java/org/crac/Resource.java) 接口，在 checkpoint 前关闭资源，在 restore 后重新打开。

但有一些资源，不是 Java 应用直接打开持有的，而是底层的 JVM 或依赖的 C 库打开的，这类资源要追踪定位是谁创建的比较困难。

## 初步排查过程
最近我就碰到了这样一个例子。在对 Java 应用创建 checkpoint 时，报如下异常：

```java
An exception during a checkpoint operation:
jdk.internal.crac.mirror.CheckpointException
        Suppressed: jdk.internal.crac.mirror.impl.CheckpointOpenSocketException: FD fd=6 type=socket path=socket:[2885875383],port=29295
                at java.base/jdk.internal.crac.mirror.Core.translateJVMExceptions(Core.java:116)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestore1(Core.java:189)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestore(Core.java:315)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestoreInternal(Core.java:328)
```
表明 Java 应用持有文件描述 (fd) 6，类型是 socket，inode 为 2885875383.

于是查看该 Java 进程持有的所有 socket 资源：

```bash
ll /proc/2444401/fd  | grep socket
...
lrwx------ 1 root root 64 Sep  3 11:19 6 -> socket:[2885875383]
```

果然 Java 进程持有的 fd 6，类型是 socket，inode 为 2885875383。进一步使用 `lsof` 命令查看这个 socket 资源的详细信息：

```bash
lsof -p 2444401 | grep 2885875383
exe     2444401 root    6u     unix 0x00000000b8935f1c       0t0 2885875383 type=STREAM
```

第五列为 `unix`，表示该文件描述符对应的资源是 `unix domain socket`。

想进一步调研这个 domain socket 的来源和作用，它的一端是 Java 进程持有，那么另一端是谁持有呢？

```bash
# -U ：筛选 Unix 域套接字（Unix Domain Sockets）。
# -a：逻辑 “与”（AND） 操作，将前面的条件（-p 和 -U）组合，表示必须同时满足
# +E：显示套接字的 端点信息（Endpoints）。
lsof -p 2444401 -U -a +E
COMMAND     PID USER   FD   TYPE             DEVICE SIZE/OFF       NODE NAME
exe     2444401 root    6u  unix 0x00000000b8935f1c      0t0 2885875383 type=STREAM
exe     2444401 root   57u  unix 0x00000000a4202b14      0t0 2885883519 type=STREAM
```

输出中并没有发现对端的进程信息。

一般情况下，加了 `+E` 参数，输出中会通过 `->INO=` 指示出对端进程信息，例如：

```bash
$ sudo lsof -p 1167 -U -a +E
...
container 1131 root    8u  unix 0xffff9d34cdf28c00      0t0 14525 /run/containerd/containerd.sock type=STREAM ->INO=21711 1167,dockerd,8u (CONNECTED)
...
dockerd   1167 root    8u  unix 0xffff9d34cdf29400      0t0 21711 type=STREAM ->INO=14525 1131,container,8u (CONNECTED)
...
```
可以看出，在 `containerd` 这一端，它使用文件描述符 8（标记为 8u，表示可读可写）来管理 domain socket，这个套接字被绑定到文件系统路径 `/run/containerd/containerd.sock` 上。行末的 `->INO=21711 1167,dockerd,8u (CONNECTED)` 表明，该套接字的对端是进程 ID 为 1167 的 `dockerd`。`dockerd` 同样使用文件描述符 8 来管理这个连接，且连接状态为已建立`（CONNECTED）`。

再查看 `dockerd` 进程这端的情况，它也使用文件描述符 8（8u）来维护这个通信通道。与 `containerd`不同，`dockerd` 端的套接字没有绑定任何文件系统路径，这是一个匿名套接字。同样，行末的 `->INO=14525 1131,container,8u (CONNECTED)` 验证了连接的对称性，对端是进程 ID 为 1131 的 containerd，连接状态同样是已建立。

而我们上面查看 Java 进程 2444401 持有的 domain socket，并么有发现对端信息。我们现在只知道 Java 进程通过文件描述符 6 持有一个 domain socket，但是 domain socket 的作用是什么，为什么会被创建，完全不知道。如何进一步排查呢？

## strace

首先想到的是通过 strace 追踪系统调用，看能不能发现什么有用的信息。在 Java 进程的启动命令前加入 strace：

```bash
# -f 跟踪子进程（fork/vfork/clone 创建的进程）
# -yy 增强网络信息显示，打印 socket 的完整协议信息
# -e trace=desc,network  过滤系统调用类型，跟踪文件描述符和网络相关的调用
strace -f -yy -e trace=desc,network -o /tmp/startup_trace.log java ...
```
最终在 `startup_trace.log` 文件中，没有找到文件描述 6 相关的信息，这条路没走通。

## 安装 BCC

然后想到，可以使用 eBPF 追踪系统中，所有 domain socket 的创建信息。先安装 [BCC](https://github.com/iovisor/bcc)，当前内核版本较低，从源码编译安装 BCC。

```bash
# 安装编译依赖
dnf groupinstall -y "Development Tools"
dnf install -y cmake clang llvm python3-devel elfutils-libelf-devel

# 安装 LLVM 和 Clang 开发包
dnf install -y llvm-devel clang-devel

# 查找 LLVM 安装路径
find /usr -name "LLVMConfig.cmake" 2>/dev/null
/usr/lib64/cmake/llvm/LLVMConfig.cmake
which llvm-config
/usr/bin/llvm-config

# 克隆 BCC 源码
git clone https://github.com/iovisor/bcc.git
cd bcc

# 创建构建目录
mkdir build && cd build

# 配置编译，指定 Python 路径
# 使用找到的 LLVM 路径配置 CMake，`-DLLVM_DIR` 参数应该指向包含 `LLVMConfig.cmake` 文件的目录
cmake .. -DPYTHON_CMD=python3 -DLLVM_DIR=/usr/lib64/cmake/llvm

# 编译（使用多核加速）
make -j$(nproc)

# 安装
make install
ldconfig
```

安装完成后进行验证：    
```bash
python3 -c "from bcc import BPF; print('BCC 安装成功！')"
BCC 安装成功！
```

## BCC 脚本

指挥 AI 写一个追踪 domain socket 创建的脚本 `trace_unix_socket_creation.py`：

```python
#!/usr/bin/env python3
import re
from bcc import BPF

# AF_UNIX 的值为 1
AF_UNIX = 1

prog = r"""
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>
#include <net/sock.h> // For AF_UNIX

struct data_t {
    u32 tid;
    u32 ppid;
    int stack_id;
};
BPF_HASH(infotmp, u32, struct data_t);
BPF_STACK_TRACE(stack_traces, 1024);

// Hook for socket() syscall
int trace_socket_entry(struct pt_regs *ctx, int domain, int type, int protocol) {
    if (domain != AF_UNIX) {
        return 0;
    }
    
    u32 tid = bpf_get_current_pid_tgid() >> 32;
    struct task_struct *task = (struct task_struct *)bpf_get_current_task();
    u32 ppid = 0;
    bpf_probe_read_kernel(&ppid, sizeof(ppid), &task->real_parent->tgid);

    struct data_t data = {};
    data.tid = tid;
    data.ppid = ppid;
    data.stack_id = stack_traces.get_stackid(ctx, BPF_F_USER_STACK);
    infotmp.update(&tid, &data);
    return 0;
}

int trace_socket_return(struct pt_regs *ctx) {
    u32 tid = bpf_get_current_pid_tgid() >> 32;
    struct data_t *datap = infotmp.lookup(&tid);
    if (!datap) return 0;

    int retval = PT_REGS_RC(ctx);
    if (retval >= 0) {
        bpf_trace_printk("socket(AF_UNIX) call: ppid=%d tid=%d new_fd=%d\\n", datap->ppid, datap->tid, retval);
        bpf_trace_printk("stackid=%d\\n", datap->stack_id);
    }
    infotmp.delete(&tid);
    return 0;
}
"""

b = BPF(text=prog)
b.attach_kprobe(event="__sys_socket", fn_name="trace_socket_entry")
b.attach_kretprobe(event="__sys_socket", fn_name="trace_socket_return")

print("Tracing socket(AF_UNIX, ...) calls... Ctrl-C to stop.\n")

tid_to_tgid_cache = {}

def get_tgid(tid):
    if tid in tid_to_tgid_cache:
        return tid_to_tgid_cache[tid]
    
    # --- 关键修改：处理竞态条件 ---
    try:
        with open(f"/proc/{tid}/status", "r") as f:
            for line in f:
                if line.startswith("Tgid:"):
                    tgid = int(line.split()[1])
                    tid_to_tgid_cache[tid] = tgid
                    return tgid
    # 捕获 FileNotFoundError 或 ProcessLookupError
    except (FileNotFoundError, ProcessLookupError):
        # 线程已退出，返回 -1 表示无效
        return -1
    return -1

while True:
    try:
        (task, tid, cpu, flags, ts, msg) = b.trace_fields()
        msg = msg.decode('utf-8', errors='replace').strip()
        print(msg)

        if "stackid=" in msg:
            m = re.search(r'stackid=(\d+)', msg)
            if m:
                sid = int(m.group(1))
                tgid = get_tgid(tid)

                # 只有在成功获取 TGID 时才尝试解析符号
                if tgid != -1:
                    for addr in b["stack_traces"].walk(sid):
                        print("    %s" % b.sym(addr, tgid, show_module=True, show_offset=True))
                else:
                    print(f"    [Could not get symbols for exited tid {tid}]")

    except KeyboardInterrupt:
        exit()
```

AI 还贴心的给出了脚本执行的流程图：
![flow](https://cdn.mazhen.tech/2024/202509031554586.svg)

## 最终定位
先停止 Java 进程，然后运行该脚本，在启动 Java 进程。

```bash
./trace_unix_socket_creation.py > socket_trace
```
在 `socket_trace` 中根据 Java 进程的 PID 搜索，找到了 文件描述 6 的创建调用堆栈：

```
socket(AF_UNIX) call: ppid=1527014 tid=1527479 new_fd=6\n
stackid=523\n
    b'socket+0x8 [libc-2.28.so]'
    b'[unknown] [libnss_sss.so.2]'
    b'_nss_sss_getpwuid_r+0xe4 [libnss_sss.so.2]'
    b'getpwuid_r+0x14c [libc-2.28.so]'
    b'get_user_name(unsigned int)+0x5c [libjvm.so]'
    b'PerfMemory::create_memory_region(unsigned long)+0x94 [libjvm.so]'
    b'perfMemory_init()+0xa8 [libjvm.so]'
    b'vm_init_globals()+0x24 [libjvm.so]'
    b'Threads::create_vm(JavaVMInitArgs*, bool*)+0x234 [libjvm.so]'
    b'JNI_CreateJavaVM+0x80 [libjvm.so]'
    b'JavaMain+0x80 [libjli.so]'
    b'ThreadJavaMain+0xc [libjli.so]'
    b'[unknown] [libpthread-2.28.so]'
    b'[unknown] [libc-2.28.so]'
```

调用堆栈记录了文件描述符 6 被创建的全过程，让我们从下往上，一步步还原：

1.  **`JNI_CreateJavaVM`**: Java 虚拟机的诞生。当我们执行 `java` 命令时，`libjli.so` (Java Launcher Interface) 库会加载核心的 JVM 库 (`libjvm.so`)，并调用 `JNI_CreateJavaVM` 这个函数来真正创建和初始化一个 JVM 实例。

2.  **`Threads::create_vm` -> `vm_init_globals`**: 在 JVM 内部，`create_vm` 函数开始搭建虚拟机运行所需的基础环境，其中一步就是初始化各种全局组件 (`vm_init_globals`)。

3.  **`perfMemory_init`**: `perfMemory` 子系统被初始化。这个子系统是 JVM 性能监控的关键。它负责创建一块特殊的内存区域，用于存放 JVM 内部的性能计数器，例如 JIT 编译统计、垃圾回收次数和耗时等。我们熟知的 `jps`、`jstat`、`jcmd` 等命令行工具，正是通过读取这块内存区域来获取 JVM 实时运行数据的。

4.  **`PerfMemory::create_memory_region` -> `get_user_name`**: `PerfMemory` 创建的这块内存区域，实际上是以内存映射文件（memory-mapped file）的形式存在的，这些文件通常位于 `/tmp/hsperfdata_<username>/` 目录下（`hsperfdata` 是 HotSpot Performance Data 的缩写）。为了构建这个目录路径，JVM 需要知道当前运行它的用户名是什么，于是它调用了内部函数 `get_user_name`。

5.  **`getpwuid_r` -> `_nss_sss_getpwuid_r`**: `get_user_name` 函数通过调用标准的 glibc 函数 `getpwuid_r`，根据当前用户的 UID (User ID) 来查询其用户名。在现代 Linux 系统中，这个查询过程并非简单地读取 `/etc/passwd` 文件。它是由 **NSS (Name Service Switch)** 机制来管理的。NSS 会根据 `/etc/nsswitch.conf` 的配置，将这类查询请求转发给相应的处理模块。在我们的例子中，请求被转发给了 **SSSD (System Security Services Daemon)** 的客户端库 `libnss_sss.so.2`。

6.  **`[unknown] [libnss_sss.so.2]` -> `socket`**: `libnss_sss.so.2` 库只是一个“中间人”，它需要和后台运行的 `sssd` 守护进程进行通信，才能真正完成用户信息的查询。它选择的通信方式正是 Unix Domain Socket。于是，它调用了 `socket()` 系统调用来创建一个套接字，准备连接 `sssd` 服务。

7.  **`new_fd=6`**: 在这个精确的时间点，我们的 Java 进程中最小的可用文件描述符编号恰好是 6。因此，内核将 6 这个编号分配给了这个新创建的套接字。

所以，文件描述符 6 并非由我们的 Java 应用代码直接创建，而是在 JVM 启动的极早期，由其内部的性能监控子系统为了获取当前用户名，间接触发了系统的 NSS 模块，进而由 SSSD 客户端库创建的、用于和 SSSD 后台服务通信的 Unix Domain Socket。

## SSSD 是什么

从堆栈分析可知，这个 Domain Socket 是为了连接 SSSD 服务。SSSD (System Security Services Daemon) 是现代 Linux 系统中用于集中管理身份认证、授权和用户信息查询的核心服务。它充当了一个缓存和转发层，可以对接多种后端服务，如本地文件、LDAP、Kerberos 或 Active Directory。

SSSD 的作用是响应来自 `libnss_sss` 库的请求，查询并返回当前 UID 对应的用户名。`libc` 库在第一次查询后，通常会保持这个套接字连接打开，以便后续的查询可以复用这个连接，避免重复创建和连接的开销。这解释了为什么在 Java 进程的整个生命周期中，我们都能看到这个 FD 6 的存在。

### 查看 SSSD 进程及其配置

我们可以使用 `systemctl` 来确认 `sssd` 服务的运行状态。

```bash
# 检查 sssd 服务的总体状态
systemctl status sssd
● sssd.service - System Security Services Daemon
   Loaded: loaded (/usr/lib/systemd/system/sssd.service; enabled; vendor preset: disabled)
   Active: active (running) since Tue 2025-08-05 11:03:10 CST; 4 weeks 1 days ago
 Main PID: 916 (sssd)
    Tasks: 4
   Memory: 45.8M
   CGroup: /system.slice/sssd.service
           ├─ 916 /usr/sbin/sssd -i --logger=files
           ├─1027 /usr/libexec/sssd/sssd_be --domain implicit_files --uid 0 --gid 0 --logger=files
           ├─1054 /usr/libexec/sssd/sssd_nss --uid 0 --gid 0 --logger=files
           └─1055 /usr/libexec/sssd/sssd_pam --uid 0 --gid 0 --logger=files
```

查看 `/etc/nsswitch.conf` 文件，是否在 passwd、group、shadow 等关键数据库的查询源列表中包含了 sss。
```bash
cat /etc/nsswitch.conf
...
passwd:      sss files systemd
shadow:     files sss
group:       sss files systemd
...
```

`passwd` 这一行表明，当需要查询用户信息时，首先就去问 SSSD 服务 (sss)，然后再查找本地的 /etc/passwd 文件 (files)，如果前两者都找不到，还会查询由 systemd-logind 管理的动态用户。

这个配置清晰地表明，SSSD 在该系统的用户和组信息查询中处于最高优先级。这解释了，在 JVM 在启动时，为了获取当前用户名，就会立即触发与 SSSD 的通信。

## 总结

通过 `lsof` 初步定位问题，再借助强大的 eBPF 工具 BCC 深入追踪系统调用，成功地揭开了 JVM 底层的文件描述符的秘密。它并非由应用代码创建，而是 JVM 初始化性能监控模块时，通过 NSS 机制与系统核心的 SSSD 服务建立的通信通道。
