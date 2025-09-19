# 使用 DWARF 还原完整的 glibc 调用堆栈

在上一篇文章[追踪定位 Java 进程的 Socket 创建](https://mazhen.tech/p/%E8%BF%BD%E8%B8%AA%E5%AE%9A%E4%BD%8D-java-%E8%BF%9B%E7%A8%8B%E7%9A%84-socket-%E5%88%9B%E5%BB%BA/)中，我们使用 BCC 脚本成功捕获了 Java 进程在启动阶段对 `Unix socket` 的创建。其中，对于一个由 JVM 原生代码触发的 `socket` 调用，我们得到了如下调用堆栈：

```shell
[12:23:13.184181] [socket(AF_UNIX)] [PPID=177932] [PID=177961] [FD=4]
    b'socket+0xb [libc.so.6]'
    b'__nscd_get_mapping+0xbd [libc.so.6]'
    b'__nscd_get_map_ref+0xcf [libc.so.6]'
    b'[unknown]'
```

虽然我们根据堆栈中的 `__nscd_get_mapping` 等函数名推断出其行为与名称服务查询有关，但这个结果留下了一个明显的问题：调用堆栈在 `__nscd_get_map_ref` 函数之后就中断了，最后一行显示为 `[unknown]`。

`__nscd_get_map_ref` 这个函数名表明它是一个内部实现函数，而非供外部程序直接调用的公共 API。因此，调用它的函数极有可能仍然位于 `glibc` 库的内部。这就引出了一个疑问：**为什么堆栈会在 `glibc` 库的内部中断？`[unknown]` 的背后究竟隐藏着哪个函数？**

本文将深入分析这个现象的底层原因，并提供一个能获取完整原生调用堆栈的方法。

## 丢失的帧指针 (Frame Pointers)

要理解堆栈为什么会中断，我们首先需要了解 BCC 默认是如何获取调用堆栈的。它使用的是一种名为“**帧指针回溯**”（Frame Pointer Walking）的技术。

简单来说，当一个函数被调用时，它会在自己的**栈帧**（**Stack Frame**）中保存上一个函数栈帧的基地址。这个基地址通常存储在一个专用的寄存器中（在 x86-64 架构上是 `%rbp`），这个寄存器就被称为**帧指针**（**Frame Pointer**）。通过当前函数的帧指针，可以找到上一个函数的帧指针，如此层层递进，形成一个类似链表的结构。BCC 正是依赖这条“指针链”来快速地回溯出完整的函数调用关系。

这种方法的优点是速度快、性能开销很低。但它的缺点是，它严重依赖于被追踪的程序在编译时保留了帧指针。如果编译器为了性能优化（例如，使用 `-fomit-frame-pointer` 标志），决定不使用 `%rbp` 寄存器作为帧指针，而是将其当作一个通用寄存器来使用，那么这个函数栈帧中就不会包含保存上一个帧指针的操作。当堆栈回溯到这个函数时，指针链就会在此处断裂。

基于以上原理，我们可以做出一个合理的推测：BCC 的堆栈追踪之所以在 `glibc` 内部中断，是因为调用链中的某个函数在编译时被优化，省略了帧指针，导致回溯链条在此处断裂。

## 历史回顾，为何帧指针会“丢失”？

前面我们推测堆栈中断是由于 glibc 中的某些函数在编译时省略了帧指针。但为什么编译器会做出这样的优化呢？

知名性能专家 [Brendan Gregg](https://www.brendangregg.com/) 在他的文章 [The Return of the Frame Pointers](https://www.brendangregg.com/blog/2024-03-17/the-return-of-the-frame-pointers.html) 中详细阐述了这个问题。在过去的 32 位 CPU 架构（如 x86）时代，CPU 可用的通用寄存器数量非常有限。为了尽可能地提升性能，编译器开发者无所不用其极。其中一项广为采用的优化就是通过 `-fomit-frame-pointer` 编译标志，不再将 `%ebp` 寄存器固定用作帧指针，而是将其解放出来，作为一个通用的寄存器参与数据计算。

这个优化的理论收益是：可以减少函数调用时的指令数，并多出一个可用的寄存器，从而生成体积更小、执行速度可能更快的代码。代价则是牺牲了程序的调试性和可分析性，使得基于帧指针的堆栈回溯变得不可靠。在那个年代，这个选择被认为是值得的，并逐渐成为了 GCC 等主流编译器的默认行为。

在现代的 64 位 CPU (x86-64) 上，寄存器数量已经大大增加，省略帧指针带来的性能优势已经微乎其微，但这个编译默认选项却作为一种“历史惯例”被长期保留了下来。因此，我们当前系统上使用的 `glibc` 库，其二进制文件很可能就是遵循这一惯例构建的产物。

业界已经重新认识到帧指针对于系统可观测性的重要性。现在，主流的 Linux 发行版（如 Ubuntu 24.04 和 Fedora 38）已经开始在其软件包的编译选项中重新默认启用帧指针。但在那之前，我们仍然需要处理这些“历史遗留”的二进制文件。

## 使用 GDB 验证猜想

现在我们使用 GDB (GNU Debugger) 直接检查相关函数的汇编代码，验证 `glibc` 内部的某些函数可能省略了帧指针的推测。

**加载 `libc.so.6` 到 GDB**

我们首先启动 GDB，并将 `glibc` 库作为分析目标加载进来。

```bash
$ gdb /usr/lib/x86_64-linux-gnu/libc.so.6
```

**检查导致回溯失败的函数 `__nscd_get_map_ref`**

根据 BCC 的输出，`__nscd_get_map_ref` 是堆栈中可见的最后一个函数名。回溯正是在尝试寻找**它的调用者**时失败的。因此，我们首先来检查这个函数的汇编代码。

```gdb
(gdb) disassemble __nscd_get_map_ref
```

输出结果会类似下面这样：

```assembly
Dump of assembler code for function __nscd_get_map_ref:
   0x0000000000171670 <+0>:	endbr64 
   0x0000000000171674 <+4>:	push   %r15
   0x0000000000171676 <+6>:	push   %r14
   0x0000000000171678 <+8>:	push   %r13
   0x000000000017167a <+10>:	push   %r12
   0x000000000017167c <+12>:	push   %rbp
   0x000000000017167d <+13>:	push   %rbx
   0x000000000017167e <+14>:	sub    $0x28,%rsp
   ...
```
我们看到，这个函数的序言**没有** `push %rbp` 和 `mov %rsp,%rbp`。它直接通过 `sub` 指令来操作栈顶指针 `%rsp` 以分配栈空间。这明确地证明了 `__nscd_get_map_ref` 这个函数在编译时**省略了帧指针**。正是因为缺少这个关键的“路标”，BCC 无法找到其调用者的栈帧，导致回溯在此中断，下一行只能显示为 `[unknown]`。

**检查被成功“穿越”的函数 `__nscd_get_mapping`**

我们再来检查一下堆栈中的前一个函数 `__nscd_get_mapping`。BCC 成功地从这个函数回溯到了 `__nscd_get_map_ref`，这说明 `__nscd_get_mapping` 自身应该保留了帧指针。

```gdb
(gdb) disassemble __nscd_get_mapping
```

这一次，输出结果证实了我们的推断：

```assembly
Dump of assembler code for function __nscd_get_mapping:
   0x0000000000171210 <+0>:	endbr64 
   0x0000000000171214 <+4>:	push   %rbp
   0x0000000000171215 <+5>:	mov    %rsp,%rbp
   ...
```
头两行指令 `push %rbp` 和 `mov %rsp,%rbp` 是设置帧指针的标准操作。这证明了 `__nscd_get_mapping` **确实保留了帧指针**，BCC 正是利用了这一点，才得以成功地从它回溯到上一层。

至此，证据确凿。BCC 的帧指针回溯之旅在成功经过 `__nscd_get_mapping` 之后，到达了 `__nscd_get_map_ref` 这一站。由于 `__nscd_get_map_ref` 函数本身缺少帧指针，导致 BCC 无法定位其调用者的信息，回溯过程被迫中断。我们的猜想得到了验证。

## 使用 DWARF 堆栈回溯

既然基于帧指针的“快捷方式”因为道路中断而无法走通，我们就需要启用一种更可靠的回溯方法。这个方法就是基于 **DWARF** 调试信息。

`DWARF` 是一种被广泛使用的调试信息格式。当程序以 `-g` 等标志编译时，编译器会生成丰富的 `DWARF` 信息，并将其存储在可执行文件中，或者是一个独立的符号文件。这些信息详尽地描述了程序的结构，包括函数、变量、类型，以及如何在任何指令点精确地重建调用堆栈的规则。

与依赖运行时指针链的帧指针回溯不同，`DWARF` 回溯更像是拿着一张由编译器绘制的“代码地图”和“堆栈布局说明书”。性能分析工具（如 `perf`）可以利用这份“地图”，即使在函数省略了帧指针的情况下，也能通过分析当前的指令指针和栈内容，精确地计算出调用者的位置。

要成功使用 `DWARF` 回溯，我们必须满足两个前提条件：

1.  **安装必要的依赖**：`perf` 的 `DWARF` 回溯功能依赖于一些系统库（如 `libunwind`, `libdw`）。我们需要确保这些库已经安装，以保证 `perf` 的核心引擎能够正常工作。

2.  **调试信息必须存在**：被追踪的程序库（在我们的案例中是 `glibc`）必须有对应的调试信息包（在 Ubuntu 上通常是 `dbgsym` 或 `dbg` 包）。这些包包含了 `perf` 解读堆栈所需的 `DWARF` 地图。

满足这两个条件后，我们就可以指示 `perf` 工具使用 `DWARF` 模式来捕获调用堆栈，从而绕过帧指针缺失的问题。

## 还原完整的调用堆栈

现在，来捕获之前中断的完整调用堆栈。

**准备环境**

在开始追踪之前，我们需要确保 `perf` 的 `DWARF` 引擎及其依赖是完整的，并且 `glibc` 的调试信息也已经就绪。

在 Ubuntu 系统上，可以通过以下命令安装所有必要的软件包：

```bash
# 安装 perf 进行 DWARF 回溯所需的核心依赖库
sudo apt install libunwind-dev libdw-dev

# 安装 glibc 的调试符号包
sudo apt install libc6-dbg
```

**使用 `perf` 捕获数据**

环境准备就绪后，我们就可以使用 `perf record` 命令来启动 Java 应用并捕获系统调用事件。这次，我们将明确要求 `perf` 使用 `DWARF` 模式进行堆栈回溯。

```bash
# 放宽内核对 perf 的安全限制，允许追踪用户空间
echo -1 | sudo tee /proc/sys/kernel/perf_event_paranoid

# 使用 --call-graph dwarf 参数启动追踪
sudo perf record -e syscalls:sys_enter_socket --call-graph dwarf -- java Main
```
参数说明：

*   `--call-graph dwarf`：这是最关键的参数。它指示 `perf` 在捕获事件时，必须使用 `DWARF` 信息来回溯并记录完整的调用堆栈。
*   `-- java Main`：`--` 符号告诉 `perf`，后面的所有内容都是要执行的命令。请将其替换为您实际的 Java 应用启动命令。

`perf` 会启动 Java 应用，并从进程开始到结束持续追踪。应用退出后，追踪数据会自动保存在当前目录下的 `perf.data` 文件中。

**分析结果**

最后，我们使用 `perf script` 命令来读取 `perf.data` 文件，并将其中的原始数据翻译成人类可读的调用堆栈。

```bash
sudo perf script
```

这一次，输出的结果将不再有 `[unknown]`。您会看到一份类似下面这样完整、清晰的调用堆栈：

```
java 391724 [003] 952479.264820: syscalls:sys_enter_socket: family: 0x00000001, type: 0x00080801, protocol: 0x00000000
            7fb4c96feb3b __GI_socket+0xb (inlined)
            7fb4c9747c4f open_socket+0x3f (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7fb4c97482cc __nscd_get_mapping+0xbc (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7fb4c974873e __nscd_get_map_ref+0xce (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7fb4c97449c6 nscd_getpw_r+0x66 (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7fb4c9744e45 __nscd_getpwuid_r+0x65 (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7fb4c96c0e3f __getpwuid_r+0x1bf (inlined)
            7fb4c8aaba99 get_user_name+0x59 (/usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so)
            7fb4c8aad1e3 PerfMemory::create_memory_region+0x93 (/usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so)
            7fb4c8aab773 perfMemory_init+0x73 (/usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so)
            7fb4c863b3c6 vm_init_globals+0x26 (/usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so)
            7fb4c8cfa409 Threads::create_vm+0x249 (/usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so)
            7fb4c8709c6d JNI_CreateJavaVM+0x4d (/usr/lib/jvm/java-17-openjdk-amd64/lib/server/libjvm.so)
            7fb4c98050f0 JavaMain+0x90 (/usr/lib/jvm/java-17-openjdk-amd64/lib/libjli.so)
            7fb4c9809a38 ThreadJavaMain+0x8 (/usr/lib/jvm/java-17-openjdk-amd64/lib/libjli.so)
            7fb4c966bac2 start_thread+0x2f2 (/usr/lib/x86_64-linux-gnu/libc.so.6)
            7fb4c96fd84f __clone3+0x2f (inlined)
```

通过与 BCC 的输出进行对比，我们可以清晰地看到，之前中断的堆栈已经被完美地补充完整了。`[unknown]` 的真实身份——`nscd_getpw_r`、`__getpwuid_r` 等函数——以及更上层的 `libjvm.so` 内部调用，现在都已一目了然。

通过 `perf` 捕获的这份完整调用堆栈，为我们揭示了 `Unix socket` 创建的完整路径：

1.  **事件的起点**：在 JVM 启动的极早期阶段（`JNI_CreateJavaVM` -> `Threads::create_vm`），JVM 需要初始化其性能数据子系统（`PerfMemory`）。

2.  **直接原因**：`PerfMemory::create_memory_region` 函数在初始化过程中，调用了其内部的 `get_user_name` 函数，目的是为了获取当前运行用户的名称。这通常是为了创建用户专属的性能数据文件（例如 `/tmp/hsperfdata_<user>/<pid>`）。

3.  **触发系统调用**：为了从用户 ID 解析出用户名，`get_user_name` 函数调用了 `glibc` 提供的标准 POSIX 函数 `__getpwuid_r`。

4.  **NSS 介入**：系统的 NSS(Name Service Switch) 框架接管了这个请求。`glibc` 内部的 `__nscd_*` 系列函数开始工作，准备通过配置好的名称服务守护进程（如 `lwsmd` 或 `nscd`）来完成查询。

5.  **最终结果**：为了与这个守护进程通信，`glibc` 的客户端逻辑最终调用了 `socket()` 系统调用，创建了我们最初观察到的那个 `Unix socket`。

**这个 `Unix socket` 的创建，其根本原因在于 JVM 在初始化性能监控模块时，需要获取当前用户信息。** 

## 总结

本文深入探讨了[上一篇文章]((https://mazhen.tech/p/%E8%BF%BD%E8%B8%AA%E5%AE%9A%E4%BD%8D-java-%E8%BF%9B%E7%A8%8B%E7%9A%84-socket-%E5%88%9B%E5%BB%BA/)) 中遗留的一个问题：为何使用 BCC 追踪 `Unix socket` 创建时，调用堆栈会在 `glibc` 库内部中断并显示 `[unknown]`。

首先分析了 BCC 默认依赖的“帧指针回溯”技术在面对编译优化时的问题。通过 GDB 对 `glibc` 库相关函数的汇编代码进行现场取证，证实了堆栈中断的确是因为 `glibc` 内部某个函数在编译时省略了帧指针。

为了解决这个问题，文章介绍了“DWARF 回溯”方法，最终成功捕获到了从 JVM 内部 C++ 函数到 `glibc` 再到系统调用的完整调用堆栈，彻底解开了 `[unknown]` 背后的秘密。