# Latency Numbers Every Programmer Should Know

对于[冯·诺伊曼体系结构](https://en.wikipedia.org/wiki/Von_Neumann_architecture)的计算机，CPU 要数据才能正常工作。如果没有可处理的数据，那么CPU的运算速度再快也没有用，它只能等待。

在计算机和芯片发展的历史中，CPU 速度不断提高，但主内存的访问速度改进相对较慢，导致 CPU经常处于等待数据的状态，无法充分发挥其处理能力。为了解决这个问题，出现了 CPU 缓存。

寄存器和 CPU 缓存共同构成 CPU 内部的高速缓冲存储体系。

寄存器直接位于 CPU 内部，是距离 CPU 最近的存储单元。CPU 缓存分为多级，距离 CPU 最近的一级缓存(L1缓存)接在寄存器之后。

在内存层次结构中，从和 CPU 的接近程度来看是：

> 寄存器 > L1缓存 > L2缓存 > L3缓存 > 主内存

![Cache Hierarchy](https://upload.wikimedia.org/wikipedia/commons/0/00/Cache_Hierarchy_Updated.png)

寄存器访问速度最快，但容量最小。随着级别升高，访问速度下降但容量增大，以平衡访问速度和存储空间。

[Brown University](https://www.brown.edu/) 的 [Fundamentals of Computer Systems](https://cs.brown.edu/courses/csci0300/2023/index.html)课程有专门介绍计算机的[存储层次结构](https://cs.brown.edu/courses/csci0300/2023/notes/l12.html)：以**大小和速度**为标准，将不同类型的存储设备从**最快但容量最小**到**最慢但容量最大**进行排列。

![storage hierarchy](https://cs.brown.edu/courses/csci0300/2023/notes/assets/l10-storage-hierarchy.png)
存储层次结构这样设计是基于不同存储设备的**成本和性能特点**。**内存成本高但访问速度快**，而**硬盘成本低但访问速度慢**。所以采用这种层次结构可以在**平衡成本和性能**的前提下更好地利用各种存储设备。

在2010年，Google 的 [Jeff Dean](https://en.wikipedia.org/wiki/Jeff_Dean_(computer_scientist)) 在斯坦福大学发表了一次精彩的演讲 [Designs, Lessons and Advice from Building Large Distributed Systems](https://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf)，在演讲中总结了计算机工程师应该了解的一些重要数字：

![Numbers Everyone Should Know](https://raw.githubusercontent.com/mz1999/material/master/images/202310111506531.png)

后来有人做了一个非常好的[交互式web UI](https://colin-scott.github.io/personal_website/research/interactive_latency.html)，展示了这些数字随着时间的变化。
![#### Latency Numbers Every Programmer Should Know](https://raw.githubusercontent.com/mz1999/material/master/images/202310111518524.png)

这些数字对人的感觉不那么直观，它们之间的差异可以相差数个数量级，让我们很难真正理解这些差距有多大。于是 [Brendan Gregg](https://www.brendangregg.com/index.html)在他的书 [Systems Performance](https://www.brendangregg.com/systems-performance-2nd-edition-book.html)中，以 3.3 GHz 的 CPU 寄存器访问开始，放大成日常生活的时间单位，直观感受各系统组件访问时间的数量级差异：

![Example time scale of system latencies](https://raw.githubusercontent.com/mz1999/material/master/images/202310111532122.png)
如果一个 CPU cycle 是1秒，那么内存访问的延迟是6分钟，从旧金山到纽约（相当于深圳到乌鲁木齐）的网络延迟就是4年！
