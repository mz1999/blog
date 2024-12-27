# 从源码构建 Linux 内核

![title](https://cdn.mazhen.tech/2024/202412271447033.jpg)

对于希望深入了解 Linux 内核的开发者来说，亲自编译和安装内核是一次有趣的实践。在这篇文章中，我将带你从环境准备开始，逐步完成下载内核源代码、配置、编译和构建自定义内核，最终安装并使用它。让我们开始吧。

## 构建环境准备

在准备构建环境之前，我们需要先理解 **native toolchain (本地工具链)** 和 **cross-toolchain (交叉工具链)** 这两个重要的概念。

工具链 (toolchain) 是编译软件所需的一系列工具集合，包括 `make` 构建工具、`gcc` 编译器、标准 GNU C 库 (`glibc`)、`binutils` (包含链接器和汇编器)、`gdb` 调试器等。

**native toolchain (本地工具链)** 指的是运行在特定架构上，用于为**相同架构**编译代码的工具集合，例如在 `x86_64` 架构的 Linux 系统上，使用系统自带的 `gcc` 编译出的程序，可以直接本地运行。

**cross-toolchain (交叉工具链)** 则是在一个架构上运行，用于为**另一个不同架构**的系统构建软件的工具链。例如在 `x86_64` 架构的 Linux 系统上，如果需要为 `ARM` 架构的设备构建程序，就需要使用一个针对 `ARM` 架构的 **cross-toolchain**。

本文将讨论在 `x86_64` 架构上为本机构建内核的情况，没有涉及交叉编译。在本机为另一个目标平台构建内核在嵌入式开发领域很常见，感兴趣的读者可以参考这篇文档：[如何交叉编译为树莓派构建内核](https://www.raspberrypi.com/documentation/computers/linux_kernel.html)。

我选择常用的 Linux 发行版 Ubuntu 作为示例进行说明。针对其他发行版，以下步骤和原理基本相同，只是具体的命令可能略有差异。

### 安装本地工具链和开发工具

执行下面的命令安装构建内核所需本地工具链和开发工具：

```bash
$ sudo apt install git vim build-essential perl gdb dwarves linux-headers-$(uname -r)
```

其中，`build-essential` 是一个用于构建 Debian 软件包的基础工具集，它包含了编译软件包所需的关键工具和库，例如编译器、构建工具以及开发头文件。我们可以使用 `apt-cache depends` 命令来查看 `build-essential` 包所包含的内容：

```bash
$ apt-cache depends build-essential
build-essential
 |Depends: libc6-dev
  Depends: <libc-dev>
    libc6-dev
  Depends: gcc
  Depends: g++
  Depends: make
    make-guile
  Depends: dpkg-dev
```

`dwarves` 包包含了一组高级的 DWARF 工具，主要用于处理由编译器插入到 ELF 二进制文件中的 DWARF 调试信息。这些调试信息对于调试内核模块至关重要。

`$(uname -r)` 命令用于动态获取当前正在运行的内核的版本号。 `linux-headers-$(uname -r)` 则代表与当前运行的 Linux 内核版本相对应的内核头文件。这些头文件是编译与内核模块相关的软件所必需的。

### 安装构建内核需要的依赖

执行下面的命令安装构建内核需要的依赖包：

```bash
$ sudo apt install \
	bison flex libncurses5-dev ncurses-dev \
	libelf-dev libssl-dev bc zstd
```

下面详细解释这些软件包的作用：

- **bison**：是一个语法分析器生成器。它能将编程语言的语法规则转换为可解析这些规则的程序，即生成语法分析器。从 Linux 4.16 版本开始，构建系统会在构建过程中自动生成解析器，这需要 Bison 2.0 或更高版本。
- **flex**：是一个快速的词法分析器生成器。它用于生成对文本进行模式匹配的程序，即生成词法分析器。词法分析器的作用是对输入文本进行扫描，识别出符合特定规则的字符序列。自 Linux 4.16 版本起，构建系统也会在构建过程中自动生成词法分析器，因此需要 Flex 2.5.35 或更高版本。
- **ncurses**： (全称 "new curses") 是一个编程库，提供应用程序编程接口（API），允许开发者在终端中创建与终端类型无关的基于文本的用户界面（TUI）。简而言之，它是开发“类 GUI”应用程序的软件工具包，这些应用程序在终端模拟器中运行，不依赖图形界面。`libncurses5-dev` 和 `ncurses-dev` 这两个软件包都是开发基于 `ncurses` 的文本用户界面程序的开发包，它们包含了 `ncurses` 库的头文件和开发文件，用于编译和链接基于 `ncurses` 的程序。
- **libelf-dev**：提供了用于处理 ELF (Executable and Linkable Format，可执行与可链接格式) 文件的开发库。ELF 是一种常见的文件格式，用于可执行文件、目标代码、共享库和核心转储文件。`libelf` 库使开发者能够以与架构无关的方式读取、修改或创建 ELF 文件，同时处理文件大小和字节序（Endian）等问题。Linux 内核和内核模块都使用 ELF 文件格式，因此在编译和调试内核时，`libelf` 库是必不可少的。
- **libssl-dev**：是 OpenSSL 项目实现 SSL（Secure Sockets Layer）和 TLS（Transport Layer Security）加密协议的开发包。`libssl-dev` 包含了 OpenSSL 的头文件和开发库，供开发者在程序中使用这些加密协议来进行加密、解密、证书验证、密钥交换等操作。文档指出，从 Linux 内核 v4.3 版本（如果启用模块签名，则从 v3.7 版本开始）及更高版本，需要安装 OpenSSL 开发包。
- **bc**：是一种任意精度的数字处理语言。它支持交互式执行语句，常用于执行精确的数学计算。在内核构建过程中，`bc` 用于在头文件中生成时间常数。
- **zstd**：是一种高效的压缩算法工具，具有更好的压缩率和压缩速度。在内核构建过程中，构建成功的未压缩的内核镜像 `vmlinux` 会使用 ZSTD 算法压缩为 `bzImage`，这是一种可启动的内核镜像格式。

可以参考内核文档 [Minimal requirements to compile the Kernel](https://www.kernel.org/doc/html/latest/process/changes.html#minimal-requirements-to-compile-the-kernel)，列出了编译当前版本 Linux 内核所需的最低软件要求。

## 获取内核源码

获取 Linux 内核源码主要有两种方式：

1. 从 [www.kernel.org](https://www.kernel.org) 下载并解压特定版本的内核源码压缩包。
2. 使用 `git clone` 命令从 Git 仓库克隆内核源码。

### 下载特定版本的内核源码

访问 [www.kernel.org](https://www.kernel.org)，该网站首页会展示多个类型的内核源码，包括：

- 主线版（**mainline**）， 开发阶段的版本，包含最新的功能和改动。**mainline** 由 [Linus Torvalds](https://en.wikipedia.org/wiki/Linus_Torvalds) 亲自维护。
- 稳定版（**stable**），在 **mainline** 的基础上进行修复和测试，更加稳定可靠，适合一般用户使用。
- 长期支持版（**longterm**）会长期维护，修复 bug 并保持安全，适合一些有长期支持需求的场景。
- **linux-next** 版本用于测试和集成即将进入主线的特性。

关于这些内核版本的具体说明，可以参考官方文档 [Active kernel releases](https://www.kernel.org/releases.html)，

这里我们选择当前最新的稳定版 **6.12.6** 为例。点击相应的 `tarball` 链接下载：

![kernel.org](https://cdn.mazhen.tech/2024/202412231059439.png)

浏览器会将 .tar.xz 格式的压缩文件下载到本地。下载完成后，可以使用 `tar` 命令解压：

```bash
$ tar -Jxvf linux-6.12.6.tar.xz
$ cd linux-6.12.6
$ ls -l
```

###  从 git 仓库 clone 源码

除了下载压缩包，还可以使用 Git 从内核仓库克隆源码。如前所述，Linux 内核有多个类型的源码仓库。

如果想获取包含最新功能的 mainline 源码仓库，执行下面的命令：

```bash
$ git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
```

这个命令会克隆 Linus Torvalds 亲自维护的 Linux 内核 mainline 仓库。

我们的需求是获取最新稳定版本的源码仓库，则需要 clone 稳定版的源码仓库：

```bash
$ git clone git://git.kernel.org/pub/scm/linux/kernel/git/stable/linux-stable.git
$ cd linux-stable
$ git checkout v6.12.6
```

这里，我们先克隆稳定版仓库，然后使用 `git checkout` 命令切换到 `v6.12.6` 标签。

克隆完整的 Linux 内核源码通常非常耗时，特别是在网络环境不佳的情况下。为了加快克隆速度，可以使用 `depth` 参数来限制历史提交的深度，从而减少下载的数据量。例如：

```bash
git clone --depth 1 <...>
```

其中 `--depth 1` 表示只下载最新的提交记录，可以显著减少下载时间。

如果只是想基于最新稳定版的源码构建内核，那么还是推荐第一种方法，直接从 kernel.org 下载内核源码的压缩包。

### 内核源码结构概览

内核源码按照子系统和功能，被组织成目录及其子目录，这种结构化的设计使得内核的维护和开发更加高效。下面我们以鸟瞰的方式，对内核源码的整体结构进行一个粗略的认识：

| 文件或目录     | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| README         | 项目的 README 文件，提供了关于内核的简要说明，以及如何访问最新的官方内核文档的链接：https://www.kernel.org/doc/html/latest/。 |
| COPYING        | 内核源代码的许可条款                                         |
| MAINTAINERS    | 内核子系统的维护者列表，提供了维护者的联系方式，是贡献代码的重要参考。 |
| Makefile       | 内核顶层的 Makefile                                          |
| kernel/        | 核心子系统，包含了进程和线程的生命周期管理、CPU 任务调度、锁机制、cgroups (控制组)、定时器、中断处理、信号、内核模块机制、tracing (跟踪)、eBPF 等核心功能。 |
| mm/            | 内存管理子系统，负责内核的内存分配、页面管理、交换等功能。   |
| fs/            | 内核虚拟文件系统 (VFS) 和各个具体文件系统的实现，如 ext4, btrfs, overlayfs 等等。 |
| block/         | 块设备 I/O 实现，包含了页面缓存、通用块设备 I/O 层、I/O 调度等，负责管理磁盘和其他块设备的数据读写。 |
| net/           | 网络协议栈的实现，包含了 TCP, UDP, IP 等网络协议的实现。     |
| ipc/           | 进程间通信 (IPC) 子系统，提供了进程间交换数据的机制，例如信号量、共享内存、消息队列等。 |
| sound/         | 音频子系统，提供了对音频设备的支持。                         |
| virt/          | 虚拟化子系统，包含了 KVM (Kernel-based Virtual Machine) 的实现，是 Linux 系统支持虚拟化的关键组件。 |
| Documentation/ | 内核官方文档，包含了关于内核各个子系统的详细说明             |
| LICENSES/      | 内核代码遵循的所有许可证。                                   |
| arch/          | 体系结构相关的代码，例如 `arm`, `x86`，`riscv` 等，包含了特定于不同 CPU 架构的内核实现。 |
| certs/         | 用于生成签名模块的代码，用于确保内核模块的安全性。           |
| crypto/        | 内核实现的加密和解密算法，为内核提供安全加密功能。           |
| drivers/       | 设备驱动程序代码，包含了各种硬件设备的驱动程序，例如显卡、网卡、USB 设备等。 |
| include/       | 架构无关的内核头文件，包含了内核编程所需的各种数据结构和函数声明。特定架构的头文件位于 `arch/<cpu>/include/` 目录下。 |
| init/          | 内核初始化代码，包含了内核启动时的核心代码。内核主函数 `start_kernel()` 定义在 `init/main.c` 中，是内核启动的入口点。 |
| io_uring/      | io_uring 的实现，是一个高性能的异步 I/O 框架，用于提高 I/O 性能。 |
| lib/           | 类似于用户态应用的共享库 `glibc`，但这是内核代码所使用的库，提供了一些常用的内核数据结构和辅助函数。 |
| rust/          | 支持 Rust 编程语言的内核基础设施，用于在内核中使用 Rust 语言编写模块。 |
| samples/       | 各种内核特性的示例代码，可以帮助开发者理解和使用内核 API。   |
| scripts/       | 各种有用的脚本，其中一些用于内核构建过程中，例如配置工具、编译脚本等。 |
| security/      | Linux 安全模块 (LSM) 的实现，包括 SELinux, AppArmor 等，提供了内核安全机制。 |
| tools/         | 用户态工具的源代码，例如 `perf` 和 eBPF 的用户态工具         |
| usr/           | 用于生成和加载 initramfs 镜像，initramfs 是一个小的文件系统，内核在初始化阶段利用 initramfs 执行用户空间代码。 |

### 关于 MAINTAINERS 文件

在内核源码的根目录，有一个名为 [MAINTAINERS](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/MAINTAINERS) 的重要文件。该文件详细列出了所有内核子系统的维护者，以及他们的联系方式，包括邮件列表、代码仓库位置、网站等信息。

随着内核的不断发展，源代码行数已经非常庞大，目前估计接近 3000 万行。即使是 Linus Torvalds 本人，也无法完全掌握所有细节。因此，如果想要向内核上游贡献补丁，通常不会直接提交到 mainline。Linus 不可能对每个补丁进行详细的审查和合并。`MAINTAINERS` 文件是内核贡献者了解内核组织结构、找到对应维护者以及提交补丁的关键入口。

实际上，大多数内核子系统都有自己独立的源码仓库。补丁的讨论、提交和合并过程通常发生在这些子系统的源码仓库中，由该子系统的维护者负责把关。子系统的维护者会定期将已经合并的补丁向 mainline 提交。Linus 信任这些维护者，通常会直接合并他们提交的补丁。这就是内核社区的“信任链”模式。由此可见，维护者在内核开发中扮演着至关重要的角色，他们是内核质量的守护者，也是社区活跃的重要力量。

今年 10 月发生了一件引人注目的事件，即[移除俄罗斯维护者事件](https://lore.kernel.org/all/2024101835-tiptop-blip-09ed@gregkh/)。实际上，该事件是将邮件后缀为 `ru` 的维护者从 `MAINTAINERS` 文件中移除。我们可以在 mainline 源码仓库中看到当时的提交记录：[MAINTAINERS: Remove some entries due to various compliance requirements.](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/MAINTAINERS?id=6e90b675cf942e50c70e8394dfb5862975c3b3b2)。

关于内核的开发流程，以及如何向上游提交补丁，可以参考内核文档 [A guide to the Kernel Development Process](https://www.kernel.org/doc/html/latest/process/development-process.html)。该文档详细介绍了内核贡献的各个环节，包括代码风格、补丁格式、提交方式等等，是每个想参与内核贡献的开发者必读资料。

## 配置内核

配置是构建内核的关键一步。通过配置，我们可以基于统一的源码，构建出适用于服务器、桌面或嵌入式等不同场景的内核。内核的强大之处在于其高度的可定制性，可以根据不同的硬件平台和应用需求进行裁剪和优化。

### Kconfig

内核的可配置项定义在一系列 **Kconfig** 文件中，每个 **Kconfig**  文件中定义了多个 `config` 项，表示内核编译选项。这些选项决定了内核最终编译包含哪些功能和特性。通过配置这些选项，可以灵活定制内核以满足不同的需求。

**Kconfig**  文件分布在源码的各个子目录。如前所述，内核源码是按照子系统和功能组织的，因此每个目录下的 **Kconfig** 文件通常定义了该目录所实现功能的相关配置选项。例如，`mm/Kconfig` 文件定义了与内存管理子系统相关的配置选项，而 `drivers/net/ethernet/Kconfig` 则定义了以太网驱动相关的配置选项。源码根目录下的 **Kconfig** 文件通过 `source` 指令引用各个子系统的 **Kconfig** 文件，而各个子系统的 **Kconfig** 文件也会通过 `source` 引用其子目录中的 **Kconfig** 文件。这种层层引用的方式，使得内核的所有可配置项按照层次结构组织起来，方便管理和维护。

**Kconfig** 文件使用 [Kconfig 语法](https://www.kernel.org/doc/html/latest/kbuild/kconfig-language.html#kconfig-language) 来定义配置项的名称、类型、依赖关系等。我们以一个具体的配置项为例，简单了解一下[Kconfig 语法](https://www.kernel.org/doc/html/latest/kbuild/kconfig-language.html#kconfig-language)。打开 `mm/Kconfig`文件，找到配置项 **ZSWAP**：

```kconfig
config ZSWAP
        bool "Compressed cache for swap pages"
        depends on SWAP
        select CRYPTO
        select ZPOOL
        help 
          A lightweight compressed cache for swap pages.  It takes
          pages that are in the process of being swapped out and attempts to
          compress them into a dynamically allocated RAM-based memory pool.
          This can result in a significant I/O reduction on swap device and,
          in the case where decompressing from RAM is faster than swap device
          reads, can also improve workload performance.
```

下面的表格解释了配置项 **ZSWAP** 每一行的含义：

| Item                                   | 描述                                                         |
| -------------------------------------- | ------------------------------------------------------------ |
| config ZSWAP                           | 定义了配置项的名称为 `ZSWAP`                                 |
| bool "Compressed cache for swap pages" | 指定配置项为布尔类型                                         |
| depends on SWAP                        | 定义了此配置依赖的配置项，即只有当 `SWAP` 配置项被启用时， `ZSWAP` 配置项才会被显示出来并可以被配置。 |
| select CRYPTO<br/>select ZPOOL         | 定义了反向依赖关系，即依赖此配置项的其他配置项               |
| help                                   | 配置项的帮助信息，描述了 `ZSWAP` 的功能和使用方法。          |

内核构建系统可以读取并解析所有 **Kconfig**  文件，并以可视化的方式展示出来。

对内核的配置过程，本质上就是从所有 **Kconfig** 文件中选择想要设置的配置项，并将最终的配置结果记录在源码根目录下的 **.config** 文件中。

内核提供了多种配置方式。可以在源码根目录运行 `make help` 查看配置目标，它们位于 `Configuration targets` 标题下：

![kconfig](https://cdn.mazhen.tech/2024/202412241808723.png)

接下来，我们会选择其中几个常用的目标进行介绍。

### 默认配置

最新的 Linux 内核源代码已经接近 3000 万行，其配置项之庞大复杂可想而知。我们可以在源码的根目录下执行以下脚本，统计当前内核的可配置项数量：

```bash
$ find . -name "Kconfig*" -print0 \      
| xargs -0 grep -P '^\s*config\s+\w+' \
| wc -l 
20961
```

从零开始配置两万多个配置项简直是噩梦。好在内核已经准备了默认配置。在前面列出的配置目标中，有一个 `defconfig` 目标：

```shell
  defconfig	 - New config with default from ARCH supplied defconfig
```

其含义是：基于 CPU 架构提供的默认配置，创建一个新的配置。

内核为每一种 CPU 架构都提供了一个默认配置，这些配置存放在 `arch/<cpu>/configs` 目录下。运行 `defconfig` 目标：

```bash
$ make defconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  HOSTCC  scripts/kconfig/confdata.o
  HOSTCC  scripts/kconfig/expr.o
  LEX     scripts/kconfig/lexer.lex.c
  YACC    scripts/kconfig/parser.tab.[ch]
  HOSTCC  scripts/kconfig/lexer.lex.o
  HOSTCC  scripts/kconfig/menu.o
  HOSTCC  scripts/kconfig/parser.tab.o
  HOSTCC  scripts/kconfig/preprocess.o
  HOSTCC  scripts/kconfig/symbol.o
  HOSTCC  scripts/kconfig/util.o
  HOSTLD  scripts/kconfig/conf
*** Default configuration is based on 'x86_64_defconfig'
#
# configuration written to .config
#
```

可以看到，`make defconfig` 会基于当前 x86_64 架构的默认配置 `x86_64_defconfig`，生成最终的配置结果 `.config`。你可以使用 vi 打开 `.config` 查看详细的配置信息：

```shell
#
# Automatically generated file; DO NOT EDIT.
# Linux/x86 6.12.6 Kernel Configuration
#
CONFIG_CC_VERSION_TEXT="gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0"
CONFIG_CC_IS_GCC=y
CONFIG_GCC_VERSION=130300
CONFIG_CLANG_VERSION=0
CONFIG_AS_IS_GNU=y
CONFIG_AS_VERSION=24200
CONFIG_LD_IS_BFD=y
...
# CONFIG_KERNEL_BZIP2 is not set
...
CONFIG_NF_LOG_SYSLOG=m
...
```

`.config` 文件中的每一行都表示一个配置项，有以下几种形式：

- **`CONFIG_NAME=y`**: 表示该配置项被启用，并直接编译进内核。
- **`CONFIG_NAME=m`**: 表示该配置项被启用，但会编译为可加载的内核模块。
- **`# CONFIG_NAME is not set`**: 表示该配置项未被启用。
- **`CONFIG_NAME="text"`**: 表示该配置项的值是一个文本字符串，通常用于设置内核版本、模块名称等文本信息，例如`CONFIG_LOCALVERSION="-my-kernel"`。
- **`CONFIG_NAME=number`**: 表示该配置项的值是一个数字，通常用于设置内核参数、缓冲区大小等数值信息，例如 `CONFIG_NR_CPUS=8` 表示 CPU 的核数。

注意，正如 `.config` 文件开头所强调的，该文件是自动生成的，切勿手动修改。

### 基于现有发行版的配置

另一种简便的配置方法是：基于现有发行版的内核配置。这种配置方式与默认配置一样简单，但通常比直接使用默认配置更好。因为发行版的内核配置通常由专业的工程师进行裁剪和优化，并经过了厂商的充分测试，更能适应实际的应用场景。

以我当前的构建环境为例，我使用的是 Ubuntu 24 server 版：

```shell
$ cat /etc/lsb-release        
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=24.04
DISTRIB_CODENAME=noble
DISTRIB_DESCRIPTION="Ubuntu 24.04.1 LTS"
```

在重新配置之前，运行 `make mrproper` 清理所有构建过程生成的内容，包括已存在的 `.config` 文件。这确保我们从一个干净的状态开始：

```shell
$ make mrproper  
  CLEAN   scripts/basic
  CLEAN   scripts/kconfig
  CLEAN   include/config include/generated .config
```

然后运行 **oldconfig** 目标：

```shell
make oldconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
...
#
# using defaults found in /boot/config-6.8.0-51-generic
#
.config:965:warning: symbol value '0' invalid for BASE_SMALL
.config:10817:warning: symbol value 'm' invalid for ANDROID_BINDER_IPC
.config:10818:warning: symbol value 'm' invalid for ANDROID_BINDERFS
*
* Restart config...
*
*
* General setup
*
Compile also drivers which will not load (COMPILE_TEST) [N/y/?] n
Compile the kernel with warnings as errors (WERROR) [N/y/?] n
Local version - append to kernel release (LOCALVERSION) [] 
Automatically append version information to the version string (LOCALVERSION_AUTO) [N/y/?] n
...
```

可以看到，`make oldconfig` 会基于 `/boot/config-6.8.0-51-generic` 文件进行配置。在安装和升级内核时，内核镜像 `vmlinuz-version-EXTRAVERSION` 默认存储在 `/boot` 目录下，同时该内核对应的配置文件也会存放在 `/boot` 目录。`make oldconfig` 正是使用了当前运行内核所对应的配置。

由于正在构建的最新内核可能新增或修改了配置项，与当前内核的配置存在差异，因此接下来需要用户逐条设置必要的配置项。通常情况下，一路回车，接受默认选项即可，最终会生成 `.config` 文件。

你可能会觉得，对于差异的配置项，逐条确认默认选项比较繁琐。有没有办法一次性确认使用默认选项呢？当然可以，使用 **olddefconfig** 目标。

```shell
$ make mrproper 
$ make olddefconfig
  HOSTCC  scripts/basic/fixdep
  HOSTCC  scripts/kconfig/conf.o
  HOSTCC  scripts/kconfig/confdata.o
...
#
# using defaults found in /boot/config-6.8.0-51-generic
#
.config:965:warning: symbol value '0' invalid for BASE_SMALL
.config:10817:warning: symbol value 'm' invalid for ANDROID_BINDER_IPC
.config:10818:warning: symbol value 'm' invalid for ANDROID_BINDERFS
#
# configuration written to .config
#
```

与 **oldconfig** 类似，**olddefconfig** 也会基于当前内核的配置生成 `.config` 文件。不同之处在于，使用 **olddefconfig** 时，对于新增的配置项，会直接设置为默认值，无需用户逐个确认。

### 基于当前加载的内核模块

基于现有发行版的内核配置生成 `.config` 文件虽然非常方便，但如果你觉得发行版预置的配置有些臃肿，启用了过多的模块或内置功能，那么你可以考虑使用另一种方式：基于当前加载到内存中的内核模块，来生成自定义的配置。这种配置方式通常比发行版的默认配置更为紧凑，因为它只包含你当前系统实际使用的模块和功能。

`lsmod` 命令可以列出当前驻留在内存中的所有内核模块。我们可以将 `lsmod` 命令的输出提供给内核构建系统，让构建系统基于当前系统正在运行的内核模块生成配置：

```shell
$ lsmod > /tmp/lsmod.now
$ make LSMOD=/tmp/lsmod.now localmodconfig
```

这里，我们将 `lsmod` 命令的输出保存到一个临时文件 `/tmp/lsmod.now` 中，然后通过 **LSMOD** 环境变量传递给 Makefile 的 `localmodconfig` 目标。这样，内核构建系统就能基于内存中实际加载的内核模块，生成最终的配置 `.config` 文件。

由于这种配置方式比较常用，内核还提供了类似于 `localmodconfig` 目标的辅助脚本 `scripts/kconfig/streamline_config.pl`。该脚本会自动检查系统当前加载的模块，然后基于这些模块生成精简的内核配置。你可以使用以下方式来生成配置：

```shell
$ scripts/kconfig/streamline_config.pl
$ make olddefconfig
```

首先运行脚本 `streamline_config.pl` 生成 `.config` 文件，然后再运行 `make olddefconfig`，为新版本的内核可能新增的配置项设置默认值。

### 使用图形界面配置

内核构建系统还提供了图形界面的配置方式。通常，我们会先使用前述的方法生成一个基础配置，然后使用图形界面进行一些微调。在命令行运行 `make menuconfig`，构建系统会编译并执行 `scripts/kconfig/mconf` 程序，从而启动一个图形化的配置界面：

![menuconfig](https://cdn.mazhen.tech/2024/202412251521070.png)

让我们简单介绍一下这个界面的元素：

* 方括号 `[]` 表示布尔类型的选项
  * `[*]` 表示启用该特性，编译到内核镜像中
  * `[]` 表示关闭该特性
* 尖括号 `<>` 具有三种状态
  * `<*>` 表示启用该特性，编译到内核镜像中
  * `<M>` 表示启用该特性，编译为内核模块
  * `<>` 表示关闭该特性
* `-*-`  表示由于依赖要求，此特性必须被启用
* `{M|*}` 表示由于依赖要求，此特性必须编译到内核镜像中（*），或者编译为内核模块（M）
* `(...)` 表示需要输入字符或数字。在此选项上按下回车键，将弹出一个输入提示框。
* `<...> --->` 表示子菜单，按下回车键可进入该子菜单。

在界面上，使用**上、下方向键**在不同的选项之间进行导航，使用**左、右方向键**在屏幕下方的菜单 `<Select>`、`<Exit>` 等之间进行导航。在当前选中的选项上，导航到 `<Help>` 菜单并按下回车键，会显示该配置项的帮助信息：

![menuconfig](https://cdn.mazhen.tech/2024/202412251603485.png)

帮助信息界面会显示当前配置项的名称、类型、依赖关系等，其中还包括了该配置项具体定义在哪个 `Kconfig` 文件中。这些信息与我们在 `Kconfig` 文件中看到的内容是一致的。

在当前高亮的配置项上按**空格键**可以修改其值。

下面我们尝试使用图形界面设置几个配置项：

1. **CONFIG_LOCALVERSION**

配置项 `CONFIG_LOCALVERSION` 的作用是在内核版本号的末尾附加一个自定义的字符串。

我们先简单了解一下 Linux 内核的版本号命名规则：

```shell
major.minor[.patchlevel][-EXTRAVERSION]
```

* major：主版本号
* minor：次版本号，隶属于主版本号
* patchlevel：修订版本号，通常用于修复重大错误和安全问题
* EXTRAVERSION：由内核发行版指定，用于跟踪内部修改

可以使用 `uname -r` 命令查看当前内核的版本信息。以我当前的系统为例：

```shell
$ uname -r
6.8.0-51-generic
```

其中，前面三段 `6.8.0` 分别为主版本号、次版本号和修订版，最后一部分 `-51-generic` 是 `EXTRAVERSION`。

配置项 `CONFIG_LOCALVERSION` 的导航路径为 **General setup -> Local version - append to kernel release**。选中该选项并按下回车键，会出现输入提示框，将值修改为你想要设置的内容，例如 `-apusic-kernel`，可以将公司名称嵌入到版本号中，以标识该内核发行版的厂商。

2. **CONFIG_IKCONFIG** 和 **CONFIG_IKCONFIG_PROC**

配置项 `CONFIG_IKCONFIG` 的作用是控制在构建内核时，是否将 `.config` 文件保存到内核映像中。如果启用此选项，那么可以使用脚本 `scripts/extract-ikconfig` 从内核镜像文件中提取到所有的配置信息。

配置项 `CONFIG_IKCONFIG_PROC` 的作用是，如果启用此选项，则可以通过 `/proc/config.gz` 访问内核的配置文件。

`CONFIG_IKCONFIG` 的导航路径为 **General setup -> Kernel .config support**。选中该选项并使用空格键将其值修改为 `<*>`。这时下方会出现新的选项 **Enable access to .config through /proc/config.gz (NEW)**，再次使用空格键激活此选项。

3. **CONFIG_HZ_250**

当前内核使用四个不同的配置项，分别代表不同的定时器中断频率：

- **CONFIG_HZ_100**：100Hz，即每秒 100 次中断。较低的频率可能更适合服务器、SMP 和 NUMA 系统。服务器通常不需要像桌面计算机那样快速响应用户交互，而是需要高效地处理大量后台任务。
- **CONFIG_HZ_250**：250Hz，即每秒 250 次中断。这是一种折衷选择，既能保证服务器性能，又能在 SMP 和 NUMA 系统上表现出良好的交互响应。
- **CONFIG_HZ_300**：300Hz，即每秒 300 次中断。与 250Hz 类似，也是一种折衷选择，并且能精确适配 PAL 制和 NTSC 制的帧率，因此非常适合视频和多媒体工作。因为 PAL 制的帧率为 50Hz，NTSC 制的帧率为 60Hz，300Hz 可以被这两种制式的帧率整除，有利于视频播放和同步，减少视频处理过程中的误差和抖动。
- **CONFIG_HZ_1000**：1000Hz，即每秒 1000 次中断。这种高频率通常用于需要快速响应用户交互的系统，如桌面计算机。

首先进入子菜单 **Processor type and features**，找到 **Timer frequency** 选项并按下回车键进入。然后在弹出的选项中选择你想要设置的频率。这里我选择设置为 250Hz。

完成设置后，使用左右方向键导航到 `<Exit>`，一路退出，最后选择保存设置。这样，我们就完成了使用图形界面对内核进行配置。

### 使用脚本配置

除了使用图形界面进行配置，内核构建系统还提供了一个 Bash 脚本 `scripts/config`，允许我们以非交互的方式完成配置。这在需要自动化配置，或者需要在脚本中批量设置配置项时非常有用。

例如，在 Ubuntu 系统上构建内核时，可能会遇到[证书问题](https://askubuntu.com/questions/1329538/compiling-kernel-5-11-11-and-later/1329625)。这时，我们可以使用 `scripts/config` 脚本，将以下两个配置项设置为空字符串：

```shell
scripts/config --set-str SYSTEM_REVOCATION_KEYS ""
scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
```

`scripts/config` 脚本提供了多种选项，可以用于设置不同类型的配置值。通过 `--set-str` 选项，我们可以将配置项的值设置为指定的字符串。除了设置字符串类型的值，`scripts/config` 还支持其他操作，例如：

- `--enable <CONFIG_NAME>`: 启用一个配置项。
- `--disable <CONFIG_NAME>`: 禁用一个配置项。

例如前面我们通过图形界面启用的`CONFIG_IKCONFIG` 和 `CONFIG_IKCONFIG_PROC` 配置项，等效于使用下面的命令：

```shell
$ scripts/config --enable IKCONFIG --enable IKCONFIG_PROC
```

至此，我们已经学习了内核的多种配置方式，接下来就可以开始构建内核了。

## 构建内核镜像和内核模块

Linux 内核采用了递归式的 `make` 构建方式。在内核源码的根目录下，有一个顶层的 `Makefile` 文件，该文件会递归地解析嵌入在各个子目录中的 `Makefile` 文件。通过运行 `make help` 命令，我们可以查看默认情况下 `all` 目标会构建哪些内容：

```shell
$ make help
...
Other generic targets:
  all		 - Build all targets marked with [*]
* vmlinux	 - Build the bare kernel
* modules	 - Build all modules
...
Architecture-specific targets (x86):
* bzImage		- Compressed kernel image (arch/x86/boot/bzImage)
```

从输出中可以看到，执行 `make all` 命令会构建前面标记为 `*` 的目标，包括：

- **vmlinux** 目标：构建出未压缩的内核镜像文件 **vmlinux**。
- **modules** 目标：在内核配置中被标记为 `M` 的选项，都会被构建为内核模块（`.ko` 文件）。
- **bzImage** 目标：构建出被压缩过的内核镜像文件 **bzImage** 。

在系统启动过程中，真正被使用的是压缩过的内核镜像。引导程序会将它加载到内存中，并在内存中解压，然后引导系统进入内核。`vmlinux` 是未压缩的内核映像，其中包含了所有内核符号等额外的调试信息，虽然它不会被直接用于系统启动，但在进行内核调试时会用到，所以它仍然非常重要。

在内核源码的根目录下执行 `make` 命令，默认会执行 `all` 目标，即等同于执行 `make all` 命令，这样就可以构建出内核镜像和内核模块。

由于 Linux 内核源码库非常庞大，构建内核是一项非常消耗内存和 CPU 资源的任务。为了加快构建速度，`make` 工具支持多进程并行处理。我们可以使用 `-jn` 选项，生成多个进程，并行处理构建过程中相互独立的任务。其中，`n` 表示可以并行生成的任务数量的上限，通常可以根据下面的经验公式确定 `n` 的值：

```bash
n = number-of-CPU-cores * factor;
```

`factor` 是一个系数，一般选择为 2。

那么，如何知道当前系统的 CPU 核数呢？可以使用 `nproc` 命令：

```bash
$ nproc
12
```

`lscpu` 命令也可以显示 CPU 核数，并且提供了更详细的 CPU 信息：

```shell
$ lscpu
Architecture:             x86_64
  CPU op-mode(s):         32-bit, 64-bit
  Address sizes:          39 bits physical, 48 bits virtual
  Byte Order:             Little Endian
CPU(s):                   12
  On-line CPU(s) list:    0-11
....
```

我当前系统的 CPU 核数为 12，因此 `n` 可以设置为 24。使用下面的命令构建内核：

```shell
$ make -j24
```

构建过程的输出信息非常多，我们可以使用 `tee` 命令，将标准输出和标准错误信息显示在控制台，并将所有的输出信息保存到 `out.log` 文件中：

```shell
$ make -j24 2>&1 | tee out.log
```

如果想查看更详细的构建信息，例如 gcc 编译选项，可以使用 `V=1` 详细模式选项：

```shell
$ make -j24 V=1 2>&1 | tee out.log
```

构建内核是一个耗时的过程。如果你想了解整个构建过程具体花费了多少时间，可以使用 `time` 命令，它会在 `make` 命令执行完成后显示执行时间：

```shell
$ time make -j24 2>&1 | tee out.log
```

一切准备就绪，让我们开始构建吧！

如果没有意外，`make` 命令将成功执行完成，并生成以下关键文件：

* 未压缩的内核映像文件 **vmlinux**，位于内核源码根目录下
* 符号地址映射文件 **System.map**，位于内核源码根目录下
* 压缩的内核映像文件 **bzImage**，位于 `arch/<cpu>/boot/` 目录下。对于 x86 架构，它的实际位置是 `arch/x86/boot/bzImage`。
* **.ko** 文件，内核配置选项中被标记为 `M` 的内核模块，它们会散布在内核源码的各个子目录中。

获得这些构建成果后，下一步就是安装和使用它们了。

## 安装内核模块

在上一步构建完成后，我们生成了许多内核模块（`.ko` 文件）。可以使用 `find` 命令找到这些模块文件：

```shell
$ find . -name "*.ko"
./arch/x86/platform/atom/punit_atom_debug.ko
./arch/x86/crypto/aria-aesni-avx-x86_64.ko
./arch/x86/crypto/sm3-avx-x86_64.ko
./arch/x86/crypto/curve25519-x86_64.ko
...
```

内核模块以模块化的方式提供内核功能，这使得我们能够根据需要加载或从内核内存中移除特定的功能模块，从而提高内核的灵活性和可维护性。

构建完成的内核模块需要被安装到指定的位置，这样在系统启动时，才能被正确找到并加载到内核内存中。执行以下命令来安装内核模块：

```shell
$ sudo make modules_install
[sudo] password for mazhen: 
  SYMLINK /lib/modules/6.12.6-apusic-kernel/build
  INSTALL /lib/modules/6.12.6-apusic-kernel/modules.order
  INSTALL /lib/modules/6.12.6-apusic-kernel/modules.builtin
  INSTALL /lib/modules/6.12.6-apusic-kernel/modules.builtin.modinfo
...
```

从输出中可以看出，内核模块被安装到了 `/lib/modules/6.12.6-apusic-kernel` 目录下。注意，目录名 `6.12.6-apusic-kernel` 实际上就是我们构建的内核版本号。在系统中，每个已安装的内核都会在 `/lib/modules/` 目录下有一个对应的目录，并以内核版本号命名，用于存放对应版本的内核模块。这样，系统在启动时就可以加载正确版本的内核模块。

安装完内核模块后，下一步就要安装内核本身了。

## 安装内核

执行以下命令来安装新构建的内核：

```shell
sudo make install
```

`install` 目标实际上完成了以下三项关键任务：

1. 生成 `initramfs` (以前称为 `initrd`) 镜像。
2. 将内核镜像及相关文件安装到 `/boot` 目录。
3. 为新内核镜像配置 `GRUB` 引导程序。

下面将分别详细介绍这些步骤。

### initramfs 镜像

`install` 目标首先会生成 `initramfs` 镜像。那么，`initramfs` 是什么，它又有什么作用呢？

`initramfs` 的全称是 **initial RAM filesystem**，即初始 RAM 文件系统。它是一个使用 `cpio` 工具创建的归档文件包。`tar` 工具内部也使用了 `cpio`，所以可以认为 `initramfs` 就是一个经过压缩的归档文件包。

那么，`initramfs` 包含了什么内容呢？简单来说，`initramfs` 打包了一个精简版的 root 文件系统，其中包含了在系统初始化阶段需要用到的内核模块、设备驱动和用户态工具。

为什么需要 `initramfs` 呢？可以想象一下这个场景：当引导程序加载了内核镜像后，内核开始初始化工作，准备挂载真正的 root 文件系统。这时，内核需要加载文件系统对应的内核模块，但是这些内核模块此时还在磁盘上，在 root 文件系统完成挂载之前，内核还不能访问它们。这就产生了一个“先有鸡还是先有蛋”的问题：为了挂载 root 文件系统，需要从磁盘加载内核模块；而为了能加载内核模块，又需要先挂载 root 文件系统。

`initramfs` 就是为了解决这个问题而存在的。它包含了系统初始化阶段必须用到的内容。在内核挂载真正的 root 文件系统之前，`initramfs` 作为临时的 root 文件系统被挂载，为内核提供了一个基本的运行环境，使得内核能够执行一些必要的初始化操作。

### 安装内核镜像

`install` 目标在生成 `initramfs` 镜像后，会将内核镜像及相关文件复制到 `/boot` 目录，包括以下文件：

- **config-6.12.6-apusic-kernel**：内核配置文件。
- **System.map-6.12.6-apusic-kernel**：内核符号地址映射文件。
- **initrd.img-6.12.6-apusic-kernel**：上一步生成的 `initramfs` 镜像文件。`initrd` 是 `initramfs` 的旧称，现在仍被沿用。
- **vmlinuz-6.12.6-apusic-kernel**：`arch/x86/boot/bzImage` 文件的一个副本，也就是压缩过的内核镜像文件。 `vmlinuz` 中的 `z` 表示使用了 gzip 压缩，但实际上现在默认使用的是更优秀的 `ZSTD` 压缩算法，但文件命名依然保留。这也是为什么在最开始准备构建环境安装依赖时，需要安装 `zstd` 包。

另外，Ubuntu 提供了 `mkinitramfs` 和 `unmkinitramfs` 脚本，用于打包和解包 `initramfs` 镜像。我们可以使用 `unmkinitramfs` 脚本来查看 `initramfs` 镜像内部的结构：

```shell
$ TMPDIR=$(mktemp -d)
$ unmkinitramfs /boot/initrd.img-6.12.6-apusic-kernel
$ tree ${TMPDIR}
|-- early
|   `-- kernel
|       `-- x86
|           `-- microcode
|               `-- AuthenticAMD.bin
...
`-- main
    |-- bin -> usr/bin
    |-- conf
    |   |-- arch.conf
    |   |-- conf.d
    |   |-- initramfs.conf
    |   `-- modules
    |-- cryptroot
    |   `-- crypttab
    |-- etc
    |   |-- console-setup
...
    |-- init
    |-- lib -> usr/lib
    |-- lib.usr-is-merged -> usr/lib.usr-is-merged
...
    |-- usr
    |   |-- bin
    |   |   |-- [
    |   |   |-- [[
    |   |   |-- acpid
    |   |   |-- arch
    |   |   |-- ascii
    |   |   |-- ash
    |   |   |-- awk
    |   |   |-- base32
...
    |   |   |-- modules
    |   |   |   `-- 6.12.6-apusic-kernel
    |   |   |       |-- kernel
    |   |   |       |   |-- arch
    |   |   |       |   |   `-- x86
    |   |   |       |   |       `-- crypto
    |   |   |       |   |           |-- aegis128-aesni.ko
...
    |   |   |       |   |-- drivers
    |   |   |       |   |   |-- acpi
    |   |   |       |   |   |   |-- nfit
    |   |   |       |   |   |   |   `-- nfit.ko
    |   |   |       |   |   |   |-- platform_profile.ko
    |   |   |       |   |   |   `-- video.ko
    |   |   |       |   |   |-- ata
    |   |   |       |   |   |   |-- acard-ahci.ko
...
    |   |-- sbin
    |   |   |-- blkid
    |   |   |-- cache_check -> pdata_tools
    |   |   |-- cryptsetup
    |   |   |-- dhcpcd
...
447 directories, 1540 files
```

###  更新 `GRUB` 配置

`install` 目标的最后一步是为新构建的内核镜像配置 `GRUB` 引导程序。我们可以查看 `GRUB` 的配置文件 `/boot/grub/grub.cfg`，会发现它已经被更新，配置了我们最新构建的内核镜像和 `initramfs` 镜像。

```cfg
...
menuentry 'Ubuntu, with Linux 6.12.6-apusic-kernel' --class ubuntu --class gnu-linux --class gnu --class os $menuentry_id_option 'gnulinux-6.12.6-apusic-kernel-advanced-c3b02810-62dc-430b-b030-79c44c7a231f' {
                recordfail
                load_video 
                gfxmode $linux_gfx_mode
                insmod gzio
                if [ x$grub_platform = xxen ]; then insmod xzio; insmod lzopio; fi
                insmod part_gpt
                insmod ext2
                search --no-floppy --fs-uuid --set=root c3b02810-62dc-430b-b030-79c44c7a231f
                echo    'Loading Linux 6.12.6-apusic-kernel ...'
                linux   /boot/vmlinuz-6.12.6-apusic-kernel root=UUID=c3b02810-62dc-430b-b030-79c44c7a231f ro quiet splash quiet splash $vt_handoff
                echo    'Loading initial ramdisk ...'
                initrd  /boot/initrd.img-6.12.6-apusic-kernel
        }
...
```

现在，重启系统，默认就会使用我们新构建的内核了。

## 定制 GRUB

`GRUB` 默认会使用最新构建和安装的内核进行引导。然而，这种默认行为有时可能不符合我们的需求。例如，我们自己构建的内核可能包含实验性的代码和配置，其稳定性可能不如发行版提供的内核。因此，我们可能希望在系统引导阶段看到 `GRUB` 菜单，以便选择使用哪个内核启动，并将默认启动选项设置为一个稳定的发行版内核。

定制 `GRUB` 非常简单，我们只需要以 `root` 用户身份编辑 `GRUB` 的主配置文件`/etc/default/grub`。

```shell
GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-51-generic"
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=3
GRUB_DISTRIBUTOR=`( . /etc/os-release; echo ${NAME:-Ubuntu} ) 2>/dev/null || echo Ubuntu`
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
GRUB_CMDLINE_LINUX="quiet splash"
```

首先，我们需要设置默认启动选项 `GRUB_DEFAULT`。`GRUB_DEFAULT` 的默认值为 `0`，表示 `GRUB` 菜单中的第一个启动项。这样的设置会导致 `GRUB` 总是使用最新安装的内核作为默认选项。为了避免这种情况，我们将 `GRUB_DEFAULT` 的值设置为一个具体的菜单项，这样就可以固定默认启动选项，即使安装了新内核也不会改变。注意，子菜单之间需要使用 `>` 连接，并且文本内容要与 `GRUB` 菜单项严格一致。

`GRUB_TIMEOUT_STYLE=menu` 设置了在引导阶段显示 `GRUB` 菜单选项。菜单的超时时间由 `GRUB_TIMEOUT` 设置，本例中设置为 3 秒。如果在 3 秒内没有用户操作，系统将会使用默认选项引导。

在编辑并保存 `/etc/default/grub` 文件后，我们需要以 `root` 用户身份运行 `update-grub` 命令，以使修改生效：

```shell
sudo update-grub
```

此时，当我们重启系统时，就可以看到 `GRUB` 菜单。在 `Advanced options for Ubuntu` 子菜单下，可以选择使用我们新构建的内核 `Ubuntu, with Linux 6.12.6-apusic-kernel` 进行启动。

## 验证新内核的配置

现在，我们已经成功地使用自己构建的内核启动了系统！接下来，让我们检查一下之前在配置内核时设置的配置项是否已经生效。

首先，我们可以检查内核版本：

```shell
$ uname -r                          
6.12.6-apusic-kernel
```

没错，正是我们为 **CONFIG_LOCALVERSION** 配置项设置的值。

启用 `CONFIG_IKCONFIG` 配置项后，内核会包含自身的配置信息。我们可以使用脚本 `scripts/extract-ikconfig` 从内核镜像中提取出配置信息：

```shell
$ scripts/extract-ikconfig /boot/vmlinuz-6.12.6-apusic-kernel
#
# Automatically generated file; DO NOT EDIT.
# Linux/x86 6.12.6 Kernel Configuration
#
CONFIG_CC_VERSION_TEXT="gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0"
CONFIG_CC_IS_GCC=y
CONFIG_GCC_VERSION=130300
CONFIG_CLANG_VERSION=0
...
```

如果启用了 `CONFIG_IKCONFIG_PROC` 选项，内核配置信息会通过 `proc` 文件系统的文件 `/proc/config.gz` 以压缩格式暴露给用户。我们可以使用 `gunzip` 和 `grep` 命令来查找配置项 `CONFIG_HZ_250`：

```shell
$ gunzip -c /proc/config.gz | grep CONFIG_HZ_250
CONFIG_HZ_250=y
```

输出结果显示 `CONFIG_HZ_250=y`，说明我们配置的定时器中断频率也已生效。一切都是那么 perfect！

## 总结

从准备构建环境开始，我们了解了如何获取内核源码，掌握了多种配置内核的方法，然后成功地构建并安装了内核，定制了 `GRUB` 启动选项，最终验证了我们自己构建的内核。希望通过这一系列的实践，你能有所收获。
