# Perf 的 cpu-clock 与 cpu-cycles 采样实现与对比

![Feature image](https://cdn.mazhen.tech/2024/202506201359538.png)

在上一篇[博客](https://mazhen.tech/p/wall-clock-%E4%B8%8E-cpu-cycles-%E9%87%87%E6%A0%B7%E7%9A%84%E5%8C%BA%E5%88%AB/)中，我们深入探讨了性能分析中两个概念：**Wall-Clock 采样**与 **CPU-Cycles 采样**的本质区别。这一次我想使用 `perf` 这个强大的工具来验证上一篇的内容。

对于 **CPU-Cycles 采样**，路径非常明确：`perf record -e cycles`，或者干脆不带 `-e` 参数时，默认使用的就是这种基于硬件的 On-CPU 分析。

但问题来了：如何用 `perf` 实现 Wall-Clock 采样呢？

一个看似可能的答案是 `cpu-clock` 软件事件。然而，[官方文档](https://man7.org/linux/man-pages/man2/perf_event_open.2.html)对它的描述只有一句，语焉不详：

> This reports the CPU clock, a high-resolution per-CPU timer.
>
> 报告提供 CPU 时钟信息，即一种高精度的每 CPU 计时器。

我尝试用各种 AI DeepResearch 一番，仍然没有找到答案，而且发现自己之前写的那篇[《深入探索 perf CPU Profiling 实现原理》](https://mazhen.tech/p/%E6%B7%B1%E5%85%A5%E6%8E%A2%E7%B4%A2-perf-cpu-profiling-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86/)竟排在参考引用文档列表中很高的位置。同时，另一位知名开发者 Robert Haas（PostgreSQL 的核心贡献者）也曾[抱怨过](https://rhaas.blogspot.com/2012/06/perf-good-bad-ugly.html) `cpu-clock` 缺乏文档说明。

既然现有的资料无法给出确切答案，唯一的办法就是到 Linux 内核源码中找答案，搞清楚 `cpu-clock` 和 `cycles` 这对采样事件，究竟在底层是如何实现的，也算给 AI 贡献一些原创素材。

## 深入内核：两种采样事件的实现

### `cpu-clock` 的实现

通过内核源码 [kernel/events/core.c](https://github.com/torvalds/linux/blob/master/kernel/events/core.c)，我们可以分析出 `cpu-clock` 事件实现的核心逻辑，它的实现依赖于 Linux 的高精度定时器（`hrtimer`）。

整个流程可以概括为：

1.  **初始化**：当一个 `cpu-clock` 事件被创建时，`cpu_clock_event_init()` 会调用 `perf_swevent_init_hrtimer()`。这会在每个被监控的 CPU 上设置一个周期性的 `hrtimer`。
2.  **定时中断**：这个 `hrtimer` 会以我们指定的频率（例如 `-F 99` 就是 99Hz）周期性地触发一个硬件中断。
3.  **中断处理**：中断会激活 `perf_swevent_hrtimer()` 这个处理函数。在这个函数里，内核会判断事件是否需要记录样本。
4.  **样本采集**：如果需要，它会调用 `__perf_event_overflow()`，最终触发 `overflow_handler`（如 `perf_event_output_forward`），后者会收集当前上下文的数据（如调用栈、寄存器等），打包成一个 `PERF_RECORD_SAMPLE`，然后写入 Ring Buffer。

简单来说，**`cpu-clock` 是一个纯粹的软件事件，使用 hrtimer 周期性地触发一个中断，中断处理函数负责生成并记录一个样本。** 它就像一个设定的闹钟，每隔固定的时间就响一次，然后记录下那个瞬间 CPU 正在处理的任务。

### `cpu-cycles` 的实现

`cpu-cycles` 是一个硬件事件，它直接利用 CPU 内部的性能监控单元（PMU）来统计 CPU 时钟周期，它的实现与 `cpu-clock` 软件事件有本质区别。

以  x86 平台上为例，`cpu-cycles` 事件的实现在 [arch/x86/events/core.c](https://github.com/torvalds/linux/blob/master/arch/x86/events/core.c) 文件中，我们可以看到它的实现与硬件紧密耦合。整体流程如下：

1.  **初始化与编程**：当一个 `cpu-cycles` 事件被创建时，`x86_pmu_event_init()` 会将这个通用事件类型映射为一个特定的硬件事件编码。事件被启用时，调度器（`x86_schedule_events`）会为其分配一个可用的硬件性能计数器（PMC）。内核通过 `wrmsr` 指令将事件编码写入 `PerfEvtSel` 寄存器，并设置 `PMC` 寄存器的初始值为一个负数（例如 `-sample_period`），然后启动该计数器。

2.  **硬件计数**：CPU 硬件开始自动对每个时钟周期进行计数。`PMC` 的值从初始的负数开始，每个周期加一，向零逼近。这个过程完全由硬件执行，没有软件开销。

3.  **硬件中断 (PMI)**：当硬件计数器从 `0xFFFF...` 溢出到 `0` 时，PMU 会自动触发一个**性能监控中断（PMI）**。在 x86 平台上，这通常是一个**不可屏蔽中断（NMI）**。

4.  **中断处理**：NMI 会激活 `perf_event_nmi_handler()`，它会调用特定于 PMU 的中断处理函数（如 `x86_pmu.handle_irq`）。此函数会：
    a.  **识别溢出**：检查是哪个 `PMC` 发生了溢出。
    b.  **重置计数器**：调用 `x86_perf_event_set_period()` 将该 `PMC` 的值重新设置为初始的负数，为下一次采样做准备。
    c.  **样本采集**：调用 `perf_event_overflow()`，最终触发 `overflow_handler`（如 `perf_event_output_forward`），后者会收集当前上下文的数据（如调用栈、寄存器等），打包成一个 `PERF_RECORD_SAMPLE`，然后写入 Ring Buffer。

**简而言之，`cpu-cycles` 是一个硬件事件。** 它不像闹钟，更像一个只在工作时才计数的“计步器”。它关心的是 **CPU 完成的工作量**。

### 它们采集了哪些数据？

无论是哪种方式触发了采样，最终内核都会通过 `perf_prepare_sample()` 和 `perf_output_sample()` 这一系列的函数，将当时的“现场快照”打包成一个 `PERF_RECORD_SAMPLE` 记录。这份快照通常包含（取决于你的 `-g` 和其他 `perf` 参数）：

*   **指令指针 (IP)**: CPU 当时正在执行哪一条指令的地址。
*   **进程 ID 和线程 ID (PID/TID)**: 是哪个进程/线程在执行。
*   **时间戳 (Time)**: 采样发生的时间。
*   **CPU 编号**: 在哪个 CPU 核心上发生的采样。
*   **调用链 (Callchain)**: 这是通过 `-g` 参数启用的。内核会从当前的栈帧指针（Frame Pointer）开始，回溯整个调用栈，记录下从当前函数到顶层函数的完整路径。对于系统调用，它能同时抓取到用户栈和内核栈，拼接成一个完整的调用链。
*   **其他元数据**: 如事件周期、地址信息等。

正是这些丰富的数据，让我们能够在事后通过 `perf report` 或火焰图，重构出程序的运行画像。

### 两个事件的对比

| 特性 | `cpu-clock` | `cpu-cycles` |
| :--- | :--- | :--- |
| **事件源** | 软件 (内核 `hrtimer`) | 硬件 (CPU PMU) |
| **触发机制**| 固定**时间**间隔到达，触发定时器中断 | 固定**工作量**完成，触发 PMC 溢出中断 |
| **CPU 空闲时** | 计时器**继续**运行，会采样到`idle`任务 | 计数器**暂停**，不会产生任何采样 |

## 动手验证

理论分析之后，我们需要一个实验来直观地感受这种差异。一个好的实验，应该能创造出极端的对比场景，直观的展示出两种采样方式的差异。

我的思路是构建一个“脉冲式”的负载模型，让程序在两种截然不同的状态间切换：

- **CPU 饱和阶段**: 计算密集型，消耗系统所有的 CPU 资源。
- **空闲阶段**: 程序进入长时间的休眠，让所有 CPU 核心休息。

### 实验程序

实验程序是一个多线程 C 程序 `work_and_wait_mt.c`，它会创建与核心数相同的工作线程，所有线程通过屏障（barrier）同步，同时开始计算，同时结束计算，实现“集体计算 100 毫秒，集体休眠 900 毫秒”的脉冲式负载。

```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>
#include <stdbool.h>
#include <stdlib.h>

// --- Configuration ---
long burn_time_ms = 100;
long idle_time_ms = 900;
int total_cycles = 10;
long multiplier = 200000; // **重要**: 请先用单线程校准这个值！

// --- Globals for thread coordination ---
pthread_barrier_t barrier;
volatile bool time_to_exit = false;

// CPU 密集型计算函数
void __attribute__((noinline)) burn_cpu(long milliseconds) {
    volatile unsigned long i;
    for (i = 0; i < milliseconds * multiplier; i++) {
        // Burn baby burn
    }
}

// 工作线程的执行函数
void* worker_function(void* arg) {
    int thread_id = *(int*)arg;
    printf("Worker thread %d started.\n", thread_id);

    while (!time_to_exit) {
        // 1. 等待主线程的“开始计算”信号
        //    所有工作线程和主线程都会在此集合
        pthread_barrier_wait(&barrier);

        // 如果是退出信号，则跳出循环
        if (time_to_exit) {
            break;
        }

        // 2. 执行计算任务
        burn_cpu(burn_time_ms);

        // 3. 等待所有其他工作线程完成计算
        //    主线程也在此等待，以确认计算阶段结束
        pthread_barrier_wait(&barrier);
    }

    printf("Worker thread %d exiting.\n", thread_id);
    return NULL;
}

int main() {
    // 自动检测可用的 CPU 核心数
    long num_cores = sysconf(_SC_NPROCESSORS_ONLN);
    if (num_cores <= 0) {
        num_cores = 1; // 备用值
    }
    printf("Detected %ld CPU cores.\n", num_cores);

    // +1 是因为主线程也要参与屏障同步
    pthread_barrier_init(&barrier, NULL, num_cores + 1);

    pthread_t* threads = malloc(num_cores * sizeof(pthread_t));
    int* thread_ids = malloc(num_cores * sizeof(int));

    // 创建工作线程
    for (long i = 0; i < num_cores; i++) {
        thread_ids[i] = i;
        pthread_create(&threads[i], NULL, worker_function, &thread_ids[i]);
    }

    printf("\nStarting multi-threaded workload...\n");
    printf("----------------------------------------------------------\n");

    for (int i = 0; i < total_cycles; i++) {
        printf("Cycle %d/%d: Orchestrating CPU burn...\n", i + 1, total_cycles);
        
        // 1. 主线程到达屏障，释放所有等待的 worker 线程开始计算
        pthread_barrier_wait(&barrier);

        // 2. 主线程也在此等待，直到所有 worker 线程都完成计算
        pthread_barrier_wait(&barrier);

        printf("Cycle %d/%d: Orchestrating idle period...\n", i + 1, total_cycles);
        usleep(idle_time_ms * 1000);
    }
    
    printf("----------------------------------------------------------\n");
    printf("Workload finished. Signaling threads to exit.\n");

    // 通知所有线程退出
    time_to_exit = true;
    // 最后一次释放屏障，让所有在等待的线程可以检查到退出标志
    pthread_barrier_wait(&barrier); 

    // 等待所有线程结束并清理资源
    for (long i = 0; i < num_cores; i++) {
        pthread_join(threads[i], NULL);
    }
    
    pthread_barrier_destroy(&barrier);
    free(threads);
    free(thread_ids);
    
    return 0;
}

```

在编写和测试这样的程序时，有两个细节需要注意。

#### 防止函数内联

现代编译器非常智能，当你使用 `-O2` 等优化选项时，它可能会发现 `burn_cpu` 函数非常简单，从而执行**函数内联（Function Inlining）**——直接把 `burn_cpu` 的循环代码“复制 - 粘贴”到调用它的 `worker_function` 中。这会导致在 `perf` 报告里，我们看不到 `burn_cpu` 这个独立的函数，所有的开销都会被算在 `worker_function` 头上，这会干扰我们的分析。为了避免这种情况，我们必须明确地告诉编译器不要这么做：

```c
void __attribute__((noinline)) burn_cpu(long milliseconds) {
    // ...
}
```

#### 校准计算强度

`burn_cpu` 函数里的循环次数需要根据你的 CPU 性能进行调整。一个固定的循环次数，在我的电脑上可能运行 100 毫秒，在你的高性能服务器上可能只需要 30 毫秒。我们需要一个合适的“乘数（multiplier）”，让 `burn_cpu(100)` 能在你的机器上**大致运行 100 毫秒**。

我使用了一个简单的校准程序 `calibrate.c`，它只包含 `burn_cpu(1000)`，然后用 `time` 命令来测量其**`user`时间**（即 CPU 在用户态的实际耗时）。

```c
#include <stdio.h>

void burn_cpu(long milliseconds) {
    // 这是我们要调整的乘数，先从一个初始值开始
    long multiplier = 200000;
    volatile unsigned long i;
    for (i = 0; i < milliseconds * multiplier; i++) {
        // Just burn cycles
    }
}

int main() {
    long test_duration_ms = 1000; // 我们期望它运行 1 秒钟
    printf("Burning CPU for a target of %ldms...\n", test_duration_ms);
    burn_cpu(test_duration_ms);
    printf("Calibration burn finished.\n");
    return 0;
}
```

现在，我们编译并使用 `time` 命令来测量这个程序的实际运行时间。我们使用 `-O2` 优化，这更接近生产环境的编译设置。

```bash
$ gcc -O2 -o calibrate calibrate.c
$ time ./calibrate
```

`time` 命令会输出类似下面的结果：

```bash
Burning CPU for a target of 1000ms...
Calibration burn finished.

real	0m0.052s
user	0m0.051s
sys	  0m0.001s
```

这里的关键是 **`user` 时间**。它表示程序在用户态实际消耗的 CPU 时间。

*   **目标时间**: 1.0 秒 (1000ms)
*   **实际时间**: 0.051 秒

这说明我们当前的乘数 `200000` 太小了，程序运行得太快。我们可以用一个简单的比例来计算新的乘数：

**新乘数 = 旧乘数 * (目标时间 / 实际时间)**

在这个例子中：
新乘数 = 200,000 * (1.0 / 0.051) ≈ 200,000 * 19.6 ≈ **3920000**

我们可以取一个整数，比如 **4000000**。

现在，修改 `calibrate.c` 中的乘数为 `4000000`，然后再次运行校准测试：

```bash
$ gcc -O2 -o calibrate calibrate.c
$ time ./calibrate
Burning CPU for a target of 1000ms...
Calibration burn finished.

real	0m0.987s
user	0m0.984s
sys	  0m0.002s
```

这次的输出 `user` 时间应该会非常接近 1.0 秒了，现在就可以用这个新的乘数修改 `work_and_wait_mt.c` ，来进行 `perf` 对比实验了。

### 编译实验程序

我们使用以下命令进行编译：

```bash
gcc -g -O2 -o work_and_wait_mt work_and_wait_mt.c -pthread
```

*   `-g`: 生成调试信息，这是 `perf` 能够将内存地址翻译成函数名的关键。
*   `-O2`: 开启二级优化，这更接近生产环境的编译设置，也使得我们防止内联的操作变得必要。
*   `-pthread`: 链接 `pthread` 库，因为我们用到了多线程和屏障。

### 执行采样

现在，我们用两种不同的采样方式，在**系统全局模式（`-a`）**下，对这个程序进行分析。

我们使用系统全局（`-a`）模式，以确保能捕获到 `idle` 任务。

#### 实验一：`cpu-cycles` 事件

1. **采集数据**

````bash
$ sudo perf record -a -F 99 -g -- ./work_and_wait_mt
````

2. **生成报告**

```bash
$ sudo perf report
```

3. **报告解读**

```bash
Samples: 3K of event 'cpu_core/cycles/P', Event count (approx.): 124421182834
  Children      Self  Command          Shared Object          Symbol
+   99.28%    99.12%  work_and_wait_m  work_and_wait_mt       [.] burn_cpu
     0.25%     0.00%  swapper          [kernel.kallsyms]      [k] secondary_startup_64_no_verify
     0.25%     0.00%  swapper          [kernel.kallsyms]      [k] cpu_startup_entry
     0.25%     0.00%  swapper          [kernel.kallsyms]      [k] do_idle
     0.20%     0.00%  swapper          [kernel.kallsyms]      [k] start_secondary
     0.19%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] asm_sysvec_apic_timer_interrupt
     0.19%     0.00%  swapper          [kernel.kallsyms]      [k] cpuidle_idle_call
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] sysvec_apic_timer_interrupt
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] __sysvec_apic_timer_interrupt
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] hrtimer_interrupt
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] __hrtimer_run_queues
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] tick_nohz_highres_handler
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] tick_sched_handle
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] update_process_times
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] scheduler_tick
     0.16%     0.00%  work_and_wait_m  [kernel.kallsyms]      [k] task_tick_fair
     0.16%     0.00%  swapper          [kernel.kallsyms]      [k] call_cpuidle
...
```

第一行是报告的标题：

- **Samples: 3K**: 在程序运行期间，perf 总共捕获了 3000 个样本。
- **event 'cpu_core/cycles/P'**: perf 告诉你它采样的是 cycles 事件。这个名字 cpu_core/cycles/P 是一个更具体的硬件事件名称，但本质就是 CPU 时钟周期。
- **Event count (approx.)**: 程序在运行时，大约产生了 1 千 2 百亿个 CPU 周期。

接下来是表格的列：

- **Children**: **子项开销**。表示这个函数**以及它调用的所有子函数**总共占用的 CPU 时间百分比。可以理解为，当程序执行到这个函数所在的调用链上时，所花费的时间。
- **Self**: **自身开销**。表示 CPU 时间**只花在这个函数本身内部**的百分比，不包括它调用的其他函数。这是定位代码热点的最关键指标。
- **Command**: 运行的命令名 (work_and_wait)。
- **Shared Object**: 该符号所在的库或可执行文件。
- **Symbol**: 函数名或内核符号名。

**Self: 99.12%**，这是最重要的信息。在所有样本中，有 **99.12%** 的样本都是在 CPU 执行 `burn_cpu` 函数内部的指令时捕获的。这清晰地表明，这个函数是程序中唯一的 CPU 性能热点。

`swapper`（即 idle 任务）在这里也出现了，但它的占比极小，与 `burn_cpu` 产生的海量周期相比，这些内核自身的管理开销几乎可以忽略不计。

**总结 cpu-cycles 报告：** `burn_cpu`消耗了 **99.12%** 的 CPU 算力，`perf` 准确地抓住了那个让 CPU 发热的元凶。**`swapper`（idle 任务）几乎可以忽略不计**，证明了 `cpu-cycles` 采样在程序休眠时是“暂停”的，它只关心工作，不关心等待。

#### 实验二：`cpu-clock` 事件

1. **采集数据**

```bash
$ sudo perf record -a -F 99 -e cpu-clock -g -- ./work_and_wait_mt
```

2. **生成报告**:

```bash
$ sudo perf report
```

4. **报告解读**:

```bash
Samples: 15K of event 'cpu-clock', Event count (approx.): 156434341870
  Children      Self  Command          Shared Object          Symbol
+   63.88%     0.00%  swapper          [kernel.kallsyms]      [k] secondary_startup_64_no_verify
+   63.88%     0.00%  swapper          [kernel.kallsyms]      [k] cpu_startup_entry
+   63.88%     0.00%  swapper          [kernel.kallsyms]      [k] do_idle
+   63.87%     0.00%  swapper          [kernel.kallsyms]      [k] cpuidle_idle_call
+   63.87%     0.00%  swapper          [kernel.kallsyms]      [k] call_cpuidle
+   63.87%    63.85%  swapper          [kernel.kallsyms]      [k] cpuidle_enter_state
+   63.87%     0.00%  swapper          [kernel.kallsyms]      [k] cpuidle_enter
+   58.09%     0.00%  swapper          [kernel.kallsyms]      [k] start_secondary
+   36.08%    36.08%  work_and_wait_m  work_and_wait_mt       [.] burn_cpu
+    5.79%     0.00%  swapper          [kernel.kallsyms]      [k] x86_64_start_kernel
+    5.79%     0.00%  swapper          [kernel.kallsyms]      [k] x86_64_start_reservations
+    5.79%     0.00%  swapper          [kernel.kallsyms]      [k] start_kernel
+    5.79%     0.00%  swapper          [kernel.kallsyms]      [k] arch_call_rest_init
+    5.79%     0.00%  swapper          [kernel.kallsyms]      [k] rest_init
     0.01%     0.01%  Lingma           [kernel.kallsyms]      [k] finish_task_switch.isra.0
...
```
情况发生了戏剧性的反转。在这份报告中，`swapper` （空闲任务）成为了绝对的主角。有 **63.85%** 的时间样本，都捕获到 CPU 正处于 `cpuidle_enter_state` 这个函数内部，这正是内核让 CPU“休眠”的地方。

`burn_cpu` **沦为配角 (36.08% Self)**，曾经消耗了 99% CPU 算力的 `burn_cpu` 函数，在时间的维度上，只占了 36.08% 的份额。

**总结 cpu-clock 报告：** **`swapper` 成了主角**，占据了 **63.85%** 的时间样本，证明了 CPU 在绝大部分**时间**里，都处于内核安排的“睡眠”状态。**`burn_cpu` 沦为配角**，只占了 **36.08%** 的时间。

#### 对比总结

| 采样事件       | 核心问题             | burn_cpu 表现      | swapper 表现       | 分析视角                       |
| -------------- | -------------------- | ------------------ | ------------------ | ------------------------------ |
| **cpu-cycles** | “谁消耗了**算力**？” | **绝对主力 (99%)** | 微不足道           | **On-CPU** (工作量分析)        |
| **cpu-clock**  | “**时间**花在哪了？” | 少数派 (~36%)      | **绝对主力 (64%)** | **Wall-Clock** (延迟/等待分析) |

### 一个有用的技巧：进程专属模式下的 `cpu-clock`

如果不使用 `-a` 参数，而是运行以下命令：

```bash
$ sudo perf record -F 99 -e cpu-clock -g -- ./work_and_wait_mt
```

得到的报告竟然和 `cycles` 采样的结果惊人地一致，`swapper` 消失了，`burn_cpu` 再次占据了 99% 以上的份额，休息时间（swapper/idle）完全消失了。

```bash
Samples: 5K of event 'cpu-clock', Event count (approx.): 56252524690
  Children      Self  Command          Shared Object      Symbol
+   99.98%    99.98%  work_and_wait_m  work_and_wait_mt   [.] burn_cpu
     0.02%     0.02%  work_and_wait_m  [kernel.kallsyms]  [k] __check_heap_object
     0.02%     0.00%  work_and_wait_m  work_and_wait_mt   [.] worker_function
     0.02%     0.00%  work_and_wait_m  libc.so.6          [.] __printf_chk
     0.02%     0.00%  work_and_wait_m  libc.so.6          [.] __vfprintf_internal
     0.02%     0.00%  work_and_wait_m  libc.so.6          [.] __printf_buffer_to_file_done
     0.02%     0.00%  work_and_wait_m  libc.so.6          [.] _IO_file_xsputn@@GLIBC_2.2.5
     0.02%     0.00%  work_and_wait_m  libc.so.6          [.] _IO_do_write@@GLIBC_2.2.5
     0.02%     0.00%  work_and_wait_m  libc.so.6          [.] _IO_file_write@@GLIBC_2.2.5
     0.02%     0.00%  work_and_wait_m  libc.so.6          [.] __GI___libc_write
     0.02%     0.00%  work_and_wait_m  [kernel.kallsyms]  [k] entry_SYSCALL_64_after_hwframe
     0.02%     0.00%  work_and_wait_m  [kernel.kallsyms]  [k] do_syscall_64
     0.02%     0.00%  work_and_wait_m  [kernel.kallsyms]  [k] x64_sys_call
...
```

原因是这次使用的命令是**进程专属（Process-Specific）模式**，而不是**系统全局（System-Wide）模式**。

`-a`参数的全称是 **`--all-cpus`**，是对全系统范围内所有 CPU 进行收集。如果不加 `-a`，上面的命令明确告诉 perf，**只监控 ./work_and_wait_mt 这个进程以及它创建的所有子进程。对于任何其他进程的活动，请直接忽略。**

当 `perf` 在**进程专属模式**下运行时，它内部有一个强大的“过滤器”，当 `cpu-clock` 的“闹钟”响起时，`perf` 会检查当前在 CPU 上运行的是哪个进程。如果恰好是我们的 `work_and_wait_mt`，样本就被记录下来。如果在程序休眠期间，CPU 上运行的是 `swapper` 任务，`perf` 会发现这个任务不属于它的监控目标，于是**直接丢弃了这个样本**。

通过这个实验我们发现，在**进程专属模式**下，`cpu-clock` 的行为会因为进程过滤器的存在而变得非常像 `cpu-cycles`。这意味着，**在不方便使用硬件 `cycles` 事件，或者想纯粹用软件事件来分析一个特定程序的 On-CPU 热点时，使用“进程专属模式”下的 `cpu-clock` 是一个完全可行的替代方案。** 同时，要真正地用 `cpu-clock` 来分析等待时间（Off-CPU），**必须使用 -a 参数切换到系统全局模式**。

## 总结

通过对内核源码的剖析，以及一系列实验，现在我们可以回答开头那个令人困惑的问题了：cpu-clock 和 cycles 到底有何不同？

*   **`cpu-clock`** 基于**时间**，它的闹钟忠实地记录下每个时间点的系统状态。
*   **`cpu-cycles`** 基于**工作量**，它的计步器精准地衡量了 CPU 的实际消耗。

另外，在**进程专属模式**下，perf 内置的“过滤器”会将所有非目标进程的样本（包括 swapper）丢弃，使得 `cpu-clock` 的行为趋近于 `cpu-cycles`，成为一种纯软件实现的 On-CPU 分析替代方案。

当面对文档的语焉不详和网络信息的模糊不清时，只有深入源码、亲手验证，才是揭开迷雾正确路径。希望这篇文章对你也有帮助。
