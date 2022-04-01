# 深入理解 Page Cache      

长时间运行的Linux服务器，通常 free 的内存越来越少，让人觉得 Linux 特别能“吃”内存，甚至有人专门做了个网站 [LinuxAteMyRam.com](http://www.linuxatemyram.com/)解释这个现象。实际上 Linux 内核会尽可能的对访问过的文件进行缓存，来弥补磁盘和内存之间巨大的延迟差距。缓存文件内容的内存就是 **Page Cache**。

Google 的大神 [Jeffrey Dean](https://research.google/people/jeff/)总结过一个[Latency numbers every programmer should know](https://gist.github.com/hellerbarde/2843375)，其中提到从磁盘读取 1MB 数据的耗时是内存的80倍，即使换成 SSD 也是内存延迟的 4 倍。

我在本机做了实验，来体会一下 **Page Cache** 的作用。首先生成一个 1G 大小的文件：

```
# dd if=/dev/zero of=/root/dd.out bs=4096 count=262144
```

清空 Page Cache：

```
# sync && echo 3 > /proc/sys/vm/drop_caches
```

统计第一次读取文件的耗时：

```
# time cat /root/dd.out &> /dev/null

real	0m2.097s
user	0m0.010s
sys	    0m0.638s
```

再此读取同一个文件，由于系统已经将读取过的文件内容放入了 **Page Cache** ，这次耗时大大缩短：

```
# time cat /root/dd.out &> /dev/null

real	0m0.186s
user	0m0.004s
sys	    0m0.182s
```

**Page Cache** 不仅能加速对文件内容的访问，对共享库建立 **Page Cache**，可以在多个进程间共享，避免每个进程都单独加载，造成宝贵内存资源的浪费。

## **Page Cache** 是什么

**Page Cache** 是由内核管理的内存，位于 [VFS(Virtual File System)](https://www.kernel.org/doc/html/latest/filesystems/vfs.html) 层和具体文件系统层（例如ext4，ext3）之间。应用进程使用 `read`/`write` 等文件操作，通过系统调用进入到 **VFS** 层，根据 **O_DIRECT** 标志，可以使用 **Page Cache** 作为文件内容的缓存，也可以跳过 **Page Cache** 不使用内核提供的缓存功能。

另外，应用程序可以使用 [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) ，将文件内容映射到进程的虚拟地址空间，可以像读写内存一样直接读写硬盘上的文件。进程的虚拟内存直接和 **Page Cache** 映射。

![page cache](https://raw.githubusercontent.com/mz1999/material/master/images/202203251243899.png)

为了了解内核是怎么管理 Page Cache 的，我们先看一下 **VFS** 的几个核心对象：

* [file](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L956) 存放已打开的文件信息，是进程访问文件的接口；
* [dentry](https://elixir.bootlin.com/linux/v5.17/source/include/linux/dcache.h#L81) 使用 `dentry` 将文件组织成目录树结构；
* [inode](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L614) 唯一标识文件系统中的文件。对于同一个文件，内核中只会有一个 **inode** 结构。

对于每一个进程，打开的文件都有一个文件描述符，内核中进程数据结构 [task_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/sched.h#L728) 中有一个类型为 [files_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fdtable.h#L49:8) 的 [files](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/sched.h#L1073) 字段，保存着该进程打开的所有文件。[files_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fdtable.h#L49:8) 结构的[fd_array](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fdtable.h#L67) 字段是 [file](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L956) 数组， 数组的下标是文件描述符，内容指向一个 [file](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L956) 结构，表示该进程打开的文件。[file](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L956) 与打开文件的进程相关联，如果多个进程打开同一个文件，那么每个进程都有自己的  [file](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L956) ，但这些 [file](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L956)  指向同一个 [inode](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L614)。

![vfs](https://raw.githubusercontent.com/mz1999/material/master/images/202203251244454.png)

如上图所示，进程通过文件描述符与 `VFS` 中的 `file` 产生联系， 每个 `file` 对象又与一个 `dentry` 对应，根据 `dentry` 能找到 `inode`，而 `inode` 则代表文件本身。上图中进程 A 和进程 B 打开了同一个文件，进程 A 和进程 B 都维护着各自的 `file` ，但它们指向同一个 `inode`。

 [inode](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L614) 通过 [address_space](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L450) 管理着文件已加载到内存中的内容，也就是 **Page Cache**。[address_space](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L450) 的字段 `i_pages` 指向一棵 [xarray](https://www.kernel.org/doc/html/latest/core-api/xarray.html) 树，与这个文件相关的 **Page Cache** 页都挂在这颗树上。我们在访问文件内容的时候，根据指定文件和相应的页偏移量，就可以通过 [xarray](https://www.kernel.org/doc/html/latest/core-api/xarray.html) 树快速判断该页是否已经在 **Page Cache** 中。如果该页存在，说明文件内容已经被读取到了内存，也就是存在于 **Page Cache** 中；如果该页不存在，就说明内容不在 **Page Cache** 中，需要从磁盘中去读取。
 
![address space](https://raw.githubusercontent.com/mz1999/material/master/images/202203251244781.png)

 由于文件和  [inode](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L614) 一一对应，我们可以认为  [inode](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L614) 是 **Page Cache** 的宿主（`host`），内核通过 `inode->imapping->i_pages` 指向的树，管理维护着 **Page Cache**。

**Page Cache** 是如何产生和释放，又是如何与进程相关联的呢？我们需要先了解进程虚拟地址空间。

## 进程虚拟地址空间

Linux 是多任务系统，它支持多个进程的的并发执行。操作系统和 CPU 联手制造了一个假象：每个进程都独享连续的虚拟内存空间，并且各个进程的地址空间是完全隔离的，因此进程并不会意识到彼此的存在。从进程的角度来看，它会认为自己是系统中唯一的进程。

进程看到的是虚拟内存的地址空间，它也不能直接访问物理地址。当进程访问某个虚拟地址的时候，该虚拟地址由内核负责转换成物理内存地址，即完成虚拟地址到物理地址的映射。这样不同进程在运行的时候，即使访问相同的虚拟地址，但内核会将它们映射到不同的物理地址，因此不会发生冲突。

进程在 Linux 内核由 [task_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/sched.h#L728) 所描述。估计 [task_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/sched.h#L728) 是你学习内核时第一个熟悉的数据结构，因为它实在太重要了。 [task_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/sched.h#L728)描述了进程相关的所有信息，包括进程状态，运行时统计信息，进程亲缘关系，进度调度信息，信号处理，进程内存管理，进程打开的文件等等。我们这里关注的进程虚拟内存空间，是由 [task_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/sched.h#L728) 中的 [mm](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/sched.h#L860) 字段指向的 [mm_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L458) 所描述，它是一个进程内存的运行时摘要信息。

![process_address_space](https://raw.githubusercontent.com/mz1999/material/master/images/202203262218327.png)

进程的虚拟地址是线性的，使用结构体 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 来描述。内核将每一段具有相同属性的内存区域当作一个 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 进行管理，每个 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375)  是一个连续的虚拟地址范围，这些区域不会互相重叠。 [mm_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L458) 里面有一个单链表 `mmap`，用于将 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 串联起来，另外还有一颗红黑树 `mm_rb` ，[vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 根据起始地址挂在这颗树上。使用红黑树可以根据地址，快速查找一个内存区域。

[vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 可以直接映射到物理内存，也可以关联文件。如果 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 是文件映射，由成员 `vm_file` 指向对应的文件指针。一个没有关联文件的 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 是匿名内存。

开发者使用 [malloc](https://man7.org/linux/man-pages/man3/free.3.html) 等 glibc 库函数分配内存的时候，不是直接分配物理内存，而是在进程的虚拟内存空间中申请一段虚拟内存，生成相应的数据结构 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) ，然后将它插进 [mm_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L458) 的链表 `mmap`，同时挂在红黑树 `mm_rb` 上，就算完成了工作，根本没有涉及到物理内存的分配。只有当第一次对这块虚拟内存进行读写时，发现该内存区域没有映射到物理内存，这时会触发缺页中断，然后由内核填写页表，完成虚拟内存到物理内存的映射。

当开发者使用 [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) 进行文件映射时，内核根据 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375)  中代表文件映射关系 `vm_file`，将文件内容从磁盘加载到物理内存，也就是 **Page Cache** 中，最后建立这段虚拟地址到物理地址的映射。

另外，在虚拟内存中连续的页面，在物理内存中不必是连续的。只要维护好从虚拟内存页到物理内存页的映射关系，你就能正确地使用内存。由于每个进程都有独立的地址空间，为了完成虚拟地址到物理地址的映射，每个进程都要有独立的进程页表。在一个实际的进程里面，虚拟内存占用的地址空间，通常是两段连续的空间，而不是完全散落的随机的内存地址。基于这个特点，内核使用多级页表保存映射关系，可以大大减少页表本身的空间占用。最顶级的页表保存在  [mm_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L458) 的 `pgd` 字段中。

好了，我们对进程虚拟地址空间有了基本的了解，下面看看 **Page Cache** 的产生和释放，以及如何与进程空间发生联系的。

## Page Cache 的产生和释放

**Page Cache** 的产生有两种不同的方式：

* Buffered I/O
* Memory-Mapped file

![Memory Mapped File](https://raw.githubusercontent.com/mz1999/material/master/images/202203272309698.png)

使用这两种方式访问磁盘上的文件时，内核会根据指定的文件和相应的页偏移量，判断文件内容是否已经在 **Page Cache** 中，如果内容不存在，需要从磁盘中去读取并创建 **Page Cache** 页。

这两种方式的不同之处在于，使用 `Buffered I/O`，要先将数据从 **Page Cache** 拷贝到用户缓冲区，应用才能从用户缓冲区读取数据。而对于 `Memory-Mapped file` 而言，则是直接将 **Page Cache** 页映射到进程虚拟地址空间，用户可以直接读写 **Page Cache** 中的内容。由于少了一次 copy，使用 `Memory-Mapped file` 要比 `Buffered I/O`  的效率高一些。

随着服务器运行时间的增加，系统中空闲内存会越来越少，其中很大一部分都被 Page Cache 占用。访问过的文件都被 Page Cache 缓存，内存最终会被耗尽，那什么时候回收 Page Cache 呢？ 内核认为，Page Cache 是可回收内存，当应用在申请内存时，如果没有足够的空闲内存，就会先回收 Page Cache，再尝试申请。回收的方式主要是两种：直接回收和后台回收。

![reclaim_page_cache](https://raw.githubusercontent.com/mz1999/material/master/images/202203291742454.png)

使用 `Buffered I/O` 时，**Page Cache** 并没有和进程的虚拟内存空间产生直接的关联，而是通过用户缓冲区作为中转。效率更好的`Memory-Mapped file`方式，看着比较简单，但背后的实现却有些复杂。下面我们看一下内核是如何实现 `Memory-Mapped file` 的。

## 内存文件映射

前面我们介绍过，  [inode](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L614) 是 **Page Cache** 的宿主（`host`），内核通过 `inode->imapping->i_pages` 指向的树，管理维护着 **Page Cache**。那么内核是如何完成内存文件映射，直接把缓存了文件内容的 **Page Cache** 映射到进程虚拟内存空间的呢？

![address_space_memory_mapping](https://raw.githubusercontent.com/mz1999/material/master/images/202203291049324.png)

我们知道，进程结构体 [task_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 中的字段 `mm` 指向该进程的虚拟地址空间 [mm_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/mm_types.h#L458) ，而一段虚拟内存由结构体 [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 所描述，将 [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 串在一起的链表 `mmap` 就代表了已经申请分配的虚拟内存。

如果是进行内存文件映射，那么映射了文件的虚拟内存区域  [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) ，它的 `vm_file` 会指向被映射的文件结构体  [file](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L956)。[file](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L956) 表示进程打开的文件，它的成员 `f_mapping` 指向 [address_space](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L450)，这样就和管理文件着 **Page Cache** 的 [address_space](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L450) 关联了起来。

当第一次访问文件映射的虚拟内存区域时，这段虚拟内存并没有映射到物理内存，这时会触发缺页中断。内核在处理缺页中断时，发现代表这段虚拟内存的 [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 有关联的文件，即 `vm_file` 字段指向一个文件结构体 [file](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L956)。内核拿到该文件的 [address_space](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/fs.h#L450)，根据要访问内容的页偏移量，对 `address_space->i_pages` 指向的 [xarray](https://www.kernel.org/doc/html/latest/core-api/xarray.html) 树进行查找。这颗树上挂的都是页偏移量对应的内存页，如果没找到，就说明文件内容还没加载进内存，就会分配内存页，将文件内容加载到内存中，然后把内存页挂在  [xarray](https://www.kernel.org/doc/html/latest/core-api/xarray.html) 树上。下次再访问同样的页偏移量时，文件内容已经在树上，可直接返回。 `address_space->i_pages` 指向的树就是内核管理的 **Page Cache**。

将文件内容加载到 **Page Cache** 后，内核就可以填写进程相关的页表项，将这块文件映射的虚拟地址区域，直接映射到 **Page Cache** 页，完成缺页中断的处理。

当内存紧张需要回收 **Page Cache** 时，内核需要知道这些  Page Cache 页映射到了哪些进程，这样才能修改进程的页表，解除虚拟内存和物理内存的映射。我们知道，同一个文件可以映射到多个进程空间，所以需要保存**反向映射关系**，即根据 Page Cache 页找到进程。

Page Cache  页的反向映射关系保存在 [address_space](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L450) 维护的另一颗树 `i_mmap`。`address_space->i_mmap` 是一个优先查找树（Priority Search Tree），关联了这个文件 Page Cache 页的 [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 就挂在这棵树上，而这些  [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728)都将指向各自的进程空间描述符 [mm_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/mm_types.h#L458)，从而建立了 Page Cache 页到进程的联系。

当需要解除一个 Page Cache 页的映射时，利用 `address_space->i_mmap` 指向的树，查找 Page Cache 页映射到哪些进程的哪些 [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728)，从而确定需要修改的进程页表项内容。

简单总结一下，一个文件对应的  [address_space](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L450) 主要管理着两颗树：`i_pages` 指向的 [xarray](https://www.kernel.org/doc/html/latest/core-api/xarray.html) 树，维护着的所有 Page Cache 页；`i_mmap` 指向的 PST 树，维护着文件映射所形成的  [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 虚拟内存区域，用来在释放 Page Cache 页时，查找映射了该文件的进程。如果文件没有被映射到进程空间，那么 `i_mmap` 对应的 PST 树为空。

## Page Cache 的观测

可以通过查看 `/proc/meminfo` 文件获知 **Page Cache** 相关的各种指标。

`/proc` 是伪文件系统（Pseudo filesystems ）。Linux 通过伪文件系统，让系统和内核信息在用户空间可用。使用 `free`、`vmstat`等命令查看到的内存信息，数据实际上都来自 `/proc/meminfo`。

我们看一个示例： 

<pre>
$ cat /proc/meminfo 
MemTotal:        8052564 kB
MemFree:          129804 kB
MemAvailable:    4956164 kB
<b>Buffers:          175932 kB</b>
<b>Cached:          4896824 kB</b>
SwapCached:           40 kB
Active:          2748728 kB  <- Active(anon) + Active(file)
Inactive:        4676540 kB  <- Inactive(anon) +Inactive(file)  
<b>Active(anon):       3432 kB</b>
<b>Inactive(anon):  2513172 kB</b>
<b>Active(file):    2745296 kB</b>
<b>Inactive(file):  2163368 kB</b>
Unevictable:       65496 kB
Mlocked:               0 kB
SwapTotal:       2097148 kB
SwapFree:        2095868 kB
Dirty:                12 kB
Writeback:             0 kB
<b>AnonPages:       2411440 kB</b>
Mapped:           761076 kB
<b>Shmem:            170868 kB</b>
...
</pre>

关于 `/proc/meminfo` 每一项的详细解释，可以查看 [Linux 内核文档 - The /proc Filesystem]([The /proc Filesystem — The Linux Kernel documentation](https://www.kernel.org/doc/html/latest/filesystems/proc.html#meminfo))。我们重点看一下 **Page Cache** 相关的字段。

当前系统 Page Cache 等于 Buffers + Cached 之和 ：

```
Buffers + Cached = 5072756 kB
```

前面讨论过，如果 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 关联到文件，那么这段内存区域就是 File-backed 内存。没有关联文件的 [vm_area_struct](https://github.com/torvalds/linux/blob/f443e374ae131c168a065ea1748feac6b2e76613/include/linux/mm_types.h#L375) 内存区域是匿名内存。我们是否可以认为，和磁盘文件相关联的 File-backed 内存总和，应该等于 Page Cache 呢？

```
Active(file) + Inactive(file) = 4908664 kB
```

好像有点对不上，还差了一些，差的这部分是共享内存（**Shmem**）。

Linux 为了实现“共享内存”（shared memory）功能，即多个进程共同使用同一内存中的内容，需要使用虚拟文件系统。虚拟文件并不是真实存在于磁盘上的文件，它只是由内核模拟出来的。但虚拟文件也有自己的 [inode](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L614)  和  [address_space](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L450)结构。内核在创建共享匿名映射区域时，会创建出一个虚拟文件，并将这个文件与 [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728)关联起来，这样多个进程的  [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 会关联到同一个虚拟文件，最终映射到同样的物理内存页，从而实现了共享内存功能。这就是共享内存（**Shmem**）的实现原理。

 由于 **Shmem** 没有关联磁盘上的文件，因此它不属于 File-backed 内存，而是被记录在匿名内存（**Active(anon)** 或 **Inactive(anon)**）部分。但因为 **Shmem** 有自己的 [inode](https://elixir.bootlin.com/linux/v5.17/source/include/linux/fs.h#L614) ，`inode->address_sapce` 维护的 Page Cache 页挂在 `address_space->i_pages` 指向的 [xarray](https://www.kernel.org/doc/html/latest/core-api/xarray.html) 树上，因此 **Shmem** 部分的内存也应该算在 Page Cache 里。

此外 File-backed 内存还有 Active 和 Inactive 的区别。刚被使用过的数据的内存空间被认为是 Active 的，长时间未被使用过的数据的内存空间则被认为是 Inactive 的。当物理内存不足，不得不释放正在使用的内存时，会首先释放 Inactive 的内存。 

**Page Cache** 和 匿名内存以及 File-backed 内存等之间的关系，如图下图所示。虽然难免存在误差，但大体来说下面的关系式是成立的：

![file-backed anon](https://raw.githubusercontent.com/mz1999/material/master/images/202203301748033.png)

值得注意的是，**AnonPages != Active(anon) + Inactive(anon)**。**Active(anon)** 和 **Inactive(anon)** 是用来表示不可回收但是可以被交换到 swap 分区的内存，而 **AnonPages** 则是指没有对应文件的内存，两者的角度不一样。 **Shmem** 虽然属于**Active(anon)** 或者 **Inactive(anon)**，但是 **Shmem** 有对应的内存虚拟文件，所以它不属于 **AnonPages**。

总之，**Page Cache** 肯定关联了文件，不管是真实存在的磁盘文件，还是虚拟内存文件。**AnonPages** 则没有关联任何文件。**Shmem** 关联了虚拟文件，它属于 **Active(anon)** 或者 **Inactive(anon)**，同时也算在 **Page Cache** 中。

如果我们想知道某个文件有多少内容被缓存在 Page Cache ，可以使用 [fincore]([fincore（1） - Linux 手册页 (man7.org)](https://man7.org/linux/man-pages/man1/fincore.1.html)) 命令。例如：

```
$ fincore /usr/lib/x86_64-linux-gnu/libc.so.6
RES  PAGES  SIZE FILE
2.1M   542  2.1M /usr/lib/x86_64-linux-gnu/libc.so.6
```

`RES` 是文件内容被加载进物理内存占用的内存空间大小。`PAGES` 是换算成文件内容占用了多少内存页。 在上面的例子中，文件 `/usr/lib/x86_64-linux-gnu/libc.so.6` 的全部内容，都被加载进了 Page Cache。

结合 `lsof` 命令，我们可以查看某一进程打开的文件占用了多少 Page Cache：

```
$ sudo lsof -p 1270 | grep REG | awk '{print $9}' | xargs sudo fincore
  RES PAGES   SIZE FILE
64.8M 16580  89.9M /usr/bin/dockerd
  32K     8    32K /var/lib/docker/buildkit/cache.db
  16K     4    16K /var/lib/docker/buildkit/metadata_v2.db
  16K     4    16K /var/lib/docker/buildkit/snapshots.db
  16K     4    16K /var/lib/docker/buildkit/containerdmeta.db
 284K    71 282.4K /usr/lib/x86_64-linux-gnu/libnss_systemd.so.2
 244K    61 594.7K /usr/lib/x86_64-linux-gnu/libpcre2-8.so.0.10.2
 156K    39 154.2K /usr/lib/x86_64-linux-gnu/libgpg-error.so.0.29.0
  24K     6  20.6K /usr/lib/x86_64-linux-gnu/libpthread.so.0
 908K   227 906.5K /usr/lib/x86_64-linux-gnu/libm.so.6
 ...
```

另外，对于所有缓存类型，缓存命中率都是一个非常重要的指标。我们可以使用 [bcc](https://github.com/iovisor/bcc) 内置的工具 [cachestat](https://github.com/iovisor/bcc/blob/master/tools/cachestat_example.txt)  追踪整个系统的  Page Cache 命中率：

```
$ sudo cachestat-bpfcc
    HITS   MISSES  DIRTIES HITRATIO   BUFFERS_MB  CACHED_MB
    2059        0       32  100.00%           74       1492
     522        0        0  100.00%           74       1492
      32        0        7  100.00%           74       1492
     135        0       69  100.00%           74       1492
      97        1        3   98.98%           74       1492
     512        0       82  100.00%           74       1492
     303        0       86  100.00%           74       1492
    2474        7     1028   99.72%           74       1494
     815        0      964  100.00%           74       1497
    2786        0        1  100.00%           74       1497
    1051        0        0  100.00%           74       1497
^C     502        0        0  100.00%           74       1497
Detaching...
```

使用 [cachetop](https://github.com/iovisor/bcc/blob/master/tools/cachetop_example.txt) 可以按进程追踪 Page Cache 命中率：

```
$ sudo cachetop-bpfcc 
14:20:41 Buffers MB: 86 / Cached MB: 2834 / Sort: HITS / Order: descending
PID      UID      CMD              HITS     MISSES   DIRTIES  READ_HIT%  WRITE_HIT%
   14237 mazhen   java                12823     4594     3653      52.6%      13.2%
   14370 mazhen   ldd                   869        0        0     100.0%       0.0%
   14371 mazhen   grep                  596        0        0     100.0%       0.0%
   14376 mazhen   ldd                   536        0        0     100.0%       0.0%
   14369 mazhen   env                   468        0        0     100.0%       0.0%
   14377 mazhen   ldd                   467        0        0     100.0%       0.0%
   14551 mazhen   grpc-default-ex       466        0        0     100.0%       0.0%
   14375 mazhen   ldd                   435        0        0     100.0%       0.0%
   14479 mazhen   ldconfig              421        0        0     100.0%       0.0%
   14475 mazhen   BookieJournal-3       417       58      132      60.0%       6.1%
   ...
```

## mmap系统调用

系统调用 [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html)是最重要的内存管理接口。使用 [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) 可以创建文件映射，从而产生 Page Cache。使用  [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) 还可以用来申请堆内存。glibc 提供的  [malloc](https://man7.org/linux/man-pages/man3/free.3.html)，内部使用的就是 [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) 系统调用。 由于 [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) 系统调用分配内存的效率比较低，  [malloc](https://man7.org/linux/man-pages/man3/free.3.html) 会先使用 [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) 向操作系统申请一块比较大的内存，然后再通过各种优化手段让内存分配的效率最大化。

[mmap](https://man7.org/linux/man-pages/man2/mmap.2.html) 根据参数的不同， 可以从是不是文件映射，以及是不是私有内存这两个不同的维度来进行组合：

![mmap](https://raw.githubusercontent.com/mz1999/material/master/images/202203311056958.png)

* **私有匿名映射**

在调用 `mmap(MAP_ANON | MAP_PRIVATE)` 时，只需要在进程虚拟内存空间分配一块内存，然后创建这块内存所对应的  [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 结构，这次调用就结束了。当访问到这块虚拟内存时，由于这块虚拟内存都没有映射到物理内存上，就会发生缺页中断。 [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728)关联文件属性为空，所以是匿名映射。内核会分配一个物理内存，然后在页表里建立起虚拟地址到物理地址的映射关系。

* **私有文件映射**

进程通过 `mmap(MAP_FILE | MAP_PRIVATE)` 这种方式来申请的内存，比如进程将共享库（Shared libraries）和可执行文件的代码段（Text Segment）映射到自己的地址空间就是通过这种方式。

如果文件是只读的话，那这个文件在物理页的层面上其实是共享的。也就是进程 A 和进程 B 都有一页虚拟内存被映射到了相同的物理页上。但如果要写文件的时候，因为这一段内存区域的属性是私有的，所以内核就会做一次写时复制，为写文件的进程单独地创建一份副本。这样，一个进程在写文件时，并不会影响到其他进程的读。

私有文件映射的只读页是多进程间共享的，可写页是每个进程都有一个独立的副本，创建副本的时机仍然是写时复制。

* **共享文件映射**

进程通过 `mmap(MAP_FILE | MAP_SHARED)` 这种方式来申请的内存。在私有文件映射的基础上，共享文件映射就很简单了：对于可写的页面，在写的时候不进行复制就可以了。这样的话，无论何时，也无论是读还是写，多个进程在访问同一个文件的同一个页时，访问的都是相同的物理页面。

* **共享匿名映射**

进程通过 `mmap(MAP_ANON | MAP_SHARED)` 这种方式来申请的内存。借助虚拟文件系统，多个进程的  [vm_area_struct](https://elixir.bootlin.com/linux/v5.17/source/include/linux/sched.h#L728) 会关联到同一个虚拟文件，最终映射到同样的物理内存页，实现进程间共享内存的功能。

`mmap` 的四种映射类型，和上面介绍的 `/proc/meminfo` 内存指标之间的关系：

![mmap](https://raw.githubusercontent.com/mz1999/material/master/images/202203311637230.png)

私有映射都属于 **AnonPages**，共享映射都是 **Page cache**。前面讨论过，共享的匿名映射 **Shmem**，虽然没有关联真实的磁盘文件，但是关联了虚拟内存文件，所以也属于 **Page Cache**。

私有文件映射，如果文件是只读的话，这块内存属于 **Page Cache**。如果有进程写文件，因为这一段内存区域的属性是私有的，所以内核就会做一次写时复制，为写文件的进程单独地创建一份副本，这个副本就属于 **AnonPages** 了。

## 写在最后

Page Cache 机制涉及了进程空间，文件系统，内存管理等多个内核功能，Page Cache 就像一条线将这几部分串在了一起。因此深入理解 Page Cache 机制，对学习内核会有很大的帮助。