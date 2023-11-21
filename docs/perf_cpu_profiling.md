
# 深入探索 perf CPU Profiling 实现原理

![title](https://cdn.mazhen.tech/images/202311211430792.png)

## perf 是什么

[perf](https://perf.wiki.kernel.org/index.php/Main_Page) 是由 Linux 官方提供的系统性能分析工具 。我们通常说的 perf 实际上包含两部分：

* **perf** 命令，用户空间的应用程序
* **perf_events** ，Linux 内核中的一个子系统

内核子系统 `perf_events` 提供了性能计数器（[hardware performance counters](https://en.wikipedia.org/wiki/Hardware_performance_counter)）和性能事件的支持，它以事件驱动型的方式工作，通过收集特定事件（如 CPU 时钟周期，缓存未命中等）来跟踪和分析系统性能。`perf_events`是在 [2009 年合并到 Linux 内核源代码中](https://lwn.net/Articles/339361/)，成为内核一个新的子系统。

`perf` 命令是一个用户空间工具，具备 profiling、tracing 和脚本编写等多种功能，是内核子系统 perf_events 的前端工具。通过`perf` 命令可以设置和操作内核子系统 `perf_events`，完成系统性能数据的收集和分析。

虽然 `perf` 命令是一个用户空间的应用程序，但它却位于 Linux 内核源代码树中，在 [tools/perf](https://github.com/torvalds/linux/tree/master/tools/perf) 目录下，它可能是唯一一个被包含在 Linux 内核源码中的复杂用户软件。

perf 和 perf_events 最初支持硬件计数器（performance monitoring counters，**PMC**），后来逐步扩展到支持多种事件源，包括：[tracepoints](https://docs.kernel.org/trace/events.html)、kernel 软件事件、[kprobes](https://www.kernel.org/doc/html/latest/trace/kprobes.html)、[uprobes](https://docs.kernel.org/trace/uprobetracer.html?highlight=uprobe) 和 USDT（User-level statically-defined tracing）。

下图显示了 perf 命令和 perf_events 的关系，以及 perf_events 支持的事件源。

![perf events](https://cdn.mazhen.tech/images/202311071635405.png)

## perf 事件源

perf 支持来自硬件和软件方面的各种事件。硬件事件来自芯片组中的硬件性能计数器（[hardware performance counters](https://en.wikipedia.org/wiki/Hardware_performance_counter)），而软件事件则由[tracepoints](https://docs.kernel.org/trace/events.html)、[kprobe](https://www.kernel.org/doc/html/latest/trace/kprobes.html) 和 [uprobe](https://docs.kernel.org/trace/uprobetracer.html?highlight=uprobe) 等调试设施提供。

可以使用 perf 的子命令 **list** 列出当前可用的所有事件：

```bash
$ sudo perf list

List of pre-defined events (to be used in -e or -M):

  branch-instructions OR branches                    [Hardware event]
  branch-misses                                      [Hardware event]
  bus-cycles                                         [Hardware event]
  cache-misses                                       [Hardware event]
  cache-references                                   [Hardware event]
  cpu-cycles OR cycles                               [Hardware event]
[...]
  cgroup-switches                                    [Software event]
  context-switches OR cs                             [Software event]
  cpu-clock                                          [Software event]
[...]
  L1-dcache-load-misses                              [Hardware cache event]
  L1-dcache-loads                                    [Hardware cache event]
[...]
  branch-instructions OR cpu/branch-instructions/    [Kernel PMU event]
  branch-misses OR cpu/branch-misses/                [Kernel PMU event]
[...]
  sched:sched_process_exec                           [Tracepoint event]
  sched:sched_process_exit                           [Tracepoint event]
  sched:sched_process_fork                           [Tracepoint event]
[...]
```

事件类型如下：

* **Hardware Event**：CPU 性能计数器（performance monitoring counters）
* **Software event**：内核计数器事件
* **Hardware cache event**：CPU Cache 事件
* **Kernel PMU event**：Performance Monitoring Unit (PMU) 事件
* **Tracepoint event**：包含了静态和动态代码追踪事件
  * **Kernel tracepoints**：在 kernel 中关键位置的静态追踪代码
  * **kprobes**：在内核中的任意位置动态地被插入追踪代码
  * **uprobes**：与kprobes类似，但用于用户空间。动态地在应用程序和库中的任意位置插入追踪代码
  * **USDT**：是 tracepoints 在用户空间的对应技术，是应用程序和库在它们的代码中提前加入的静态追踪代码

perf 的 “Tracepoint event” 事件源很容易引起混淆，因为除了内核的 tracepoints，基于 kprobe、uprobe 和 USDT 的跟踪事件也被标记为了“Tracepoint event”。默认情况下它们不会出现在 perf list 的输出中， 必须先初始化才会作为 "Tracepoint event" 中的事件。

内核 tracepoints 是由 [TRACE_EVENT](https://elixir.bootlin.com/linux/latest/source/include/trace/trace_events.h#L39) 宏定义。TRACE_EVENT 自动生成静态追踪代码，定义并格式化其参数，并将跟踪事件放入 tracefs （/sys/kernel/debug/tracing）和 [perf_event_open](https://www.man7.org/linux/man-pages/man2/perf_event_open.2.html) 接口。

与其他性能分析工具相比，perf 特别适合 CPU 分析，它能对运行在 CPU 上代码调用栈（stack traces）进行采样，以确定程序在 CPU 上的运行情况，识别和优化代码中的热点。这种 CPU Profiling 能力是基于硬件计数器 (performance monitoring counters，PMC) 实现的，而 PMC 被内核子系统 `perf_events`  包装成了 **Hardware Event**，下面重点介绍。

## Hardware Event

CPU 和其他硬件设备通常提供用于观测性能数据的 PMC。简单来说，**PMC** 就是 CPU 上的**可编程寄存器**，可通过编程对特定硬件事件进行计数。通过 PMC 可以监控和计算 CPU 内部各种事件，比如 CPU 指令的执行效率、CPU caches 的命中率、分支预测的成功率等 micro-architectural 级的性能信息。利用这些数据分析性能，可以实现各种性能优化。

perf 命令通过 [perf_event_open(2)](https://www.man7.org/linux/man-pages/man2/perf_event_open.2.html) 系统调用访问 PMC，配置想要捕获的硬件事件。PMC 可以在两种模式下使用：

* Counting（计数模式），只计算和报告硬件事件的总数，开销几乎为零。
* Sampling（采样模式），当发生一定数量的事件后，会触发一个中断，以便捕获系统的状态信息。可用于采集代码路径。

glibc 没有提供对系统调用 perf_event_open 的包装，perf 在 [tools/perf/perf-sys.h](https://elixir.bootlin.com/linux/v6.6.2/source/tools/perf/perf-sys.h) 定义了自己的包装函数：

```c
static inline int
sys_perf_event_open(struct perf_event_attr *attr,
        pid_t pid, int cpu, int group_fd,
        unsigned long flags)
{
 return syscall(__NR_perf_event_open, attr, pid, cpu,
         group_fd, flags);
}
```

第一个参数 [perf_event_attr](https://elixir.bootlin.com/linux/v6.6.2/source/include/uapi/linux/perf_event.h#L385) 结构体定义了想要监控的事件的类型和行为。例如想要统计 CPU 的总指令数，可以这样初始化 `perf_event_attr` 结构体：

```c
struct perf_event_attr  pe;

memset(&pe, 0, sizeof(pe));
pe.type = PERF_TYPE_HARDWARE;
pe.size = sizeof(pe);
pe.config = PERF_COUNT_HW_INSTRUCTIONS;
pe.disabled = 1;
pe.exclude_kernel = 1;
pe.exclude_hv = 1;
```

perf 在 [perf_event.h](https://elixir.bootlin.com/linux/v6.2.16/source/include/uapi/linux/perf_event.h) 中定义了适用于各类处理器的通用事件：

```c
enum perf_hw_id {
 PERF_COUNT_HW_CPU_CYCLES  = 0,
 PERF_COUNT_HW_INSTRUCTIONS  = 1,
 PERF_COUNT_HW_CACHE_REFERENCES  = 2,
  ...
}
enum perf_hw_cache_id {
 PERF_COUNT_HW_CACHE_L1D   = 0,
 PERF_COUNT_HW_CACHE_L1I   = 1,
 PERF_COUNT_HW_CACHE_LL   = 2,
  ...
}
...
```

针对每种 CPU，需要将事件枚举类型映射为特定 CPU 的原始硬件事件描述符。例如对于 Intel x86 架构的 CPU，**PERF_COUNT_HW_INSTRUCTIONS** 事件映射为 `0x00c0`，在 [arch/x86/events/intel/core.c](https://elixir.bootlin.com/linux/v6.6.2/source/arch/x86/events/intel/core.c#L31) 中定义：

```c
static u64 intel_perfmon_event_map[PERF_COUNT_HW_MAX] __read_mostly =
{
 [PERF_COUNT_HW_CPU_CYCLES]  = 0x003c,
 [PERF_COUNT_HW_INSTRUCTIONS]  = 0x00c0,
 [PERF_COUNT_HW_CACHE_REFERENCES] = 0x4f2e,
  ...
}
```

这些原始硬件事件描述符在相应的处理器软件开发人员手册中进行了说明。内核开发人员根据 CPU 厂商提供的软件开发人员手册，完成 PMC 事件和特定 CPU 代码的映射。

例如 Intel x86 架构的 CPU可以参考 [Intel® 64 and IA-32 Architectures Software Developer’s Manual](https://www.intel.com/content/www/us/en/content-details/782158/intel-64-and-ia-32-architectures-software-developer-s-manual-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html) 的第三卷第 20 章 “PERFORMANCE MONITORING”。Intel 还提供了一个网站 [perfmon-events.intel.com](https://perfmon-events.intel.com/)，可以查询 CPU 的所有 PMC 事件。

我们在使用 perf 时，可以直接指定特定 CPU 的原始事件代码：

```bash
sudo perf stat -e r00c0 -e instructions -a sleep 1
```

对于 Intel CPU，`instructions` 事件的原始事件代码为 `r00c0`，两者可以等价使用。

对于大部分通用事件，我们不需要记住这些原始事件代码，perf 都提供了可读的映射。但在某些情况下，例如最新 CPU 增加的事件还没有在 perf 中添加映射，或者某种 CPU 的特定事件不会通过可读的名称暴露出来，这时就只能通过指定原始硬件事件代码来监控事件。

简单了解了 PMC 后，我们来看如何基于 PMC 事件进行 CPU Profiling。

## CPU Profiling

perf 是事件驱动的方式工作，通过 **-e** 参数指定想要收集的特定事件，例如：

```bash
sudo perf stat -e LLC-loads,LLC-load-misses,LLC-stores,LLC-prefetches ls
```

我们在对整个系统的 CPU 进行30 秒的采样时，使用的命令如下：

```bash
sudo perf record -F 99 -a -g -- sleep 30
```

这里并未明确指定事件（没有 -e 参数），perf 将默认使用以下预定义事件中第一个可用的：

1. cycles:ppp
2. cycles:pp
3. cycles:p
4. cycles
5. cpu-clock

前四个事件都是 PMC 提供的 CPU **cycles** 事件，区别在于精确度不同，从最精确（`ppp`）到无精确设置（没有 `p`），最精确的事件优先被选择。**cpu-clock** 是基于软件的 CPU 频率采样，在没有硬件 **cycles** 事件可用时，会选择使用 **cpu-clock** 软件事件。

那么什么是CPU **cycles** 事件，为什么对 **cycles** 事件进行采样可以分析 CPU 的性能？

### cycles 事件

首先介绍一个关于 CPU 性能的重要概念，**Clock Rate**（时钟频率）。Clock（时钟）是驱动 CPU 的数字信号，CPU 以特定的时钟频率执行，例如 4 GHz的 CPU 每秒可执行40亿个 cycles（周期）。

CPU **cycles （周期）**是 CPU 执行指令的时间单位，而**时钟频率**表示 CPU 每秒执行的 CPU 周期数。每个 CPU 指令执行可能需要一个或多个 CPU 周期。 通常情况下，更高的时钟频率意味着更快的CPU，因为它允许 CPU 在单位时间内执行更多的指令。

![CPU cycles](https://cdn.mazhen.tech/images/202311091031747.jpg)

每经过一个 CPU 周期都会触发一个 **cycles** 事件。可以认为，**cycles** 事件是均匀的分布在程序的执行期间。这样，以固定频率去采样的 **cycles** 事件，也是均匀的分布在程序的执行期间。我们在采样 **cycles** 事件时，记录 CPU 正在干什么，持续一段时间收集到多个采样后，我们就能基于这些信息分析程序的行为，多次出现的同样动作，可以认为是程序的热点，成为下一步分析重点关注的方面。

因为 **cycles** 事件的均匀分布，通过以固定频率采样 **cycles** 事件获得的信息，我们就能进行 CPU 性能分析。那么如何指定采样频率呢？

### 设置采样频率

在使用 perf record 记录 PMC 事件时，会使用一个默认的采样频率，不是每个事件都会被记录。例如记录 cycles 事件：

```bash
$ perf record -vve cycles -a sleep 1
Using CPUID GenuineIntel-6-45-1
DEBUGINFOD_URLS=
nr_cblocks: 0
affinity: SYS
mmap flush: 1
comp level: 0
------------------------------------------------------------
perf_event_attr:
  size                             128
  { sample_period, sample_freq }   4000
  sample_type                      IP|TID|TIME|ID|CPU|PERIOD
  read_format                      ID|LOST
  disabled                         1
  inherit                          1
  freq                             1
  sample_id_all                    1
  exclude_guest                    1
...
[ perf record: Captured and wrote 0.422 MB perf.data (297 samples) ]
```

加了 **-vv** 选项可以输出更详细的信息。从输出中可以看出，即使我们没有明确设置采样频率，采样频率已经启用（`freq 1`），并且采样频率为 4000 （`{ sample_period, sample_freq }   4000`），即每 CPU 每秒采集约 4000个事件。cycles 事件每秒中有几十亿次，默认采样频率的设置很合理，否则记录事件的开销过高。

可以使用 **-F** 选项明确设置事件采样频率，例如：

```bash
perf record -F 99 -e cycles -a sleep 1
```

**-F 99** 设置采样频率为 99 Hertz，即每秒进行 99 次采样。[Brendan Gregg](https://www.brendangregg.com/index.html) 在大量的例子中都使用了 99 Hertz 这个采样频率，至于为什么这样设置，他在文章 [perf Examples](https://www.brendangregg.com/perf.html) 中给出了解释，大意是：选择 99 Hertz 而不是100 Hertz，是为了避免意外地与一些周期性活动同步，这会产生偏差的结果。也就是说，如果程序中有周期性的定时任务，例如每秒钟执行的任务，以 100 Hertz 频率进行采样，那么每次周期性任务运行时都会被采样，这样产生的结果“放大”了周期性任务的影响，偏离了程序正常的行为模式。

`perf record` 命令还可以使用 **-c** 选项来设置采样事件的周期，这个周期代表了采样事件之间的间隔。例如：

```bash
sudo perf record -c 1000 -e cycles -a sleep 1
```

在这个示例中，**-c** 选项设置采样周期为 1000，即每隔 1000 次事件进行一次采样。

现在我们知道了如何以固定的频率对 cycles 事件进行采样，那么如何获知在采样时，CPU 正在干什么呢？

## 背景知识

要知道  cycles 事件发生时 CPU 正在干什么，我们需要了解一些硬件知识，以及内核与硬件是如何配合工作的。先看看 CPU 是如何执行指令的。

### CPU 执行指令

CPU 内部有多种不同功能的寄存器，涉及到指令执行的，有三个重要的寄存器：

* PC 寄存器（**PC**，Program Counter），存放下一条指令的内存地址
* 指令寄存器（**IR**，Instruction Register），存放当前正在执行的指令
* 状态寄存器（**SR**，Status Register），用于存储 CPU 当前的状态，如条件标志位、中断禁止位、处理器模式标志等

CPU 还有其他用于存储数据和内存地址的寄存器，根据存放内容命名，如整数寄存器、浮点数寄存器、向量寄存器和地址寄存器等。有些寄存器既可以存放数据，又可以存放地址，被称为通用寄存器（**GR**，General register）。

![cpu registers](https://cdn.mazhen.tech/images/202311091736945.png)

程序执行时，CPU 根据 **PC** 寄存器中的地址从内存中读取指令到 **IR** 寄存器中执行，并根据指令长度自增，加载下一条指令。

只要我们在采样时获取CPU 的 **PC** 寄存器和 **IR** 寄存器的内容，就能推断出 CPU 当时正在干什么。

在 x86-64 架构中，Program Counter 的功能是由  **RIP (Instruction Pointer Register)** 寄存器实现的。

在编译程序时，可以让编译器生成一个映射，将源代码行与生成的机器指令关联起来，这个映射通常存储在 [DWARF 格式（Debugging With Attributed Record Formats）](https://dwarfstd.org/)的调试信息中。同时编译时会生成**符号表（Symbol Table）**，其中包含了程序中各种符号（如函数名、变量名）及其地址的映射。perf 借助调试信息和符号表（symbol table），可以将采样时寄存器中的指令地址转换为对应的函数名、源代码行号等信息。

知道了 CPU 当时的“动作”还不够，我们还需要知道 CPU 是怎么做这个“动作”的，也就是代码的执行路径。下面介绍函数调用栈的相关概念。

### 还原函数调用栈

函数是软件中的一个关键抽象概念，它让开发者将具有特定功能的代码打包，然后这个功能可以在程序的多个位置被调用。

假设函数 P 调用函数 Q，然后 Q 执行并返回结果给 P，这个过程涉及到以下机制：

* **传递控制**：在进入函数 Q 时，**PC** 寄存器设置为 Q 的起始地址；在从 Q 返回时，**PC** 寄存器设置为 P 中调用 Q 后的下一条指令处。
* **传递数据**：P 能够向 Q 提供一个或多个参数，Q 也能够将一个值返回给 P。
* **分配和释放内存**：Q 需要在开始时为局部变量分配空间，然后在返回前释放该存储空间。

![stack & code](https://cdn.mazhen.tech/images/202311141521021.png)

x86-64 平台上的程序使用**堆栈（Stack）**来实现函数调用。**堆栈（Stack）**的特性是**后进先出（LIFO）**，函数调用正是利用了这一特性。调用某个函数就是在堆栈上为这个函数分配所需的内存空间，这部分空间被称为**栈帧（stack frame）**，从函数返回，就是将这个函数的**栈帧（stack frame）**从堆栈中弹出，释放空间。多个**栈帧（stack frame）**组成 [Call stack](https://en.wikipedia.org/wiki/Call_stack)，体现出了函数的调用关系。

注意：编译后的程序存储在**代码段**，是静态的；而 [Call stack](https://en.wikipedia.org/wiki/Call_stack) 是动态的，反应了程序运行时的状态。

下面以示例程序为例，x86-64 平台上如何利用**堆栈（Stack）**实现函数调用的。

![stack](https://cdn.mazhen.tech/images/202311141524326.png)

`main`函数有三个局部变量`a`、`b`和 `res` 存储在自己的 **stack frame** 中。当 `main` 调用 `Calc` 函数时，会先将参数 `i`和`j` 压入 Stack，然后将 **PC** 寄存器中的值也压入 Stack。我们知道，**PC** 寄存器存放的是下一条指令的地址，这时 **PC** 寄存器中的值是函数调用指令（call）后紧跟着的那条指令的地址。把 **PC** 寄存器压入 Stack，相当于保留了函数调用结束后要执行的指令地址，这样`Calc`完成后程序知道从哪里继续执行。

**rbp** 是栈基址寄存器（register base pointer ），又叫栈帧指针（**Frame Pointer**），存放了当前 **stack frame** 的起始地址。**rsp** 是栈顶寄存器（register stack pointer），称为栈指针（**Stack Pointer**），随着入栈出栈动作而移动，始终指向栈顶元素。**x86-64** 的堆栈是**从高地址向低地址**增长的。

在为 `calc`  新建 **stack frame** 时，会先将 **rbp** 寄存器压入 Stack。当前 **rbp** 寄存器中存放的是 `main`  **stack frame** 的起始地址，将  **rbp** 寄存器压入 Stack，也就是将 `main`  **stack frame** 的起始地址压入了 Stack。

随后把 **rsp** 的值复制到 **rbp**，因为 **rsp** 始终会指向栈顶，把 **rsp** 的值复制到 **rbp** 就是让  **rbp** 寄存器指向当前位置，即 `calc` **stack frame** 的起始位置。

注意 `Calc` 的参数和返回地址包含在 `main` 的 **stack frame** 中，因为它们保存了与 `main` 相关的状态。`Calc` 函数局部变量`sum`和`result` 被分配在自己的 **stack frame** 上。

在 `Calc` 调用 `Sum` 时，重复上面的动作：参数和返回地址入栈，保存并更新 **rbp** 寄存器，为 `Sum` 的局部变量分配地址。

在函数 `Sum` 执行完成之后，会将之前保存的  **rbp** 出栈，恢复到 **rbp** 中，也就是让 **rbp** 指向 `Calc`  **stack frame** 的起始地址，将 `Sum`的  **stack frame** 弹出了 Stack，释放了 `Sum` 占用的空间。然后将返回地址出栈，更新到 **PC** 寄存器中。返回地址是函数调用指令（call）后的下一条指令，即 `Calc` 调用完 `Sum` 后紧跟着的下一条指令，把这个指令的地址恢复到 **PC** 寄存器中，实际上是将控制权返回给了 `Calc` ，让 `Calc` 剩余部分接着执行。

 `Calc` 执行完也会做同样的出栈动作，释放 **stack frame** ，将控制权返回给 `main`。

这样，函数调用利用了**堆栈（Stack）**传递参数，存储返回信息，保存寄存器中的值，以及存储函数的局部变量，来实现函数调用。

每个函数的活动记录对应一个**栈帧（stack frame）**，多个**栈帧（stack frame）**叠加在一起构成**调用栈**（ [Call stack](https://en.wikipedia.org/wiki/Call_stack)）。**Frame Pointer**（通常是 **rbp** 寄存器）指向当前激活的函数的**栈帧（stack frame）**的起始处，这个起始处保存了调用它的函数的**栈帧（stack frame）**的起始地址。通过这种链接，我们就能以 **Frame Pointer** 为起点，追溯整个调用链，即从当前函数开始，逐级访问到每个调用者的**栈帧（stack frame）**，从而重构出程序执行的路径。

需要注意的是，出于空间和时间效率的考虑，程序都会优先使用通用寄存器来传递参数，只有在寄存器不够用的时候才会将多出的参数压入栈中。

perf 正是利用 **Frame Pointer**，还原采样时的代码执行路径。

在开启编译器优化的情况下，程序会将 **rbp** 寄存器作为通用寄存器重新使用，这时就不能再使用 **Frame Pointer** 还原函数调用栈。perf 还可以使用其他方法进行 stack walking：

* **--call-graph dwarf** ：使用调试信息
* **--call-graph lbr**： 使用 Intel 的 last branch record (LBR)
* **--call-graph fp**：使用 **Frame Pointer** ，缺省方法

本文就不详细讨论其他两种还原调用栈的方法，感兴趣的可以参考[《BPF Performance Tools》](https://www.brendangregg.com/bpf-performance-tools-book.html)。

在 Linux 上，进程的执行分为了用户态和内核态，要知道完整的代码执行路径，就需要分别还原用户栈和内核栈。什么是用户态和内核态呢？

### 用户态和内核态

操作系统需要能够限制对关键系统资源的访问，提供对资源不同级别的访问权限，这样可以保护系统免受错误和恶意行为的侵害。这种访问权限由 CPU 在硬件级别上实现，例如 x86 架构定义了特权级别，也称为保护环（**protection rings**），从 **Ring 0** 到 **Ring 3** ，每个级别定义了可使用的指令集和可访问的资源，**Ring 0** 具有最高的特权级别。

![protection rings](https://cdn.mazhen.tech/images/202311151103732.png)

* **Ring 0**: 最高特权级别，用于操作系统的内核，可以直接访问所有的硬件和系统资源。
* **Ring 1 和 Ring 2**: 这些中间层次的环通常用于特定的系统任务，如设备驱动程序，但在现代操作系统中，这些任务通常也在 Ring 0 执行。
* **Ring 3**: 最低特权级别，用于普通的应用程序，这些应用程序不能直接执行影响系统稳定性或安全性的操作。

Linux 主要使用了 **Ring 0** 和 **Ring 3**，将能够访问关键资源的内核放在 Ring0，称为内核态（**Kernel Mode**），普通的应用程序将放在 Ring3，称为用户态（**User Mode**）。

### 系统调用

如果用户态代码需要访问核心资源，它必须通过**系统调用（system call ）**。系统调用是进入内核的入口点，内核通过系统调用向程序提供一系列服务，如文件读写、进程创建、输入输出操作等。应用程序通过系统调用请求内核代为执行这些服务。

调用系统调用看起来很像调用 C 函数。但实际上，在系统调用的执行过程中，会进行多个步骤。以 x86 平台的实现为例，包括以下几个关键环节：

![system call](https://cdn.mazhen.tech/images/202311211416813.png)

1. 应用程序通过调用 C 库中的包装函数来发起系统调用。
2. 包装函数负责将所有系统调用参数传递给内核。这些参数通常通过栈传递给包装函数，然后被复制到特定的 CPU 寄存器中。
3. 为了让内核识别不同的系统调用，包装函数会把系统调用的编号复制到一个特定的 CPU 寄存器（%eax）中。
4. 包装函数执行 **trap** 机器指令（ x86 架构 `sysenter` 指令），使 CPU 从用户模式切换到内核模式。
5. 内核响应中断，把当前的寄存器值保存到内核栈数据结构 **struct pt_regs** 中，根据编号在一个表格中找到相应的系统调用服务程序，并执行它，然后返回结果。
6. 完成操作后，内核将寄存器值恢复到原始状态，并将控制权返回给用户空间的应用程序，同时返回系统调用的结果。

### 用户栈和内核栈

可以看出在执行系统调用时，进程具有两个栈：**用户栈（User Stack）**和**内核栈（Kernel Stack）**。

* **用户栈（User Stack）** 保留了进入系统调用前的状态，用户栈在系统调用期间不会改变
* **内核栈（Kernel Stack）** 是在系统调用期间使用，用于存储在内核态下执行的状态信息，包括寄存器的值和系统调用的参数。此外处理中断和异常时，也会使用内核栈。

用户栈和内核栈在什么什么位置？我们需要先了解虚拟地址空间的概念。

### 进程虚拟地址空间

在现代操作系统上，用户程序都不能直接操作物理内存。操作系统会给进程分配虚拟内存空间，所有进程看到的这个地址都是一样的，里面的内存都是从 0 开始编号。

程序里指令操作的都是虚拟地址。内核会维护一个虚拟内存到物理内存的映射表，将不同进程的虚拟地址和不同的物理地址映射起来。当程序要访问虚拟地址时，会通过映射表进行转换，找到对应的物理内存地址。不同进程相同的虚拟地址，会映射到不同的物理地址，不会发生冲突。这样每个进程的地址空间都是独立的，相互隔离互不影响。

![Virtual Memory Address](https://cdn.mazhen.tech/images/202311151758027.png)

我们来看一下进程的虚拟地址空间布局。

一个进程的虚拟地址空间分为两个部分，一部分是用户态地址空间，一部分是内核态地址空间。

用户空间是应用程序执行的场所，每个进程都有自己独立的用户空间，其布局包含代码、全局变量、堆、栈和内存映射区域等多种部分。

内核空间是内核代码运行的内存区域，它并非专属于某个单独的进程，所有进程通过系统调用进入到内核之后，看到的虚拟地址空间都是一样的。

用户空间与内核空间的这种分离，确保了用户应用程序不能直接干扰内核，保证了系统的安全稳定性。

![process virtual addrsss space](https://cdn.mazhen.tech/images/202311161738057.png)

Linux 的可执行文件是 [ELF（Executable and Linkable Format）](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)格式，执行时从硬盘加载到内存，ELF 文件中的代码段和数据段被直接映射到进程虚拟地址空间用户态的数据段和代码段。

用户空间的**堆（heap）**是动态内存分配的区域，可以使用系统调用 [sbrk](https://www.man7.org/linux/man-pages/man2/brk.2.html) 、[mmap](https://www.man7.org/linux/man-pages/man2/mmap.2.html) 和 glibc 提供的 [malloc](https://elixir.bootlin.com/glibc/glibc-2.38/source/malloc/malloc.c) 函数进行堆内存的申请。内核会维护一个变量 **brk** 指向堆的顶部，**sbrk** 通过改变 **brk** 来维护堆的大小。**malloc** 内部也使用了系统调用 **sbrk** 或 **mmap**，但 glibc 会维护一个内存池，并不是每次使用 **malloc** 申请内存时都直接进行系统调用。

内存映射区域为共享库及文件映射提供空间。可以使用系统调用 [mmap](https://www.man7.org/linux/man-pages/man2/mmap.2.html) 将创建文件映射提升 IO 效率。

用户空间的**堆栈（Stack）** 是用户态函数执行的活跃记录，`%rsp`指向当前堆栈顶部。

内核空间也有代码段和数据段，映射内核的代码段和数据段。

当进程执行系统调用时，会从用户空间切换到内核空间，进程的当前状态，包括栈指针（**rsp** 寄存器）、程序计数器（**rip**，也就是 **PC** 寄存器）等，会被保存在内核数据结构 [struct pt_regs](https://elixir.bootlin.com/linux/v6.6.1/source/arch/x86/include/asm/ptrace.h#L59) 内，以便在系统调用完成后能够准确地恢复，继续执行因系统调用而暂停的用户空间操作。

执行内核代码会使用**内核栈（Kernel-Stack）**。

现在有了进程虚拟地址空间的全景图，我们再回头看看还原函数调用栈的问题。

### 还原完整调用栈

在 Linux 系统中，我们可以说在任何给定的时刻，CPU 处于下面三种状态之一：

1. 在用户空间，执行某个进程里的用户级代码
2. 在内核空间，以进程的身份运行，为特定的进程服务，也就是执行系统调用
3. 在内核空间，处于处理中断的状态，此时不与任何进程相关联，运行在内核线程中，专注于处理中断事件

对于第一、第三种情况， perf 在采样事件触发时，只要通过 **Frame Pointer**（ **rsp** 寄存器）就可以还原用户栈或内核栈，并且已经是完整的调用栈。

对于第二种情况，进程正好陷入内核执行系统调用，那么通过 **Frame Pointer**（ **rsp** 寄存器）可以还原内核代码的执行路径。然后再通过内核数据结构 [struct pt_regs](https://elixir.bootlin.com/linux/v6.6.1/source/arch/x86/include/asm/ptrace.h#L59) 内保存的寄存器状态，还原进入内核前用户空间的代码执行路径。这样我们就能获得采样事件触发时，包含了用户态和内核态的完整代码执行路径。

通过 **PC** 寄存器、 **rsp** 寄存器，以及内核数据结构**pt_regs**，我们能知道 CPU 瞬时的“动作”，以及它是怎么做这个动作的（代码执行路径），那么还剩最后一个问题，我们怎么知道采样发生时刻 CPU 寄存器的内容呢？

### 中断处理

内核是被动工作模式，它“躺”在内核空间不会主动工作，要么通过系统调用让它为用户进程服务，要么由时钟中断和各种外部设备中断事件驱动执行。

中断是 Linux 的核心功能之一，它允许 CPU 响应外部或内部事件，如源自硬件设备的键盘、鼠标、网卡，或来自软件的异常。当这些事件，也就是中断事件发生时，CPU 会暂停当前的工作，转入内核的**中断处理程序（Interrupt Handler）**。当处理完成后，会从中断处理程序返回到原来被中断的代码处继续执行。

中断可以分为**同步（Synchronous）**中断和**异步（Asynchronous）**中断两类：

* **同步中断**：由CPU当前执行的指令序列引起的。
* **异步中断**：由外部事件（定时中断和 I/O 设备）引起，与CPU当前执行的指令序列无关。

在 x86 平台上，同步中断被称为**异常（Exception）**，而异步中断被称为**中断（Interrupt）**。注意“中断”这个词根据上下文，可以仅指异步中断，也可以指包含了异常的两类中断的总称。

![interrupts](https://cdn.mazhen.tech/images/202311201051594.png)

在 x86 架构中，每个中断或异常都通过一个 0 到 255 范围内的数字来识别，这个数字是一个 8 位的无符号数，被称为“**向量（*vector*）**”。其中，异常和不可屏蔽中断的向量值是固定不变的（0～31），而可屏蔽中断的向量可以通过**可编程中断控制器**（*Programmable Interrupt Controller*，**PIC**）进行调整。

每个 I/O 设备通常有一个单独的输出线路，用来发送中断请求（**I**nterrupt **R**e**Q**uest, **IRQ**）。所有 **IRQ** 线路都连接到 **PIC**，然后 **PIC** 又连接到 CPU 的 **INTR** 引脚。当某个 I/O 设备发生了需要 CPU 注意的事件，例如用户敲击键盘，数据到达网卡，该设备在相应的 IRQ 线路上发送信号，**PIC** 将这个 **IRQ** 信号转换成一个**中断向量**，然后在 CPU 的 **INTR** 引脚上发起一个中断请求，等待 CPU 处理。

不可屏蔽中断通过 **NMI** 引脚接入 CPU。

![PIC](https://cdn.mazhen.tech/images/202311201445225.png)

我们可以在 **/proc/interrupts** 查看到系统硬件设备 **IRQ** 线路及对应 CPU 的统计信息：

```bash
$ cat /proc/interrupts 
           CPU0       CPU1       CPU2       CPU3       
  8:          0          0          0          0  IR-IO-APIC   8-edge      rtc0
  9:          0          4          0          0  IR-IO-APIC   9-fasteoi   acpi
 18:          0          2          0          0  IR-IO-APIC  18-fasteoi   i801_smbus
 23:         35          0          0          0  IR-IO-APIC  23-fasteoi   ehci_hcd:usb1
 40:          0          0          0          0  DMAR-MSI   0-edge      dmar0
 41:          0          0          0          0  DMAR-MSI   1-edge      dmar1
 42:          0          0          0          0  IR-PCI-MSI-0000:00:1c.0   0-edge      PCIe PME
...
NMI:         67        124         61         60   Non-maskable interrupts
LOC:    7386692    9261862    8162396    7051922   Local timer interrupts
...
PMI:         67        124         61         60   Performance monitoring interrupts
...
```

注意第一列输出的是 **IRQ** 线路编号，需要通过 **PIC** 转换为 CPU 使用的**中断向量**，对于 Intel CPU，IRQn 转换为的中断向量是 **n+32**，因为 0～31 是固定给了异常使用。

×86 系列的 CPU 能处理 20 种不同类型的异常，内核必须为每一种异常都提供一个专门的异常处理程序。我们可以在 [Intel's Software Development Manual: System Programming Guide](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) 中找到对这些异常的完整描述。

| Vector | Mnemonic | Description                                 | Type        | Error Code | Source                                                       |
| ------ | -------- | ------------------------------------------- | ----------- | ---------- | ------------------------------------------------------------ |
| 0      | \#DE     | Divide Error                                | Fault       | No         | DIV and IDIV instructions.                                   |
| 1      | \#DB     | Debug Exception                             | Fault/ Trap | No         | Instruction, data, and I/O breakpoints; single-step; and others. |
| 2      | -        | NMI Interrupt                               | Interrupt   | No         | Nonmaskable external interrupt.                              |
| 3      | \#BP     | Breakpoint                                  | Trap        | No         | INT3 instruction.                                            |
| 4      | \#OF     | Overflow                                    | Trap        | No         | INTO instruction.                                            |
| 5      | #BR      | BOUND Range  Exceeded                       | Fault       | No         | BOUND instruction.                                           |
| 6      | #UD      | Invalid Opcode (Undefined Opcode)           | Fault       | No         | UD instruction or reserved opcode.                           |
| 7      | #NM      | Device Not Available (No Math  Coprocessor) | Fault       | No         | Floating-point or WAIT/FWAIT instruction.                    |
| 8      | #DF      | Double Fault                                | Abort       | Yes (zero) | Any instruction that can generate an  exception, an NMI, or an INTR. |
| 9      |          | Coprocessor Segment Overrun (reserved)      | Fault       | No         | Floating-point instruction.                                  |
| 10     | #TS      | Invalid TSS                                 | Fault       | Yes        | Task switch or TSS access.                                   |
| 11     | #NP      | Segment Not Present                         | Fault       | Yes        | Loading segment registers or accessing  system segments.     |
| 12     | #SS      | Stack-Segment Fault                         | Fault       | Yes        | Stack operations and SS register loads.                      |
| 13     | \#GP     | General Protection                          | Fault       | Yes        | Any memory reference and other protection checks.            |
| 14     | \#PF     | Page Fault                                  | Fault       | Yes        | Any memory reference.                                        |
| 15     | —        | (Intel reserved. Do  not use.)              |             | No         |                                                              |
| 16     | #MF      | x87 FPU Floating-Point Error (Math Fault)   | Fault       | No         | x87 FPU  floating-point or WAIT/FWAIT instruction.           |
| 17     | #AC      | Alignment Check                             | Fault       | Yes (Zero) | Any data reference in memory.                                |
| 18     | #MC      | Machine Check                               | Abort       | No         | Error codes (if any) and source are model  dependent.        |
| 19     | #XM      | SIMD Floating-Point Exception               | Fault       | No         | SSE/SSE2/SSE3 floating-point instructions                    |
| 20     | #VE      | Virtualization Exception                    | Fault       | No         | EPT violations                                               |
| 21     | #CP      | Control Protection Exception                | Fault       | Yes        | RET, IRET, RSTORSSP, and SETSSBSY  instructions can generate this exception. When CET indirect branch tracking  is enabled, this exception can be generated due to a missing ENDBRANCH  instruction at target of an indirect call or jump. |
| 22-31  | -        | Intel reserved. Do not use.                 |             |            |                                                              |

下图描述了中断处理的大致流程：

![Interrupt Handler](https://cdn.mazhen.tech/images/202311201727048.png)

异常是由 CPU 执行的指令产生，中断是由外部设备的紧急事件产生。

CPU 在收到中断请求后，根据中断号在**中断向量表**中找到相应的**中断处理程序（Interrupt Handler）**入口，从而做出相应的处理。**中断向量表**将每个中断或异常与对应的中断处理程序关联了起来。

从上图可以看出，中断向量表的 0～31固定为异常处理，32～127 和 129～238 项用来处理外部 I/O 设备的请求。 **PIC** 会将 **IRQ** 编号转换为**中断向量**。

当系统收到中断请求时，如果不在内核态，先会从用户态切换到内核态，并在内核栈中保存当前的状态信息（主要是寄存器信息）。

接着使用中断向量号，在中断向量表中查找对应的处理代码的入口地址。然后系统跳转到这个地址，执行相应的中断处理程序。

处理完成后，系统通过中断返回机制恢复被中断任务的现场（内核态或用户态），并继续执行原来的代码。

中断处理的一个关键步骤是，**保留中断发生时的现场信息**，perf 的 CPU 采样功能正是利用了这一点。

我们使用 `perf record` 进行 CPU 分析时，会通过 **-F** 指定采样频率。当达到预设的阈值（如一定数量的指令执行或特定时间间隔），硬件性能计数器会触发一个 **PMI** （Performance monitoring interrupts）中断。

**PMI** 是个什么类型的中断呢？一般 **PMI** 是由本地 **APIC**（ *Advanced* PIC）产生，而 APIC 接入 CPU 的 **INTR** 引脚，你可能觉得它是一个可屏蔽中断。但根据[Intel's Software Development Manual: System Programming Guide](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html) ，**非屏蔽中断 (NMI)** 可通过两种方式生成：

* 外部硬件激活 CPU 的 **NMI** 引脚
* CPU 通过系统总线或 APIC 串行总线接收一条包含 **NMI** 传递模式的消息

也就是说，APIC 可以生成 **NMI** 模式的中断消息，以调用 NMI 中断处理程序。

由于 **NMI** 无法被忽略，它们常被一些系统用作硬件监控工具。如果出现某些特定情况，比如在预定的时间内没有触发中断，NMI 处理程序就会产生警告并提供关于该问题的调试信息。这种机制有助于发现并预防系统死锁。

硬件性能计数器触发的 **PMI** 中断被设置为了 **NMI**类型。我们以 x86 平台为例，当 **PMI** 中断发生时，处理入口位置在 [arch/x86/entry/entry_64.S](https://elixir.bootlin.com/linux/v6.6.1/source/arch/x86/entry/entry_64.S)：

```asm
SYM_CODE_START(asm_exc_nmi)
  ...
 pushq 5*8(%rdx) /* pt_regs->ss */
 pushq 4*8(%rdx) /* pt_regs->rsp */
 pushq 3*8(%rdx) /* pt_regs->flags */
 pushq 2*8(%rdx) /* pt_regs->cs */
 pushq 1*8(%rdx) /* pt_regs->rip */
 UNWIND_HINT_IRET_REGS
 pushq   $-1  /* pt_regs->orig_ax */
 ...
 movq %rsp, %rdi
 movq $-1, %rsi
 call exc_nmi
```

[arch/x86/entry/entry_64.S](https://elixir.bootlin.com/linux/v6.6.1/source/arch/x86/entry/entry_64.S)是用汇编语言编写，负责系统调用、中断异常处理、任务切换和信号处理，专门针对 x86_64 架构进行了优化。

 `asm_exc_nmi` 函数是处理 **NMI** 的入口，从截取代码片段的注释可以看出， `asm_exc_nmi` 会使用 `pushq` 指令将当前的寄存器状态保存到内核栈上，这些包括程序计数器（**rip**，也就是 PC 寄存器）、代码段（**cs**）、标志（**flags**）、堆栈指针（**rsp**）和堆栈段（**ss**）。这一步非常关键，保留中断发生时的现场信息。最后使用**`call exc_nmi`** 指令调用 `exc_nmi` 函数。`exc_nmi` 会根据类型，调用预先注册的 NMI 处理函数。

如果是 **PMI** 类型的中断，最终调用的处理函数是 **perf_event_nmi_handler** ，定义在 [arch/x86/events/core.c](https://elixir.bootlin.com/linux/v6.6.1/source/arch/x86/events/core.c#L1729)中：

```c
static int
perf_event_nmi_handler(unsigned int cmd, struct pt_regs *regs)
{
 u64 start_clock;
 u64 finish_clock;
 int ret;

 /*
  * All PMUs/events that share this PMI handler should make sure to
  * increment active_events for their events.
  */
 if (!atomic_read(&active_events))
  return NMI_DONE;

 start_clock = sched_clock();
 ret = static_call(x86_pmu_handle_irq)(regs);
 finish_clock = sched_clock();

 perf_sample_event_took(finish_clock - start_clock);

 return ret;
}
```

可以看到，第二个参数 **pt_regs** 在前面介绍系统调用时已经出现过，是内核用来保存寄存器状态的结构体。这样在**perf_event_nmi_handler**中，我们就可以使用 **pt_regs** 中保存的寄存器状态，还原出 **PMI** 中断发生时，CPU 当时的动作，以及包含了用户态和内核态的完整代码执行路径。

至此，我们了解了 perf 在进行 CPU Profiling 时涉及的全部技术机制。

## 总结

现在再回顾我们使用 perf 进行 CPU Profiling 时的命令：

```bash
sudo perf record -F 99 -a -g -- sleep 30
```

perf 是事件驱动的方式工作，这个命令没有指定 **-e** 参数，会收集什么事件呢？perf 会默认收集 **cycles** 相关事件，从最精确的 **cycles:ppp** 到无精确设置的 **cycles**，优先选择可用且精度高的事件。如果没有硬件 **cycles** 事件可用，退而选择 **cpu-clock** 软件事件。

为什么采样 **cycles** 事件就能分析程序的 CPU 性能？因为每个 CPU 周期都会触发一个 **cycles** 事件，**cycles** 事件均匀的分布在程序的执行期间，以固定频率采样的 **cycles** 事件同样均匀分布，如果我们在采样 **cycles** 事件时，记录 CPU 正在干什么，持续一段时间收集到多个采样后，就能基于这些信息分析程序的行为，多次出现的同样动作，就可以认为是程序的热点。

如果指定采样频率？**-F 99** 设置采样频率为 99 Hertz，即每秒进行 99 次采样。也可以使用 **-c 1000** 设置采样周期，即每隔 1000 次事件进行一次采样。

如果知道采样时 CPU 正在做什么？通过CPU 的 **PC（Program Counter）寄存器**（x86-64 平台上对应的是 **rip** 寄存器）、指令寄存器等状态信息，能推断出 CPU 的瞬时动作。

知道了 CPU 采样时的“动作”还不够，还需要知道 CPU 是怎么做这个“动作”的，也就是代码的执行路径。系统利用了**堆栈（Stack）**的**后进先出（LIFO）**实现了函数调用，每个函数在堆栈上分配的空间称为**栈帧（stack frame）**，多个**栈帧（stack frame）**组成 [Call stack](https://en.wikipedia.org/wiki/Call_stack)，体现出了函数的调用关系。通过栈帧指针**Frame Pointer**（**rbp** 寄存器），可以追溯整个调用链，逐级访问到每个调用者的**栈帧（stack frame）**，重构出程序执行的路径。这就是 **-g** 参数的作用：使用 **Frame Pointer** 还原调用栈。

操作系统为了安全会限制用户进程对关键资源的访问，将系统分为了用户态和内核态，用户态的代码必须通过**系统调用（system call ）**访问核心资源。所以在执行系统调用时，进程具有两个栈：**用户栈（User Stack）**和**内核栈（Kernel Stack）**。为了还原包含了用户栈和内核栈在内完整的调用栈，我们探索了进程虚拟地址空间的布局，以及系统调用的实现：原来在系统调用时，会将进程用户态的执行状态（**rsp**、**rip**等寄存器）保存在内核数据结构 [struct pt_regs](https://elixir.bootlin.com/linux/v6.6.1/source/arch/x86/include/asm/ptrace.h#L59) 内，这样就能通过 **Frame Pointer** 和 **pt_regs** 分别还原内核栈和用户栈。

怎么获取采样发生时刻 CPU 寄存器的内容呢？在特定的时间间隔到达时，也就是该采样的时刻，APIC 会触发 **PMI** 中断，CPU 在将控制权转给中断处理程序之前，将当前的寄存器状态保存到**pt_regs**，然后作为参数传递给 **perf_event_nmi_handler**。这样 perf 就拿到了采样发生时刻，CPU 寄存器的内容。

最后，我们可以看一下 `perf record` 收集了什么样的数据。使用 `perf script` 命令可以打印收集在 `perf.data` 中的每个样本：

```bash
$ sudo perf script
...
sshd 50588 430947.269426:      16854 cycles: 
        ffffffff824b84f6 native_write_msr+0x6 (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff82413dc5 intel_pmu_enable_all+0x15 (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff82407abb x86_pmu_enable+0x1ab (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff8273d05a perf_ctx_enable+0x3a (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff8274646a __perf_event_task_sched_in+0x15a (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff82534bf9 finish_task_switch.isra.0+0x179 (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff834a57bf __schedule+0x2bf (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff834a5b68 schedule+0x68 (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff834ac3cb schedule_hrtimeout_range_clock+0x11b (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff834ac403 schedule_hrtimeout_range+0x13 (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff8289de3a do_poll.constprop.0+0x22a (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff8289e136 do_sys_poll+0x166 (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff8289e7cc __x64_sys_ppoll+0xbc (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff834931ac do_syscall_64+0x5c (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
        ffffffff836000eb entry_SYSCALL_64+0xab (/usr/lib/debug/boot/vmlinux-6.2.0-36-generic)
                  118e5f __ppoll+0x4f (inlined)
                   89a97 server_loop2.constprop.0+0x547 (/usr/sbin/sshd)
                   89a97 wait_until_can_do_something+0x547 (inlined)
                   89a97 server_loop2.constprop.0+0x547 (/usr/sbin/sshd)
                   2bcf7 do_authenticated2+0x1e7 (inlined)
                   2bcf7 do_authenticated+0x1e7 (/usr/sbin/sshd)
                   11b66 main+0x3616 (/usr/sbin/sshd)
                   29d8f __libc_start_call_main+0x7f (/usr/lib/x86_64-linux-gnu/libc.so.6)
                   29e3f __libc_start_main_impl+0x7f (inlined)
                   12844 _start+0x24 (/usr/sbin/sshd)
...
```

任意截取了其中一段输出，可以看到包含了用户态和内核态完整的调用栈。基于多个这样的代码执行路径，我们还能生成[火焰图](https://brendangregg.com/flamegraphs.html)进一步进行分析。

为了了解 `perf record` 的实现原理，我们在 Linux 内核进行了一段深入而刺激的旅程，感谢各位参与探险！

我的博客即将同步至腾讯云开发者社区，邀请大家一同入驻：<https://cloud.tencent.com/developer/support-plan?invite_code=1k39tbi20c30f>
