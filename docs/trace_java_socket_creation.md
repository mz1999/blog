# 追踪定位 Java 进程的 Socket 创建

在使用 [CRaC](https://openjdk.org/projects/crac/) 创建 checkpoint 镜像时，需要 Java 应用能够恰当处理它持有的外部资源，例如打开的日志文件，监听的服务端口，对外创建的数据库连接池等。

对于 Java 应用打开的文件，可以通过定义[文件描述符策略](https://docs.azul.com/core/crac/fd-policies) 让 CRaC 自动处理。对于应用启用的监听端口，或创建的连接池，一般建议应用实现 CRaC 的 [Resource](https://github.com/CRaC/org.crac/blob/master/src/main/java/org/crac/Resource.java) 接口，在 `checkpoint` 前关闭资源，在 `restore` 后重新打开。

要妥善的处理 Java 持有的网络资源，首先需要知道这些资源具体是在哪里创建的。然而在实际开发中，应用程序通常不会直接调用 JDK 的基础网络 API，大多数情况下是通过 Netty、Dubbo 等框架来处理网络连接，所以造成开发人员可能自己都不知道在哪里打开了监听端口，或者创建了网络连接。

还有一些资源，例如 `Unix socket`，一般不是 Java 直接创建持有的，而是底层的 JVM 或依赖的 C 库打开的，要追踪定位这类资源的创建，就更加困难了。

本文将介绍如何使用 [BCC](https://github.com/iovisor/bcc)  和 [async-profiler](https://github.com/async-profiler/async-profiler) 追踪定位 Java 应用中创建 `Socket` 的具体位置。

## 使用 BCC 追踪定位 Unix Socket 的创建

我们从一个具体的的例子开始。在为某个 Java 应用创建 `checkpoint` 镜像时，遇到了如下异常：

```java
Suppressed: jdk.internal.crac.mirror.impl.CheckpointOpenSocketException: FD fd=4 type=socket path=socket:[7504404],port=29295
                at java.base/jdk.internal.crac.mirror.Core.translateJVMExceptions(Core.java:116)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestore1(Core.java:189)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestore(Core.java:315)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestoreInternal(Core.java:328)
```

尽管已经添加了 JVM 参数 `-Djdk.crac.collect-fd-stacktraces=true`，但日志中仍然没有输出文件描述符 4（FD 4）的具体打开位置。这通常意味着 FD 4 很可能是在 Native Code 中被打开的。

我们使用 `lsof` 命令查看 FD 4 的详细信息：

```bash
$ lsof -p 172963 | grep 7504404     
exe     172963 mazhen    4u     unix 0x00000000bfe2338f       0t0  7504404 type=STREAM
```

第五列的 `unix` 表明，该文件描述符对应的资源是一个 `Unix socket`。

为了进一步调研这个 `Unix socket` 的来源和作用，我们需要知道它的另一端连接着哪个进程：

```bash
# -U ：筛选 Unix Socket。
# -a：逻辑 “与”（AND）操作，将前面的条件（-p 和 -U）组合，表示必须同时满足
# +E：显示套接字的端点信息（Endpoints）。
$ sudo lsof -p 172963 -U -a +E
COMMAND    PID          USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
lwsmd     1373          root   73u  unix 0x000000001b73058d      0t0 7506688 /var/lib/pbis/.lsassd type=STREAM ->INO=7504404 172963,exe,4u
exe     172963        mazhen    4u  unix 0x00000000bfe2338f      0t0 7504404 type=STREAM ->INO=7506688 1373,lwsmd,73u
exe     172963        mazhen  407u  unix 0x000000005c8b74c1      0t0 7507634 type=STREAM ->INO=7507635 172963,exe,408u
exe     172963        mazhen  408u  unix 0x00000000cf9143dc      0t0 7507635 type=STREAM ->INO=7507634 172963,exe,407u
```

从输出中可以看出，文件描述符 4 连接到外部进程 `lwsmd`。这是一个安全认证相关的服务，用于将 Linux 集成到 Windows 的 Active Directory。Java 进程通过 `Unix socket` 和 `lwsmd` 通信，以实现用户认证和权限管理。

此外，我们还发现了一对文件描述符 407 和 408，它们是一个 `Unix socket` 的两端，并且都是由该 Java 进程自身创建和持有。在创建 checkpoint 镜像时，这对资源并没有抛出异常，说明它们已经被妥善处理。

现在我们知道了 Java 进程会创建这些 `Unix socket`，但关键问题是：**如何定位到具体的创建位置呢？**

要找到创建 `Unix socket` 的源头，最直接的方法就是追踪它在内核层面触发的**系统调用**。通过捕获系统调用发生时的完整调用堆栈，我们就能反向追溯到具体的代码位置。

在应用程序层面，代码通过调用 `glibc` 提供的 `socket()` 和 `socketpair()` **库函数** 创建 `Unix socket`，它们的会准备好参数，然后执行一条特殊的 CPU 指令，使程序从用户模式切换到内核模式，从而发起真正的**系统调用**。

为了在内核中观察到这个事件，我们可以利用内核的静态探测点：**Tracepoints**。`Tracepoint` 是内核开发者在内核代码中预设的静态探测点，允许我们观察内核的关键活动。当一个系统调用请求进入内核时，相应的 `tracepoint` 就会被触发，我们可以使用工具捕获这个事件，获取当时的上下文信息，包括调用堆栈。

对于创建 `Unix socket` 的操作，我们关心以下两个 `tracepoint`：

- `syscalls:sys_enter_socket`: 当 `socket()` 系统调用进入内核时触发。
- `syscalls:sys_enter_socketpair`: 当 `socketpair()` 系统调用进入内核时触发。

由于 `socket()` 函数可以创建多种类型的 socket (TCP, UDP 等)，我们必须在追踪时添加一个过滤条件，只捕获其第一个参数 `family` 为 `AF_UNIX` 的调用，这样才能精确地锁定 **Unix socket** 的创建。

要实现这一点，我们将使用 [BCC（BPF Compiler Collection）](https://github.com/iovisor/bcc) 这是一个强大的工具集。

BCC 自带了一个通用的 [trace](https://github.com/iovisor/bcc/blob/master/tools/trace.py) 工具，能够追踪任意函数，我们可以先用它来快速验证一下思路：

```bash
$ sudo python3 /usr/share/bcc/tools/trace -K -U 't:syscalls:sys_enter_socket (args->family == 1) "socket(family=%d, type=%d)", args->family, args->type'
```
* `-K`, `-U`: 分别表示捕获内核和用户空间的堆栈。
* `t:syscalls:sys_enter_socket`: 指定要追踪的 `tracepoint`。
* `(args->family == 1)`: 过滤器，只关心 `AF_UNIX` (值为 1) 类型的 socket 创建。

这个命令的输出如下：

```bash
177325  177330  java            sys_enter_socket socket(family=1, type=526337)
        -14        socket+0xb [libc.so.6]
        __nscd_open_socket+0x3b [libc.so.6]
        inet_pton+0x2e [libc.so.6]
        ...
```

可以看出，这个命令成功捕获到了 `socket` 调用及其堆栈。但它有一个很大的缺点：**输出中不包含系统调用返回的文件描述符**。因此，我们无法将这个堆栈与我们关心的 FD 4 关联起来。

为了获取 `socket` 或 `socketpair()` 返回的文件描述符，我们需要编写一个自定义的 BCC 脚本：

```python
#!/usr/bin/env python3
from bcc import BPF
from datetime import datetime

# BPF C code
prog = r"""
#include <uapi/linux/ptrace.h>
#include <linux/sched.h>
#include <net/sock.h> // for AF_UNIX

// Define data structure to send to userspace
enum call_type {
    TYPE_SOCKET = 1,
    TYPE_SOCKETPAIR = 2,
};

struct event_t {
    u64 ts;
    u32 pid;
    u32 tid;
    u32 ppid;
    int stack_id;
    int fds[2]; // fds[0] for socket, fds[0] & fds[1] for socketpair
    enum call_type call_type;
};
BPF_PERF_OUTPUT(events);

// Used to pass socketpair parameter pointer between enter and exit
struct data_t {
    int *sv_ptr;
};
BPF_HASH(infotmp, u32, struct data_t);

BPF_STACK_TRACE(stack_traces, 16384);

// --- tracepoint for socket() ---
TRACEPOINT_PROBE(syscalls, sys_enter_socket) {
    if (args->family != AF_UNIX) {
        return 0;
    }

    u32 tid = (u32)bpf_get_current_pid_tgid();
    struct data_t data = {};
    data.sv_ptr = NULL; //  Mark this as socket() call
    infotmp.update(&tid, &data);

    return 0;
}

TRACEPOINT_PROBE(syscalls, sys_exit_socket) {
    u32 tid = (u32)bpf_get_current_pid_tgid();
    struct data_t *datap = infotmp.lookup(&tid);
    if (!datap) {
        return 0;
    }

    int retval = args->ret;
    if (retval >= 0) {
        struct event_t event = {};
        struct task_struct *task = (struct task_struct *)bpf_get_current_task();
        u64 id = bpf_get_current_pid_tgid();

        event.ts = bpf_ktime_get_ns();
        event.pid = id >> 32;
        event.tid = (u32)id;
        bpf_probe_read_kernel(&event.ppid, sizeof(event.ppid), &task->real_parent->tgid);
        event.stack_id = stack_traces.get_stackid(args, BPF_F_USER_STACK);
        event.call_type = TYPE_SOCKET;
        event.fds[0] = retval;
        event.fds[1] = -1;

        events.perf_submit(args, &event, sizeof(event));
    }

    infotmp.delete(&tid);
    return 0;
}

// --- tracepoint for socketpair() ---
TRACEPOINT_PROBE(syscalls, sys_enter_socketpair) {
    if (args->family != AF_UNIX) {
        return 0;
    }

    u32 tid = (u32)bpf_get_current_pid_tgid();
    struct data_t data = {};
    data.sv_ptr = (int *)args->usockvec;
    infotmp.update(&tid, &data);

    return 0;
}

TRACEPOINT_PROBE(syscalls, sys_exit_socketpair) {
    u32 tid = (u32)bpf_get_current_pid_tgid();
    struct data_t *datap = infotmp.lookup(&tid);

    if (!datap || !datap->sv_ptr) {
        if (datap) {
            infotmp.delete(&tid);
        }
        return 0;
    }

    int retval = args->ret;
    if (retval == 0) {
        struct event_t event = {};
        struct task_struct *task = (struct task_struct *)bpf_get_current_task();
        u64 id = bpf_get_current_pid_tgid();

        event.ts = bpf_ktime_get_ns();
        event.pid = id >> 32;
        event.tid = (u32)id;
        bpf_probe_read_kernel(&event.ppid, sizeof(event.ppid), &task->real_parent->tgid);
        event.stack_id = stack_traces.get_stackid(args, BPF_F_USER_STACK);
        event.call_type = TYPE_SOCKETPAIR;
        bpf_probe_read_user(&event.fds, sizeof(event.fds), datap->sv_ptr);

        events.perf_submit(args, &event, sizeof(event));
    }

    infotmp.delete(&tid);
    return 0;
}
"""

# Load BPF program
b = BPF(text=prog)

print("Tracing socket(AF_UNIX) and socketpair(AF_UNIX) calls... Ctrl-C to stop.\n")


# Perf Buffer callback handler
def print_event(cpu, data, size):
    event = b["events"].event(data)

    time_str = datetime.fromtimestamp(event.ts / 1e9).strftime('%H:%M:%S.%f')

    tgid = event.pid

    if event.call_type == 1:  # TYPE_SOCKET
        call_str = "[socket(AF_UNIX)]"
        fds_str = f"[FD={event.fds[0]}]"
    elif event.call_type == 2:  # TYPE_SOCKETPAIR
        call_str = "[socketpair(AF_UNIX)]"
        fds_str = f"[FDs=[{event.fds[0]}, {event.fds[1]}]]"
    else:
        call_str = "UNKNOWN"
        fds_str = ""

    print(f"[{time_str}] {call_str} [PPID={event.ppid}] [PID={event.pid}] {fds_str}")

    try:
        for addr in b["stack_traces"].walk(event.stack_id):
            print("    %s" % b.sym(addr, tgid, show_module=True, show_offset=True))
    except KeyError:
        print("    [Stack trace unavailable for stack_id %d due to process exit]" % event.stack_id)

    print("")


# Open perf buffer and set callback function
b["events"].open_perf_buffer(print_event)

# Main loop, poll perf buffer
while True:
    try:
        b.perf_buffer_poll()
    except KeyboardInterrupt:
        exit()
```

这个脚本的思路是在系统调用**进入** (`sys_enter_*`) 时暂存信息，在**退出** (`sys_exit_*`) 时捕获返回值（即文件描述符），然后将文件描述符、堆栈信息等一并发送到用户空间的 Python 程序进行打印。

先运行这个脚本，然后再启动 Java 应用。

```bash
$ sudo ./trace_unix_socket.py > unixsocket
# 在另一个终端启动 Java 应用
# 获取 Java 进程的 PID
$ jps
178106 Jps
177961 GlassFishMain
# 再次使用 lsof 确认 Unix socket
$ sudo lsof -p 177961 -U -a +E 
COMMAND    PID          USER   FD   TYPE             DEVICE SIZE/OFF    NODE NAME
lwsmd     1373          root   78u  unix 0x000000001e5b48ae      0t0 7629677 /var/lib/pbis/.lsassd type=STREAM ->INO=7625335 177961,exe,4u
exe     177961        mazhen    4u  unix 0x00000000b7b4fa52      0t0 7625335 type=STREAM ->INO=7629677 1373,lwsmd,78u
exe     177961        mazhen  407u  unix 0x0000000078480cf3      0t0 7630050 type=STREAM ->INO=7630051 177961,exe,408u
exe     177961        mazhen  408u  unix 0x000000002d0320c8      0t0 7630051 type=STREAM ->INO=7630050 177961,exe,407u
```

现在，我们可以在脚本的输出文件 `unixsocket` 中，根据进程 PID (177961) 和文件描述符（FD 4，407，408）进行查找，找到对应的创建堆栈：

```bash
...
[12:23:13.184181] [socket(AF_UNIX)] [PPID=177932] [PID=177961] [FD=4]
    b'socket+0xb [libc.so.6]'
    b'__nscd_get_mapping+0xbd [libc.so.6]'
    b'__nscd_get_map_ref+0xcf [libc.so.6]'
    b'[unknown]'
...
[12:23:20.370346] [socketpair(AF_UNIX)] [PPID=2583] [PID=177961] [FDs=[407, 408]]
    b'socketpair+0xe [libc.so.6]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'[unknown]'
    b'JavaCalls::call_helper(JavaValue*, methodHandle const&, JavaCallArguments*, JavaThread*)+0x334 [libjvm.so]'
    b'JavaCalls::call_virtual(JavaValue*, Handle, Klass*, Symbol*, Symbol*, JavaThread*)+0x20c [libjvm.so]'
    b'thread_entry(JavaThread*, JavaThread*)+0x70 [libjvm.so]'
    b'JavaThread::run()+0x127 [libjvm.so]'
    b'Thread::call_run()+0xa1 [libjvm.so]'
    b'thread_native_entry(Thread*)+0xe3 [libjvm.so]'
    b'start_thread+0x2f3 [libc.so.6]'
```

对于 FD 4，调用堆栈中的 `__nscd_get_mapping` 和 `__nscd_get_map_ref` 函数是关键线索，它表明 glibc 正在尝试进行一次名称服务查询（例如，解析一个用户名）。这正好解释了前面我们观察到的现象，为何 Java 进程会连接到 `lwsmd`。`__nscd_*` 是 glibc 用于名称解析的通用客户端逻辑，而具体解析动作需要连接到哪个服务，则由系统的 NSS(Name Service Switch) 决定。在我的测试场景，NSS 的配置将认证查询指向了 `lwsmd`，所以 Java 进程才会通过 `Unix socket`连接 `lwsmd`。至此，我们知道了 FD 4 的作用和来源，至于如何处理这个资源，还需要后续再探索，但至少现在我们已经明确了问题的根源。

再看 FD 407 和 408，我们可以看到 `libjvm.so` 的函数调用，这清晰地表明是 Java 线程调用了 `socketpair` 创建了这对 `Unix socket`。然而，最关键的调用栈部分却显示为 `[unknown]`。

这是因为 JIT (Just-In-Time) 编译器在运行时会将热点代码动态编译成机器码，并存放在匿名的内存区域。这些区域没有传统的符号表，因此像 BCC 这样的通用系统级工具在试图解析这些内存地址时，找不到任何符号信息，只能无奈地显示为 `[unknown]`。

虽然这个 `Unix socket` (FD 407, 408) 已经被妥善处理，没有引起 CRaC 异常，但如果我们出于好奇，想知道究竟是哪段 Java 代码创建了它，应该如何做呢？

要解决 `[unknown]` 的问题，我们需要一个更了解 Java 运行时的工具： [async-profiler](https://github.com/async-profiler/async-profiler) 。

## 使用 async-profiler 追踪 Socket 创建

[async-profiler](https://github.com/async-profiler/async-profiler) 是一个为 Java 设计的低开销、高精度的性能分析工具，它直接利用 Linux 的 `perf_events` 子系统和 HotSpot JVM 特有的 API 来收集性能数据。

相比传统 Java Profiler，它不仅能分析 Java 代码，还能监控到非 Java 线程（如 GC、JIT 编译器线程）的活动，并能展示原生代码（Native）和内核（Kernel）的调用栈帧，让我们看到从 Java 方法到 C/C++ 库函数再到内核系统调用的完整链路。

BCC 等通用系统追踪工具，它们不理解 JVM 的内部工作原理，当遇到 JIT 编译器在匿名内存区域生成的机器码时，找不到对应的符号表，无法解析出方法名。而 `async-profiler` 是利用 HotSpot 提供的特定 API（如 AsyncGetCallTrace），它能解析 JVM 内部数据结构，从而获取 JIT 编译后代码的符号信息，将内存地址精确地映射回原始的 Java 方法名。

可以这么说，`async-profiler` 综合了系统级追踪工具和传统 Java profiler 的能力，提供了对 Java 应用完整、精确性能洞察。

### 追踪 socketpair 的创建位置

上一节，我们利用 BCC 追踪 `Unix socket` 相关的系统调用，这次我们用 `async-profiler` 完成同样的功能，并且能在调用堆栈中展示出 Java 方法名。

`async-profiler` 支持使用 Linux 的 `perf_events` ，所以它能够捕获 `perf_events` 中的 `tracepoint` 事件，实现和 BCC 一样的系统调用追踪。同时，`async-profiler` 可以作为一个 Java Agent 启动，这意味着我们可以在 Java 进程启动时就加载它，不会错过 Java 应用早期的初始化行为。

回到之前的问题，为了定位创建文件描述符 408 和 409 的 `socketpair` 调用究竟源于哪段 Java 代码，我们可以在启动 Java 应用时，通过 `-agentpath` 参数挂载 `async-profiler`，并指定追踪 `syscalls:sys_enter_socketpair` 事件。

修改 Java 启动命令如下：

```bash
java -agentpath:/path/to/libasyncProfiler.so=start,event=syscalls:sys_enter_socketpair,file=/tmp/socketpair-trace.html ...
```

参数说明：
*   `-agentpath:/path/to/libasyncProfiler.so`: 指定 `async-profiler` 动态链接库的位置。
*   `start`: 表示立即开始分析。
*   `event=syscalls:sys_enter_socketpair`: 指定要追踪的事件。这里我们关心 `socketpair` 系统调用的入口。
*   `file=/tmp/socketpair-trace.html`: 将分析结果输出为一个 HTML 格式的火焰图。

需要注意的是，一般内核默认禁止非 root 用户追踪某些敏感的内核事件（比如系统调用），所以最好使用 `sudo` 启动 Java 应用。

应用正常运行并结束后，会根据配置生成 `/tmp/socketpair-trace.html` 文件。

![sys_enter_socketpair](https://cdn.mazhen.tech/2025/202509151411050.png)

这张火焰图清晰地揭示了之前 BCC 无法解析的 [unknown] 部分，可以精准地定位到创建 `socketpair` 的 Java 代码源头。

从上图可以看出，这个 `socketpair` 是由 Java 的 WatchService API 在其 Linux 实现中创建的。WatchService 是 Java NIO 提供的一个标准接口，用于监控文件系统的目录变化（如文件的创建、修改或删除）。调用堆栈显示，是 Apache Felix 框架的 FileInstall 组件调用了 WatchService，其目的是为了监控一个部署目录，从而实现 OSGi bundle 的热部署功能。

我们前面观察到，这对由 `socketpair` 创建的文件描述符并没有在 checkpoint 过程中引发异常。原因是，CRaC 已经专门为 Java 的 `WatchService` 在 Linux 上的实现提供了内置支持（见 [OpenJDK CRaC PR #72](https://github.com/openjdk/crac/pull/72)）。这意味着 CRaC 能够自动识别并妥善处理 `WatchService` 在内部使用的相关文件描述符。

### 追踪服务端口的监听位置

同样的原理也适用于定位网络服务端口的监听位置。对于使用 NIO 框架创建的网络服务，如果要适配 CRaC，需要应用实现 `Resource` 接口，处理好监听端口的关闭和重新打开。如果未做适当的处理，那么在创建 `checkpoint` 镜像时，则会遇到类似下面的异常：

```java
jdk.internal.crac.mirror.CheckpointException
        Suppressed: sun.nio.ch.EPollSelectorImpl$BusySelectorException: Selector sun.nio.ch.EPollSelectorImpl@476d29ab has registered keys from channels: [sun.nio.ch.ServerSocketChannelImpl[closed]]
                at java.base/sun.nio.ch.EPollSelectorImpl.beforeCheckpoint(EPollSelectorImpl.java:405)
                at java.base/jdk.internal.crac.mirror.impl.AbstractContext.invokeBeforeCheckpoint(AbstractContext.java:43)
                at java.base/jdk.internal.crac.mirror.impl.AbstractContext.beforeCheckpoint(AbstractContext.java:58)
                at java.base/jdk.internal.crac.mirror.impl.BlockingOrderedContext.beforeCheckpoint(BlockingOrderedContext.java:64)
                at java.base/jdk.internal.crac.mirror.impl.AbstractContext.invokeBeforeCheckpoint(AbstractContext.java:43)
                at java.base/jdk.internal.crac.mirror.impl.AbstractContext.beforeCheckpoint(AbstractContext.java:58)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestore1(Core.java:154)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestore(Core.java:315)
                at java.base/jdk.internal.crac.mirror.Core.checkpointRestoreInternal(Core.java:328)
```

从异常信息中，我们知道 `checkpoint` 失败的直接原因是 `EPollSelectorImpl` 实例正处于“忙碌”状态。可以看出，即使 `ServerSocketChannel` 本身可能已经关闭，但它与 `Selector` 的注册关系没有被完全解除。

但这个异常信息并没有告诉我们这个端口是在应用程序的**哪个位置**被创建和监听的。要妥善的处理监听端口，必须先找到创建监听端口的具体位置。

如何做呢？我们可以追踪一个关键的系统调用：`bind`。

`bind` 系统调用是任何网络服务端程序启动的必经之路。它的作用是将一个创建好的 `socket` 文件描述符与一个具体的 IP 地址和端口号绑定起来。只有执行了 `bind` 并成功之后，服务器才能在该端口上监听并接受客户端连接。因此，通过捕获 `bind` 系统调用发生时的调用堆栈，我们就能精确地定位到是哪一段 Java 代码触发了端口监听。

使用以下命令启动 Java 应用：

```bash
java -agentpath:/path/to/libasyncProfiler.so=start,event=syscalls:sys_enter_bind,file=/tmp/bind-trace.html -jar ...
```

应用成功启动并对外提供服务后，停止应用，然后分析生成的 `/tmp/bind-trace.html` 文件。

![sys_enter_bind](https://cdn.mazhen.tech/2025/202509151452321.png)

从上面的火焰图可以看出，端口绑定操作是由 Grizzly NIO 框架执行的，该框架作为 GlassFish 应用服务器的底层网络引擎。整个过程在服务器启动阶段，由 HK2 依赖注入框架自动触发，以初始化和启动核心的网络服务。

通过分析火焰图，我们可以得出结论：在 `GrizzlyListener` 处实现 `Resource` 接口可能比较合理，可以在这里管理监听端口的关闭和重启。

## 总结
本文旨在解决为 Java 应用适配 CRaC 时，如何精确定位网络资源（如 `Unix socket` 和服务监听端口）创建位置的难题。

首先介绍了使用系统级追踪工具 **BCC** 的方法。虽然 BCC 能通过追踪 `socket` 和 `socketpair` 等系统调用，捕获到原生代码的调用堆栈，但它无法解析 JIT 编译的 Java 代码，导致关键信息显示为 `[unknown]`。

为解决此问题，文章介绍了更为强大的 **`async-profiler`**。它结合了底层 `perf_events` 追踪能力，以及对 JVM 内部的理解，能够完美解析 JIT 代码的符号。通过追踪 `sys_enter_socketpair` 和 `sys_enter_bind` tracepoint 的实际案例，展示了如何利用 `async-profiler` 生成的火焰图，将系统调用反向追溯到具体的 Java 代码。