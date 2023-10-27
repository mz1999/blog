# 从源码构建 perf

[perf](https://perf.wiki.kernel.org/index.php/Main_Page) 是 Linux 官方的性能分析工具，它具备 profiling、tracing 和脚本编写等多种功能，是内核 perf_events 子系统的前端工具。

perf_events 也被称为 Performance Counters for Linux  (PCL) ，是在 [2009 年合并到 Linux内核主线源代码中](https://lwn.net/Articles/339361/)，成为内核一个新的子系统。perf_events 最初支持 [performance monitoring counters](https://en.wikipedia.org/wiki/Hardware_performance_counter?useskin=vector)  (PMC) ，后来逐步扩展支持基于多种事件源，包括：tracepoints、kprobes、uprobes 和 USDT。

![perf_event](https://cdn.mazhen.tech/images/202310261046436.png)

## 安装预编译二进制包

perf 包含在 linux-tools-common 中，首先安装该软件包：

```bash
sudo apt install linux-tools-common  
```

运行  `perf` 命令，可能会提示你安装另一个相关的软件包：

```bash
$ perf
WARNING: perf not found for kernel 6.2.0-35

  You may need to install the following packages for this specific kernel:
    linux-tools-6.2.0-35-generic
    linux-cloud-tools-6.2.0-35-generic

  You may also want to install one of the following packages to keep up to date:
    linux-tools-generic
    linux-cloud-tools-generic
```

按照提示安装和内核版本相关的 package：

```bash
sudo apt install linux-tools-6.2.0-35-generic
```

## 安装 Debug Symbol

perf 像其他调试工具一样，需要调试符号表信息（Debug symbols）。这些符号信息用于将内存地址转换为函数和变量名称。如果没有符号信息，你将看到代表被分析内存地址的十六进制数字。

```
13.19%     0.00%  sshd             libc.so.6                    [.] __libc_start_call_main
            |
            ---__libc_start_call_main
               |          
                --12.91%--0x5641182e4b66
                          0x5641182fecf7
                          |          
                          |--9.50%--0x56411835c752
                          |          |          
                          |           --9.10%--0x5641183330dd
                          |                     |          
                          |                      --8.80%--0x5641182ee730
                          |                                |          
                          |                                |--4.64%--0x564118303c43
```

通过 package 方式安装的软件，一般都会提供以 **-dbgsym** 或 **-dbg** 结尾的调试符号信息 package。在 Ubuntu 上，首先需要将`-dbgsym` 仓库添加到更新源中，执行下面的代码：

```bash
echo "deb http://ddebs.ubuntu.com $(lsb_release -cs) main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-updates main restricted universe multiverse
deb http://ddebs.ubuntu.com $(lsb_release -cs)-proposed main restricted universe multiverse" | \
sudo tee -a /etc/apt/sources.list.d/ddebs.list
```

然后从 Ubuntu 服务器导入调试符号 package 签名密钥：

```bash
sudo apt install ubuntu-dbgsym-keyring
```

最后更新软件源，安装 glibc 和 openssh-server 的调试符号信息：

``` bash
sudo apt update
sudo apt install libc6-dbg
sudo apt install openssh-server-dbgsym
```

安装了调试符号信息后，再使用 perf report 就会将十六进制数字转换为对应的方法名：

```
    11.68%     0.00%  sshd             libc.so.6                 [.] __libc_start_call_main
            |
            ---__libc_start_call_main
               main
               |          
                --11.46%--do_authenticated
                          do_authenticated2 (inlined)
                          server_loop2.constprop.0
                          |          
                          |--8.30%--process_buffered_input_packets (inlined)
                          |          ssh_dispatch_run_fatal
                          |          ssh_dispatch_run (inlined)
                          |          |          
                          |           --7.86%--server_input_channel_req (inlined)
                          |                     |          
                          |                      --7.78%--session_input_channel_req (inlined)
```

我们还可以安装内核镜像和一些内置命令行工具的调试符号信息：

```bash
sudo apt install coreutils-dbgsym
sudo apt install linux-image-`uname -r`-dbgsym
```

## Kernel Tracepoints

Kernel **tracepoint** 是在内核源码中关键位置的埋点，允许开发人员监视内核中的各种事件和操作，例如系统调用、TCP事件、文件系统I/O、磁盘I/O等，以了解内核的行为，进行性能分析和故障诊断。

使用 perf 可以对 tracepoints 进行统计、追踪和采样。例如可以使用下面的命令对上下文切换进行 1 秒钟的跟踪：

```bash
sudo perf record -e sched:sched_switch -a -g sleep 1
```

在执行这个命令的时候可能遇到下面的错误：

```
event syntax error: 'sched:sched_switch'
                     \___ unsupported tracepoint

libtraceevent is necessary for tracepoint support
Run 'perf list' for a list of valid events
```

错误信息说明不支持 `sched:sched_switch` 这个 tracepoint。我们运行 perf list 查看可用的 tracepoint：

```bash
$ sudo perf list 'sched:*'
List of pre-defined events (to be used in -e or -M):
...
  sched:sched_stat_wait                              [Tracepoint event]
  sched:sched_stick_numa                             [Tracepoint event]
  sched:sched_swap_numa                              [Tracepoint event]
  sched:sched_switch                                 [Tracepoint event]
  sched:sched_wait_task                              [Tracepoint event]
...
```

输出中明明包含了 `sched:sched_switch`，为什么 perf 不支持呢？

使用 `perf version --build-options` 查看 perf 的 build 选项：

```bash
$ perf version --build-options
perf version 6.2.16
                 dwarf: [ on  ]  # HAVE_DWARF_SUPPORT
    dwarf_getlocations: [ on  ]  # HAVE_DWARF_GETLOCATIONS_SUPPORT
                 glibc: [ on  ]  # HAVE_GLIBC_SUPPORT
         syscall_table: [ on  ]  # HAVE_SYSCALL_TABLE_SUPPORT
                libbfd: [ OFF ]  # HAVE_LIBBFD_SUPPORT
            debuginfod: [ OFF ]  # HAVE_DEBUGINFOD_SUPPORT
                libelf: [ on  ]  # HAVE_LIBELF_SUPPORT
               libnuma: [ on  ]  # HAVE_LIBNUMA_SUPPORT
numa_num_possible_cpus: [ on  ]  # HAVE_LIBNUMA_SUPPORT
               libperl: [ OFF ]  # HAVE_LIBPERL_SUPPORT
             libpython: [ OFF ]  # HAVE_LIBPYTHON_SUPPORT
              libslang: [ on  ]  # HAVE_SLANG_SUPPORT
             libcrypto: [ on  ]  # HAVE_LIBCRYPTO_SUPPORT
             libunwind: [ on  ]  # HAVE_LIBUNWIND_SUPPORT
    libdw-dwarf-unwind: [ on  ]  # HAVE_DWARF_SUPPORT
                  zlib: [ on  ]  # HAVE_ZLIB_SUPPORT
                  lzma: [ on  ]  # HAVE_LZMA_SUPPORT
             get_cpuid: [ on  ]  # HAVE_AUXTRACE_SUPPORT
                   bpf: [ on  ]  # HAVE_LIBBPF_SUPPORT
                   aio: [ on  ]  # HAVE_AIO_SUPPORT
                  zstd: [ OFF ]  # HAVE_ZSTD_SUPPORT
               libpfm4: [ OFF ]  # HAVE_LIBPFM
         libtraceevent: [ OFF ]  # HAVE_LIBTRACEEVENT
```

[libtraceevent](https://www.man7.org/linux/man-pages/man3/libtraceevent.3.html) 提供了访问内核 tracepoint 事件的 API。注意到最后一行，说明 perf 在 build 时没有打开 `libtraceevent`的支持。因此我们安装的预编译二进制包不能进行 tracepoint 追踪。我们需要自己从源码构建 perf。

## 从源码构建 perf

### 源码下载

首先下载 perf 的源代码。perf 的源码位于 Linux 内核源码中的 `tools/perf` 目录下。perf 是一个复杂的用户空间应用程序，而它却位于Linux 内核源代码树中，可能是唯一一个被包含在 Linux 源代码中的复杂用户软件。

为了下载和内核匹配的源码，先确定内核版本：

```bash
$ uname -r
6.2.0-35-generic
```

然后去 [https://www.kernel.org/pub/](https://www.kernel.org/pub/) 浏览并下载正确版本的源码。

### 安装依赖

安装构建依赖：

```bash
sudo apt-get install build-essential flex bison python3 python3-dev
sudo apt-get install libelf-dev libnewt-dev libdw-dev libaudit-dev libiberty-dev libunwind-dev libcap-dev libzstd-dev libnuma-dev libssl-dev python3-dev python3-setuptools binutils-dev gcc-multilib liblzma-dev
```

我们需要支持 tracepoint，所以还要安装 **libtraceevent** package：

```bash
sudo apt install libtraceevent-dev
```

### 构建

解压下载的 Linux 源码，进入源码目录，运行下面的命令：

```bash
PYTHON=python3 make -C tools/perf install
```

成功构建后 perf 被安装到了 `$HOME/bin` 目录。

### 测试验证

卸载先前安装的预编译版本：

```bash
sudo apt remove linux-tools-common 
```

将 `$HOME/bin` 加入到环境变量 `$PATH`，确保我们构建的 perf 命令能被找到。注意一般我们都需要 `sudo` 执行 perf 命令，所以还要编辑 `/etc/sudoers`（必须使用 visudo）文件，在 `Defaults secure_path="..."` 中加入 perf 命令的路径。

验证 perf 的构建选项：

```bash
$ sudo perf version --build-options
perf version 6.2.0
         ...
         libtraceevent: [ on  ]  # HAVE_LIBTRACEEVENT
```

这次 libtraceevent 打开了，支持 tracepoint 的追踪。执行下面的命令追踪系统的上下文切换：

```bash
sudo perf record -e sched:sched_switch -a --call-graph dwarf  sleep 1
```

查看报告：

```bash
$ sudo perf report
# To display the perf.data header info, please use --header/--header-only options.
#
#
# Total Lost Samples: 0
#
# Samples: 499  of event 'sched:sched_switch'
# Event count (approx.): 499
#
# Children      Self  Command          Shared Object               Symbol                               
# ........  ........  ...............  ..........................  .....................................
#
    48.70%    48.70%  swapper          [kernel.vmlinux]            [k] __schedule
            |
            ---secondary_startup_64_no_verify
               |          
               |--39.28%--start_secondary
               |          cpu_startup_entry
               |          do_idle
               |          schedule_idle
               |          __schedule
               |          __schedule
...
```

另外一个例子，按类型统计整个系统的系统调用，持续 5 秒钟：

```bash
$ sudo perf stat -e 'syscalls:sys_enter_*' -a sleep 5

 Performance counter stats for 'system wide':

               ...         
                25      syscalls:sys_enter_timerfd_settime                                      
                 0      syscalls:sys_enter_timerfd_gettime                                      
                 0      syscalls:sys_enter_signalfd4                                          
                 0      syscalls:sys_enter_signalfd                                           
                 0      syscalls:sys_enter_epoll_create1                                      
                 0      syscalls:sys_enter_epoll_create                                       
                 0      syscalls:sys_enter_epoll_ctl                                          
                37      syscalls:sys_enter_epoll_wait                                         
                80      syscalls:sys_enter_epoll_pwait                                        
                 0      syscalls:sys_enter_epoll_pwait2                                       
                                         
                ...                                   

       5.004101037 seconds time elapsed
```

可以看到，我们自己 build 的 perf 可以对 tracepoint 进行追踪和统计。
