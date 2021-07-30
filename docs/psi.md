# 使用PSI（Pressure-Stall Information）监控服务器资源

我们经常会通过 `load average` 了解服务器的健康状况，检查服务器的负载是否正常。但 `load average` 有几个缺点：

- `load average` 的计算包含了 `TASK_RUNNING` 和 `TASK_UNINTERRUPTIBLE` 两种状态的进程。`TASK_RUNNING` 是进程处于运行，或等待分配 CPU 的准备运行状态。`TASK_UNINTERRUPTIBLE` 是进程处于不可中断的等待，一般是等待磁盘的输入输出。因此 `load average` 值高，可能是因为 CPU 资源不够，让处于 `TASK_RUNNING` 状态的进程过多，也可能是因为磁盘I/O资源紧张，造成很多进程处于 `TASK_UNINTERRUPTIBLE` 状态。你可以通过 `load average` 发现系统很忙，但没法区分是因为争夺 CPU 或 IO 引起的。
- `load average` 最短的时间窗口为1分钟，没法观察更短窗口的负载平均值，例如最近10秒的`load average`。
- `load average` 报告的是活跃进程数的原始数据，你还需要知道可用的CPU核数，这样`load average`才有意义。

所以，当你遇到服务器的`load average`偏高的时候，还需要查看 CPU、I/O和内存等资源的统计数据，才能进一步分析问题。

Facebook 的工程师向内核提交了一个patch：[PSI(Pressure Stall Information)](https://lwn.net/ml/cgroups/20180712172942.10094-1-hannes@cmpxchg.org/) ，通过暴露更多的系统资源利用率信息，让用户直观的了解到 CPU、内存和 IO 的压力。

`PSI` 已经包含在 4.20及以上版本的 Linux 内核中。

## PSI 概览

当 CPU、内存或 IO 设备争夺激烈的时候，系统会出现负载的延迟峰值、吞吐量下降，并可能触发内核的 `OOM Killer`。我们需要一个指标，衡量系统资源的压力情况。`PSI` 量化了由于硬件资源紧张造成的任务中断，统计了系统中任务等待硬件资源的时间，让用户了解系统所面临的资源压力。

持续监控 `PSI` 指标，可以让用户在资源变得紧张时，采取更主动、更细微的措施，例如将任务迁移到其他服务器，杀死低优先级的任务等。
