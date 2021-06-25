# 使用Ftrace开始内核探索之旅

操作系统内核对应用开发工程师来说就像一个黑盒，似乎很难窥探到其内部的运行机制。其实Linux内核很早就内置了一个强大的tracing工具：[Ftrace](https://www.kernel.org/doc/html/latest/trace/ftrace.html)，它几乎可以跟踪内核的所有函数，不仅可以用于调试和分析，还可以用于观察学习Linux内核的内部运行。虽然`Ftrace`在2008年就[加入了内核](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=16444a8a40d4c7b4f6de34af0cae1f76a4f6c901)，但很多应用开发工程师仍然不知道它的存在。本文就给你介绍一下`Ftrace`的基本使用。

## Ftrace初体验 ##

先用一个例子体验一下`Ftrace`的使用简单，且功能强大。使用 root 用户进入`/sys/kernel/debug/tracing`目录，执行 `echo` 和 `cat` 命令：

```
# echo _do_fork > set_graph_function
# echo function_graph > current_tracer 
# cat trace | head -20
# tracer: function_graph
#
# CPU  DURATION                  FUNCTION CALLS
# |     |   |                     |   |   |   |
 3)               |  _do_fork() {
 3)               |    copy_process() {
 3)   0.895 us    |      _raw_spin_lock_irq();
 3)               |      recalc_sigpending() {
 3)   0.740 us    |        recalc_sigpending_tsk();
 3)   2.248 us    |      }
 3)               |      dup_task_struct() {
 3)   0.775 us    |        tsk_fork_get_node();
 3)               |        kmem_cache_alloc_node() {
 3)               |          _cond_resched() {
 3)   0.740 us    |            rcu_all_qs();
 3)   2.117 us    |          }
 3)   0.701 us    |          should_failslab();
 3)   2.023 us    |          memcg_kmem_get_cache();
 3)   0.889 us    |          memcg_kmem_put_cache();
 3) + 12.206 us   |        }

```

我们使用`Ftrace`的`function_graph`功能显示了内核函数 `_do_fork()` 所有子函数调用。左边的第一列是执行函数的 CPU，第二列 `DURATION` 显示在相应函数中花费的时间。我们注意到最后一行的耗时之前有个 `+` 号，提示用户注意延迟高的函数。`+` 代表耗时大于 `10 μs`。如果耗时大于 `100 μs`，则显示 `!` 号。

我们知道，`fork` 是建立父进程的一个完整副本，然后作为子进程执行。那么`_do_fork()`的第一件大事就是调用 `copy_process()` 复制父进程的数据结构，从上面输出的调用链信息也验证了这一点。

使用完后执行下面的命令关闭`function_graph`：

```
# echo nop > current_tracer
# echo > set_graph_function
```

使用 `Ftrace` 的 `function_graph` 功能，可以查看内核函数的子函数调用链，帮助我们理解复杂的代码流程，而这只是 `Ftrace` 的功能之一。这么强大的功能，我们不必安装额外的用户空间工具，只要使用 `echo` 和 `cat` 命令访问特定的文件就能实现。`Ftrace` 对用户的使用接口正是**tracefs**文件系统。

## tracefs 文件系统

用户通过**tracefs**文件系统使用`Ftrace`，这很符合一切皆文件的Linux哲学。**tracefs**文件系统一般挂载在`/sys/kernel/tracing`目录。由于`Ftrace`最初是`debugfs`文件系统的一部分，后来才被拆分为自己的`tracefs`，所以如果系统已经挂载了`debugfs`，仍然会保留原始的目录结构，将`tracefs`挂载到`debugfs`的子目录下。我们可以使用 mount 命令查看当前系统`debugfs`和`tracefs`挂载点：

```
# mount -t debugfs,tracefs
debugfs on /sys/kernel/debug type debugfs (rw,nosuid,nodev,noexec,relatime)
tracefs on /sys/kernel/tracing type tracefs (rw,nosuid,nodev,noexec,relatime)
tracefs on /sys/kernel/debug/tracing type tracefs (rw,nosuid,nodev,noexec,relatime)
```

我使用的系统是`Ubuntu 20.04.2 LTS`，可以看到，为了保持兼容，`tracefs`同时挂载到了`/sys/kernel/tracing`和`/sys/kernel/debug/tracing`。本文后面的示例假定你已经处在了`/sys/kernel/tracing`或`/sys/kernel/debug/tracing`目录下。

## 函数跟踪

`Ftrace` 实际上代表的就是`function trace`（函数跟踪），因此函数追踪是`Ftrace`最初主要的一个功能。
