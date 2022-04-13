# Linux 信号(Signal)

我们经常会使用 `kill` 命令杀掉运行中的进程，对多次杀不死的进程进一步用 `kill -9` 干掉它。你可能知道这是在用 `kill` 命令向进程发送信号，优雅或粗暴的让进程退出。我们能向进程发送很多类型的信号，其中一些常见的信号 **SIGINT** 、**SIGQUIT**、 **SIGTERM** 和 **SIGKILL** 都是通知进程退出，但它们有什么区别呢？很多人经常把它们搞混，这篇文章会让你了解 Linux 的信号机制，以及一些常见信号的作用。

## 什么是信号

信号（Signal）是 Linux 进程收到的一个通知。当进程收到一个信号时，该进程会中断其执行，并执行收到信号对应的处理程序。

信号机制作为 Linux 进程间通信的一种方法。Linux 进程间通信常用的方法还有管道、消息、共享内存等。

信号的产生有多种来源：

* 硬件来源，例如 CPU 内存访问出错，当前进程会收到信号 SIGSEGV；按下 `Ctrl+C` 键，当前运行的进程会收到信号 SIGINT 而退出；
* 软件来源，例如用户通过命令 `kill [pid]`，直接向一个进程发送信号。进程使用系统调用 [int kill(pid_t pid, int sig)](https://man7.org/linux/man-pages/man2/kill.2.html) 显示的向另一个进程发送信号。内核在某些情况下，也会给进程发送信号，例如当子进程退出时，内核给父进程发送 SIGCHLD 信号。

你可以使用 `kill -l` 命令查看系统实现了哪些信号：

```
$ kill -l
 1) SIGHUP	    2) SIGINT	    3) SIGQUIT	    4) SIGILL	    5) SIGTRAP
 6) SIGABRT	    7) SIGBUS	    8) SIGFPE	    9) SIGKILL	    10) SIGUSR1
11) SIGSEGV	    12) SIGUSR2	    13) SIGPIPE	    14) SIGALRM	    15) SIGTERM
16) SIGSTKFLT	17) SIGCHLD	    18) SIGCONT	    19) SIGSTOP	    20) SIGTSTP
21) SIGTTIN	    22) SIGTTOU	    23) SIGURG	    24) SIGXCPU	    25) SIGXFSZ
26) SIGVTALRM	27) SIGPROF	    28) SIGWINCH	29) SIGIO	    30) SIGPWR
31) SIGSYS	    34) SIGRTMIN	35) SIGRTMIN+1	36) SIGRTMIN+2	37) SIGRTMIN+3
38) SIGRTMIN+4	39) SIGRTMIN+5	40) SIGRTMIN+6	41) SIGRTMIN+7	42) SIGRTMIN+8
43) SIGRTMIN+9	44) SIGRTMIN+10	45) SIGRTMIN+11	46) SIGRTMIN+12	47) SIGRTMIN+13
48) SIGRTMIN+14	49) SIGRTMIN+15	50) SIGRTMAX-14	51) SIGRTMAX-13	52) SIGRTMAX-12
53) SIGRTMAX-11	54) SIGRTMAX-10	55) SIGRTMAX-9	56) SIGRTMAX-8	57) SIGRTMAX-7
58) SIGRTMAX-6	59) SIGRTMAX-5	60) SIGRTMAX-4	61) SIGRTMAX-3	62) SIGRTMAX-2
63) SIGRTMAX-1	64) SIGRTMAX
```

使用 `man 7 signal` 命令查看系统对每个信号作用的描述：

```
Signal      Standard   Action   Comment       ────────────────────────────────────────────────────────────────────────
SIGABRT      P1990      Core    Abort signal from abort(3)
SIGALRM      P1990      Term    Timer signal from alarm(2)
SIGBUS       P2001      Core    Bus error (bad memory access)
SIGCHLD      P1990      Ign     Child stopped or terminated
SIGCLD         -        Ign     A synonym for SIGCHLD
SIGCONT      P1990      Cont    Continue if stopped
SIGEMT         -        Term    Emulator trap
SIGFPE       P1990      Core    Floating-point exception
SIGHUP       P1990      Term    Hangup detected on controlling terminal
                                or death of controlling process
SIGILL       P1990      Core    Illegal Instruction
...
```

## 信号和中断

信号处理是一种典型的异步事件处理方式：进程需要提前向内核注册信号处理函数，当某个信号到来时，内核会就执行相应的信号处理函数。

我们知道，硬件中断也是一种内核异的步事件处理方式。当外部设备出现一个必须由 CPU 处理的事件，如键盘敲击、数据到达网卡等，内核会收到中断通知，暂时打断当前程序的执行，跳转到该中断类型对应的中断处理程序。中断处理程序是由 BIOS 和操作系统在系统启动过程中预先注册在内核中的。

中断和信号通知都是在内核产生。中断是完全在内核里完成处理，而信号的处理则是在用户态完成的。也就是说，内核只是将信号保存在进程相关的数据结构里面，在执行信号处理程序之前，需要从内核态切换到用户态，执行完信号处理程序之后，又回到内核态，再恢复进程正常的运行。

可以看出，中断和信号的严重程度不一样。信号影响的是一个进程，信号处理出了问题，最多是这个进程被干掉。而中断影响的是整个系统，一旦中断处理程序出了问题，可能整个系统都会挂掉。

## 信号处理

一旦有信号产生，进程对它的处理都有下面三个选择。

1. 执行缺省操作（Default）。Linux 为每个信号都定义了一个缺省的行为。例如，信号 SIGKILL 的缺省操作是 Term，也就是终止进程的意思。信号 SIGQUIT 的缺省操作是 Core，即终止进程后，通过 Core Dump 将当前进程的运行状态保存在文件里面。
2. 捕捉信号（Catch）。这个是指让用户进程可以注册自己针对这个信号的处理函数。当信号发生时，就执行我们注册的信号处理函数。
3. 忽略信号（Ignore）。当我们不希望处理某些信号的时候，就可以忽略该信号，不做任何处理。

有两个信号例外，对于 **SIGKILL** 和 **SIGSTOP** 这个两个信号，进程是无法捕捉和忽略，它们用于在任何时候中断或结束某一进程。**SIGKILL** 和 **SIGSTOP** 为内核和超级用户提供了删除任意进程的特权。

如果我们不想让信号执行缺省操作，可以对特定的信号注册信号处理函数：

```c

#include <signal.h>

typedef void (*sighandler_t)(int);

sighandler_t signal(int signum, sighandler_t handler);
```

例如下面的例子，程序捕获了信号 SIGINT ，并且只是输出不做其他处理，这样在键盘上按 `Ctrl+C` 并不能让程序退出：

```c

#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>

void sig_handler(int signo)
{
    if (signo == SIGINT) {
           printf("received SIGINT\n");
    }
}

int main(int argc, char *argv[])
{
    signal(SIGINT, sig_handler);

    printf("Process is sleeping\n");
    while (1) {
           sleep(1000);
    }
    return 0;
}
```

通过 `signal` 注册的信号处理函数，会保存在进程内核的数据结构 [task_struct](https://elixir.bootlin.com/linux/linux-5.17.y/source/include/linux/sched.h#L728) 中。由于信号都发给进程，并由进程在用户态处理，所以发送给进程的信号也保存在 [task_struct](https://elixir.bootlin.com/linux/linux-5.17.y/source/include/linux/sched.h#L728) 中。

![signal](https://raw.githubusercontent.com/mz1999/material/master/images/202204121515766.png)

[task_struct->sighand](https://elixir.bootlin.com/linux/linux-5.17.y/source/include/linux/sched.h#L1084) 和 [task_struct->signal](https://elixir.bootlin.com/linux/linux-5.17.y/source/include/linux/sched.h#L1083) 是线程组内共享，而 [task_struct->pending](https://elixir.bootlin.com/linux/linux-5.17.y/source/include/linux/sched.h#L1089) 是线程私有的。

`stask_struct->sighand` 里面有一个 `action`，这是一个数组，下标是信号，数组内容就是注册的信号处理函数。

`task_struct->pending` 内包含了一个链表，保存了本线程所有的待处理信号。`task_struct->signal->shared_pending` 上也有一个待处理信号链表，这个链表保存的是线程组内共享的信号。

## 常见信号

下面的列表列举了一些常见的信号。

| Singal      | Value | Action | comment                                                                 | key binding |
| ----------- | ----- | ------ | ----------------------------------------------------------------------- | ----------- |
| **SIGHUP**  | 1     | Term   | Hangup detected on controlling terminal or death of controlling process |             |
| **SIGINT**  | 2     | Term   | Interrupt from keyboard                                                 | **Ctrl-c**  |
| **SIGQUIT** | 3     | Core   | Quit from keyboard                                                      | **Ctrl-\\** |
| **SIGKILL** | 9     | Term   | Kill signal                                                             |             |
| **SIGSEGV** | 11    | Core   | Invalid memory reference                                                |             |
| **SIGPIPE** | 13    | Term   | Broken pipe: write to pipe with no readers                              |             |
| **SIGTERM** | 15    | Term   | Termination signal                                                      |             |
| **SIGCHLD** | 17    | Ign    | Child stopped or terminated                                             |             |
| **SIGCONT** | 18    | Cont   | Continue if stopped                                                     |             |
| **SIGSTOP** | 19    | Stop   | Stop process                                                            |             |
| **SIGTSTP** | 20    | Stop   | Stop typed at terminal                                                  | **Ctrl-z**  |
| **SIGTTIN** | 21    | Stop   | Terminal input for background process                                   |             |
| **SIGTTOU** | 22    | Stop   | Terminal output for background process                                  |             |

第一列是信号名称，第二列是信号编号。使用 `kill` 向进程发送信号时，用信号名称和编号都可以，例如：

```
kill -1 [pid]
kill -SIGHUP [pid]
```

Action 列是信号的缺省行为，主要有如下几个：

* **Term** 终止进程
* **Core** 终止进程并core dump
* **Ign** 忽略信号
* **Stop** 停止进程
* **Cont** 如果进程是已停止，则恢复进程执行

有一些信号在 TTY 终端做了键盘按键绑定，例如 `CTRL+c`  会向终端上运行的前台进程发送  SIGINT 信号。

### SIGHUP

运行在终端中，由 bash 启动的进程，都是 bash 的子进程。终端退出结束时会向 bash 的每一个子进程发送 **SIGHUP** 信号。由于 **SIGHUP** 的缺省行为是 Term，因此，即使运行在后台的进程也会和终端一起结束。

使用 `nohup` 命令可解决这个问题，它的作用是让进程忽略 SIGHUP 信号：

```
$ nohup command >cmd.log 2>&1 &
```

这样，即使我们退出了终端，运行在后台的程序会忽视 **SIGHUP** 信号而继续运行。由于作为父进程的 bash 进程已经结束，因此 `/sbin/init` 就成为孤儿进程新的父进程。

### SIGINT, SIGQUIT, SIGTERM 和 SIGKILL

**SIGTERM** 和 **SIGKILL** 是通用的终止进程请求，**SIGINT** 和 **SIGQUIT** 是专门用于来自终端的终止进程请求。他们的关键不同点是：**SIGINT** 和 **SIGQUIT** 可以是用户在终端使用快捷键生成的，而 **SIGTERM** 和 **SIGKILL** 必须由另一个程序以某种方式生成（例如通过 kill 命令）。

当用户按下 **ctrl-c** 时，终端将发送 **SIGINT** 到前台进程。 SIGINT 的缺省行为是终止进程（Term），但它可以被捕获或忽略。 信号 SIGINT 的目的是为进程提供一种有序、优雅的关闭机制。

当用户按下 **ctrl-\\** 时，终端将发送 **SIGQUIT** 到前台进程。 SIGQUIT 的缺省行为是终止进程并 core dump，它同样可以被捕获或忽略。 你可以认为 **SIGINT** 是用户发起的*愉快的终止*，而 **SIGQUIT** 是用户发起的*不愉快终止*，需要生成 Core Dump ，方便用户事后进行分析问题在哪里。

在 ubuntu 上由 [systemd-coredump](http://manpages.ubuntu.com/manpages/focal/man8/systemd-coredump.8.html) 系统服务处理 core dump。我们可以使用 [coredumpctl](http://manpages.ubuntu.com/manpages/focal/man1/coredumpctl.1.html) 命令行工具查询和处理 core dump 文件。

```
$ coredumpctl list
TIME                         PID  UID  GID SIG     COREFILE EXE           SIZE
Tue 2022-04-12 22:09:52 CST 6754 1000 1000 SIGQUIT present  /usr/bin/cat 17.1K
```

core dump 文件缺省保存在 `/var/lib/systemd/coredump` 目录下。

**SIGTERM** 默认行为是终止进程，但它也可以被捕获或忽略。**SIGTERM** 的目的是杀死进程，它允许进程有机会在终止前进行清理，优雅的退出。当我们使用 `kill` 命令时，SIGTERM 是默认信号。

**SIGKILL**  唯一的行为是立即终止进程。 由于 **SIGKILL** 是特权信号，进程无法捕获和忽略，因此进程在收到该信号后无法进行清理，立刻退出。

例如 docker 在停止容器的时候，先给容器里的1号进程发送 **SIGTERM**，如果不起作用，那么等待30秒后会会发送 **SIGKILL**，保证容器最终会被停止。

###  SIGSTOP 、 SIGTSTP 和 SIGCONT

**SIGSTOP** 和 **SIGTSTP** 这两个信号都是为了暂停一个进程，但 **SIGSTOP**  是特权信息，不能被捕获或忽略。

**SIGSTOP** 必须由另一个程序以某种方式生成（例如：kill -SIGSTOP pid），而**SIGTSTP** 也可以由用户在键盘上键入快捷键 `Ctrl-z` 生成。

被暂停的进程通过信号 **SIGCONT** 恢复。当用户调用 **fg** 命令时，**SIGCONT** 由 shell 显式发送给被暂停的进程。

Linux 使用他们进行作业控制，让你能够手动干预和停止正在运行的应用程序，并在未来某个时间恢复程序的执行。

### SIGTTOU 和 SIGTTIN

Linux 系统中可以有多个会话（**session**），每个会话可以包含多个进程组，每个进程组可以包含多个进程。

会话是用户登录系统到退出的所有活动，从登录到结束前创建的所有进程都属于这次会话。会话有一个前台进程组，还可以有一个或多个后台进程组。只有前台进程可以从终端接收输入，也只有前台进程才被允许向终端输出。如果一个后台作业中的进程试图进行终端读写操作，终端会向整个作业发送 **SIGTTOU** 或 **SIGTTIN** 信号，默认的行为是暂停进程。

## JVM 对信号的处理

如果你使用 `strace` 追踪 Java 应用，发现 Java 程序会抛出大量 `SIGSEGV`。

```
$ strace -fe 'trace=!all' java [app]
...
[pid 21746] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061680} ---
[pid 21872] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061480} ---
[pid 21943] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061500} ---
[pid 21844] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061780} ---
[pid 21728] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061c00} ---
[pid 21906] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061980} ---
[pid 21738] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061100} ---
[pid 21729] --- SIGSEGV {si_signo=SIGSEGV, si_code=SEGV_ACCERR, si_addr=0x7f9cbc061e00} ---
...
```

`SIGSEGV` 信号的意思是 “分段错误”（segmentation fault），是当系统检测到进程试图访问不属于它的内存地址时，内核向进程发送的信号。`SIGSEGV` 对于一般应用来说是很严重的错误，但 Java 进程中的 `SIGSEGV` 几乎总是正常和安全的。

在常规的 C/C++ 程序中，当你期望指针是指向某个结构，但实际指向的是 `NULL`，会导致应用程序崩溃。这种崩溃实际上是内核向进程发送了信号 `SIGSEGV`。如果应用程序没有为该信号注册信号处理程序，则信号会返回到内核，然后内核会终止应用。实际上 JVM 为 `SIGSEGV` 注册了一个信号处理程序，因为 JVM 想使用 `SIGSEGV` 和其他一些信号来实现自己的目的。

[这篇文档](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/signals006.html) 描述了 JVM 对信号的特殊处理：

| Signal                                             | Description                                                                                                                                                                                                               |
| :------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `SIGSEGV`, `SIGBUS`, `SIGFPE`, `SIGPIPE`, `SIGILL` | These signals are used in the implementation for implicit null check, and so forth.                                                                                                                                       |
| `SIGQUIT`                                          | This signal is used to dump Java stack traces to the standard error stream. (Optional)                                                                                                                                    |
| `SIGTERM`, `SIGINT`, `SIGHUP`                      | These signals are used to support the shutdown hook mechanism (`java.lang.Runtime.addShutdownHook`) when the VM is terminated abnormally. (Optional)                                                                      |
| `SIGUSR1`                                          | This signal is used in the implementation of the `java.lang.Thread.interrupt` method. Not used starting with Oracle Solaris 10 reserved on Linux. (Configurable)                                                          |
| `SIGUSR2`                                          | This signal is used internally. Not used starting with Oracle Solaris 10 operating system. (Configurable)                                                                                                                 |
| `SIGABRT`                                          | The HotSpot VM does not handle this signal. Instead it calls the `abort` function after fatal error handling. If an application uses this signal then it should terminate the process to preserve the expected semantics. |

实际上，JVM 是使用 SIGSEGV、SIGBUS、SIGPIPE 等进行代码中的各种 `NULL` 检查。

同样，我们在终端上键入 **ctrl-\\**，也不会让前台运行的 Java 进程终止并 core dump，而是会将 Java 进程的 stack traces 输出到终端的标准错误流。

那么如何对 Java 进程进行 core dump 呢？需要在 Java 的启动命令里增加 JVM 选项 `-Xrs` ，它会让 JVM 不自己处理 SIGQUIT 信号，这样 SIGQUIT 会触发缺省行为 core dump。

一般 Java 进程的运行时内存占用都比较大，在进行 core dump 时很容易超过缺省大小而被truncated，因此需要修改配置文件 [/etc/systemd/coredump.conf](https://man7.org/linux/man-pages/man5/coredump.conf.5.html)，合理设置 ProcessSizeMax 和 ExternalSizeMax 的大小。
