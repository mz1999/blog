
# Java EE应用服务器的事务管理

在计算机科学中，**事务处理**（**transaction processing** ）是将信息处理划分为独立的、不可分割的操作，称为**事务**（**Transaction**）。每个事务必须作为一个完整的执行单元，要么整个事务成功（提交），要么失败（中止，回滚），它永远不能只是部分完成。使用事务可以简化应用程序的错误处理，因为它不需要担心部分失败，系统（通常是数据库或某些现代文件系统）的完整性始终处于已知的、一致的状态。

**事务处理**是一项关键技术，可以应用于多个问题领域——企业架构、电子商务解决方案、金融系统和许多其他领域。事务的一个很好的例子就是，资金从一家银行的账户转移到另一家银行的账户。资金转移涉及在一个账户上扣款，并在另一个账户上增加相同的金额。使用事务可以确保不会出现由于其中一项操作失败，而导致资金丢失或产生的不一致状态。

## 事务处理简史

现代事务处理技术是在20世纪60年代开始的大型机计算背景下发展起来的，在许多方面，我们今天使用的技术是对这些模型的改进和调整。第一个**事务处理系统**（**Transaction processing system**）是著名的 SABRE 航空预订系统，由IBM和美国航空公司在20世纪50年代末和60年代初开发。以今天的标准来看，SABRE 相当粗糙： ACID 事务语义完全是由应用程序实现的。IBM 很快就意识到这种技术可以应用于其他行业，由此产生了**CICS**（Customer Information Control System）产品，它最初是完全用汇编语言编写，并没有使用现代意义上的数据库，而是依赖于扁平文件和其他低级数据结构，但 CICS 已经实现了基本的事务处理功能。

1970年 [Edgar F. Codd](https://en.wikipedia.org/wiki/Edgar_F._Codd) 发表了一篇名为[《A Relational Model of Data for Large Shared Data Banks》](https://www.seas.upenn.edu/~zives/03f/cis550/codd.pdf)的论文，首次提出了关系模型的概念，这一模型为后来的关系数据库管理系统（RDBMS）奠定了基础。随后关系数据库管理系统开始兴起，IBM 的 [System R](https://en.wikipedia.org/wiki/IBM_System_R) 项目是一个关键的里程碑，它是 SQL 的第一个实现，从此成为标准的关系数据查询语言。在 System R 项目中，事务处理技术被引入到关系数据库领域，为后来的数据库系统的发展奠定了基础。数据库管理系统成为了事务处理的核心组件，负责数据存储和管理、事务处理、并发控制和恢复等关键功能。

随着计算机技术的发展和网络通信技术的普及，分布式计算逐渐成为企业应用的重要趋势。在分布式环境中实现事务处理面临着许多挑战，传统的单机事务处理系统无法满足需求，[两阶段提交协议（2PC）](https://en.wikipedia.org/wiki/Two-phase_commit_protocol)等技术应运而生。为了管理这些分布式事务，提供更好的并发控制和容错能力，**事务处理监视器**（**Transaction Processing Monitor** 或 **TP Monitor**）被引入。

**TP Monitor** 负责在分布式环境中管理和监控事务处理过程。它处理客户端请求、协调事务、确保数据一致性、管理资源访问以及处理故障恢复等。**TP Monitor** 是一个软件框架或应用程序执行环境，为应用程序提供了一个完整的运行时，允许应用以安全和事务性的方式访问后端系统（包括数据库）。**TP Monitor** 作为事务处理中间件，目标是让程序员更容易的编写和部署可靠、可扩展的事务应用程序，使程序员能够专注于业务逻辑，而不是底层的事务管理。

在20世纪90年代初，[X/Open](https://www.opengroup.org/) 发布了[X/Open DTP模型](https://pubs.opengroup.org/onlinepubs/009249599/toc.pdf)，为分布式事务处理提供了一个统一的框架和一组标准接口。遵循 X/Open DTP 模型的 TP Monitor 实现了该模型所定义的组件和接口，包括事务管理器（TM）、资源管理器（RM）和通信资源管理器（CRM）。一些广泛使用的遵循 X/Open DTP 模型的 TP Monitor 产品包括BEA 的 Tuxedo 和 Transarc 的 Encina 等。

[CORBA Object Transaction Service (OTS)](https://www.omg.org/spec/TRANS) 是一个定义在 [CORBA（Common Object Request Broker Architecture）](https://www.omg.org/spec/CORBA) 规范中的分布式事务服务。**OTS** 将分布式事务处理模型（DTP）扩展到了对象领域，它提供了一种在分布式对象系统中进行事务处理的方法。**OTS** 定义了一组标准的接口和协议，允许 CORBA 对象参与分布式事务。

Java EE 应用服务器是在 X/Open DTP 模型和 CORBA OTS 的基础上发展出来的事务处理监视器，**TP Monitor** 开始融入 Java EE应用服务器，提供更丰富的中间件服务和组件化的应用程序模型。**TP Monitor** 本质上是一个具有事务感知功能的**应用服务器**，事实上，**Java EE 应用服务器**中的许多功能都源于TP Monitor。同样地，许多现代的 TP Monitor 是带有事务服务核心的 Java EE 应用服务器。

## 事务概念基础

本章我们简要地回顾一些事务处理的基本概念，它们塑造了中间件对事务的支持。

### ACID 属性

事务提供的安全保障通常用缩写**ACID**来描述，它代表原子性（**atomicity**）、一致性（**consistency**）、隔离性（**Isolation**）和持久性（**Durability**）。事务是一个**原子**（**Atomicity**）工作单元，它将系统从一个**一致**（**Consistency**）的状态转换为另一个一致状态，执行时不受其他同时执行的事务的干扰（**isolation**），并且一旦提交，就不能因系统故障而撤消(**Durability**)。ACID 是1983年由 Theo Härder 和Andreas Reuter 在论文[《 Principles of Transaction-Oriented Database Recovery》](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.2812&rep=rep1&type=pdf)中首次提出。

下面我们更详细地研究一下ACID属性，这将让我们更深入的理解事务。

#### Atomicity

Atomicity 这个术语在不同的领域有着类似但又不相同的含义。例如在多线程编程领域，如果一个线程执行了一个**原子**操作，这意味着另一个线程不可能看到这个操作的半成品。系统只能处于操作前或操作后的状态，而不能介于两者之间。

在 ACID 的背景下，Atomicity 不是关于并发性的，它并没有描述当多个进程试图同时访问相同的数据时会发生什么，因为关于并发访问的场景是在隔离性（Isolation）中描述的。

ACID 原子性描述的是，如果一个客户想要进行多次写入，但在处理部分写操作后出现故障的情况。这些故障可能是进程崩溃、网络连接中断、磁盘已满等。如果将这些写操作组合到一个事务中，由于故障无法完成事务提交，那么该事务将被中止，并且数据库必须撤消之前的任何写操作。

在没有 Atomicity 保证的情况下，如果在进行多次修改的过程中发生错误，就很难知道哪些修改已经生效，哪些没有生效。Atomicity 简化了这个问题：如果事务被中止，应用程序可以确定它没有改变任何东西。

所以 Atomicity 的本质是，**在出错时中止事务，并丢弃该事务对数据的所有修改**。

#### Consistency

事务应该确保系统从一个一致性状态转换到另一个一致性状态。在事务开始和结束时，系统的完整性约束必须得到满足。

我们所说的一致性，有些是通过数据库的完整性约束来保证的，例如使用主键作为员工编号，那么由数据库来保证所有主键都是唯一的。

但在很多情况下，一致性都是有具体业务含义的，应用程序定义了什么状态是有效或无效的，例如每个部门的支出必须小于或等于该部门的预算，这种一致性只能由应用程序来保证。

原子性、隔离性和持久性是数据库的属性，而一致性是应用程序的属性。维护事务的一致性是应用程序和数据库的共同责任，应用程序是依靠数据库的原子性和隔离性来实现一致性。

#### Isolation

如果多个用户同时读写数据库相同的记录，就会遇到并发问题。Isolation 意味着同时执行的事务是相互隔离的，事务的执行不会受到其他并发事务的影响，每个事务都可以假装它是整个数据库中唯一运行的事务。

#### Durability

持久性是指一旦事务成功提交，它所写入的任何数据都不会丢失，即使出现硬件故障或数据库崩溃。改变系统持久状态的唯一方法是提交一个事务。

对于单节点数据库，持久性通常意味着数据已写入硬盘或 SSD 等非易失性存储中。数据库一般都会使用**WAL**（write-ahead log）技术，在向持久化存储写入未提交的变更之前，先向日志中写入相应的事务日志记录，并确保事务日志记录在事务提交之前被持久化。当遇到故障重启系统时，数据库可以通过重新执行所有已提交事务的日志记录，撤消所有中止事务的日志记录，让数据库恢复到一致性状态。

### 隔离级别

隔离性（Isolation）是事务 ACID 四个属性之一，它确保多个并发事务在操作数据库时，彼此之间不会互相干扰，从而保证数据的一致性。

当一个事务读取被另一个事务同时修改的数据，或者两个事务试图同时修改相同的数据时，会出现并发性问题。数据库通过提供**事务隔离**向应用开发人员隐藏了并发性问题的复杂性。

隔离级别（Isolation Level）定义了不同的隔离性强度，以在性能和数据一致性之间取得平衡。可串行化（serializable）的隔离意味着数据库保证事务具有与串行运行相同的效果，即事务一个接着一个的运行，没有任何并发。然而在实践中，可串行化（serializable）的隔离有一定的性能成本，因此使用较弱的隔离级别是很常见的。根据SQL标准，隔离级别分为四个等级，从低到高分别为：

* **读未提交**（Read Uncommitted）：这是最低的隔离级别。在这个级别下，一个事务可以看到其他事务尚未提交的数据。这意味着可能发生脏读（Dirty Read），即一个事务读取到了另一个尚未提交的事务所修改的数据。这个级别的优点是并发性能较高，但数据一致性较差。

* **读已提交**（Read Committed）：这个级别要求一个事务只能看到其他事务已经提交的数据。这意味着脏读不会发生，但仍然可能发生不可重复读（Non-repeatable Read），即在同一个事务中，多次读取同一数据可能得到不同的结果。这个级别在性能和数据一致性之间取得了一定的平衡。

* **可重复读**（Repeatable Read）：这个级别要求在同一个事务中，对同一数据的多次读取结果必须一致。这可以避免不可重复读的问题，但仍然可能发生幻读（Phantom Read），即在一个事务执行过程中，其他事务插入了满足查询条件的新数据。这个级别提供了较好的数据一致性保障，但并发性能受到一定影响。

* **可串行化**（Serializable）：这是最高的隔离级别。在这个级别下，事务被处理得就像是串行执行一样，完全避免了脏读、不可重复读和幻读问题。然而，这种级别的数据一致性保障是以牺牲并发性能为代价的，可能导致事务处理效率降低。

较低的隔离级别增加了用户并发访问相同数据的能力，但也增加了用户可能遇到的并发问题（例如脏读或丢失更新）的数量。相反，较高的隔离级别减少了用户遇到的并发问题，但需要更多的系统资源，并增加了一个事务阻塞另一个事务的机会。

在实践中，可串行化（serializable）隔离很少被使用，Oracle 数据库甚至没有实现它。在Oracle中，有一个叫做 "serializable "的隔离级别，但它实际上实现的是**快照隔离**（*snapshot isolation*）。

* **快照隔离**通过为每个事务提供一个数据快照来实现，在一个事务执行过程中，它只能看到和操作事务开始时的数据快照，而不会受到其他并发事务的影响。快照隔离的实现通常依赖于多版本并发控制（[MVCC，Multi-Version Concurrency Control](https://en.wikipedia.org/w/index.php?title=Multiversion_concurrency_control&useskin=vector)）技术。快照隔离能够解决脏读、不可重复读和幻读问题，但它会导致写偏斜（Write Skew），即两个或多个事务同时读取相同的数据，然后基于读取到的数据做出独立的修改，最终导致数据不一致的状态。因此，快照隔离比可串行化的保证要弱。

### 分布式事务

随着计算机网络的发展，分布式计算变得越来越普遍。这导致了分布式事务处理的需求，即在多个独立的数据库或资源管理器上执行的事务。分布式事务处理具有更高的复杂性，需要协调和管理跨越不同系统的事务。这样的事务处理通常需要遵循分布式事务处理的规范和算法，如两阶段提交协议。

### 两阶段提交

为了使分布式事务的操作表现得像一个原子单元，参与的分布式资源必须根据事务的结果全部提交或全部放弃。两阶段提交（2PC，Two-Phase Commit）协议是一种用于分布式事务处理的原子性协议，它通过在所有事务参与者之间进行协调和同步，以确保分布式事务的原子性得以维护。

两阶段提交协议引入了一个新的组件：协调器 *coordinator*，也被称为*transaction manager*。coordinator 通常和应用进程在同一个进程中，例如 Java EE应用服务器中 *transaction manager*，但它也可以是一个单独的程序或服务。

两阶段提交协议包括两个阶段：提交请求阶段（Prepare Phase）和提交阶段（Commit Phase）。

![2pc](https://cdn.mazhen.tech/images/202304171538027.png)

1. **准备阶段**（Prepare Phase）
   1. 事务协调器（Transaction Coordinator）向所有事务参与者（Transaction Participants）发送准备（Prepare）消息，要求它们准备提交事务。  
   2. 每个事务参与者在收到准备消息后，会执行本地事务操作（例如修改数据、写日志等），然后将其状态设置为“准备就绪”（Ready）。
   3. 如果事务参与者成功完成了本地操作并准备好提交事务，它会向事务协调者发送一个“同意”（Agree）消息。否则，它会发送一个“中止”（Abort）消息。

2. **提交阶段**（Commit Phase）
   1. 当事务协调者收到所有事务参与者的响应后，会做出全局决策。如果所有参与者都发送了“同意”消息，协调者会决定提交事务。否则，协调者会决定中止事务。
   2. 事务协调者向所有事务参与者发送全局决策，即“提交”（Commit）或“中止”（Abort）消息。
   3. 事务参与者根据协调者的全局决策执行相应的操作。如果接收到“提交”消息，参与者会提交本地事务，并向协调者发送一个“已提交”（Committed）消息；如果接收到“中止”消息，参与者会回滚本地事务，并向协调者发送一个“已中止”（Aborted）消息。

两阶段提交协议的目标是确保分布式事务中的所有参与者要么都提交事务，要么都中止事务，从而满足原子性要求。

两阶段提交协议也有一些局限性，例如性能开销、同步延迟和单点故障风险。

#### coordinator 故障

如果任何一个Prepare请求失败或超时，coordinator 将中止交易；如果任何一个 commit 或 abort 请求失败，coordinator 将无限期地重试它们。如果 coordinator 失败了会怎么样呢？

如果 coordinator 在发送 Prepare 请求之前就失败了，参与者可以安全地中止事务。但是，一旦参与者收到Prepare 请求并投了 "yes"，参与者不能再单方面中止，必须等待 coordinator 的回复，以确定事务是被 commit 还是被 abort。如果 coordinator 在这时崩溃或发生网络故障，事务处于 *in doubt* 状态，参与者除了等待之外什么也做不了。

![coordinator failure](https://cdn.mazhen.tech/images/202304171703938.png)

#### 单点故障风险

`两阶段提交协议`的问题是，一旦事务参与者完成投票，它必须等待 coordinator 给出指示，提交或放弃。如果这时coordinator 挂了，事务参与者除了等待什么也做不了，事务处于未决状态。coordinator 成为了整个系统的**单点故障**。

coordinator 在向参与者发送提交或中止请求之前，必须将事务的最终结果写入到磁盘上的事务日志中。当coordinator 从故障中恢复时，它通过事务日志来确定所有未决状态事务的处理。所以从本质上看，`两阶段提交协议`为了达到一致性，实际上是退化到由 coordinator 单节点来实现 atomic commit。

因为`两阶段提交协议`会在等待 coordinator 恢复的过程中处于阻塞状态，所以它被称为`阻塞原子提交`（*blocking* atomic commit）协议。

#### coordinator 的故障恢复

如果 coordinator 发生故障，如何进行故障恢复呢？有三种解决方法：

* 等待 coordinator 恢复，并接受在此期间系统将被阻塞的事实。
* 由人工选择一个新的 coordinator 节点，进行手动故障切换。
* 使用一个算法来自动选择一个新的 coordinator。

后两种解决方法的前提是，事务日志必须安全可靠的存储，不能因为 coordinator 的任何故障而被损坏。

#### 启发式决策

为了保证原子性，`两阶段提交协议`必须是阻塞的。这意味着，即使存在故障恢复机制，参与者也可能长时间内被阻塞，但一些应用可能无法容忍这种长时间的阻塞。更糟的情况是，如果事务日志丢失或损坏， 即使 coordinator 恢复了也不能决定事务的最终结果。 未决的事务不会自动解决，它们会驻留在数据库中，持有锁并阻塞其他事务。这时即使重启数据库也不能解决问题，因为数据库必须在重新启动时保留对未决事务的锁定，否则将可能违反两阶段提交的原子性保证。

为了打破两阶段提交的阻塞性，事务参与者在没有 coordinator 的明确指示下，独立决定中止或提交一个未决事务，这就是**启发式决策**（heuristic decision）。

启发式决策可能导致数据不一致，因为事务参与者在没有 coordinator 指示的情况下独立决定事务的命运。这可能导致某些参与者提交事务，而另一些参与者中止事务。事实上，启发式决策违反了两阶段提交协议的承诺，因此，做出启发式决策只是用于摆脱灾难性的情况，而不是常规使用。

JTA 定义了几种与启发式决策有关异常。

* **javax.transaction.HeuristicCommitException** coordinator 要求事务参与者回滚，但事务参与者此前已经做出了提交的启发式决策。
* **javax.transaction.HeuristicRollbackException** coordinator 要求事务参与者提交，但事务参与者此前已经做出了回滚的启发式决策。
* **javax.transaction.HeuristicMixedException** 是最糟糕的启发式异常。抛出它表示事务的一部分已提交，而其他部分被回滚。当一些事务参与者进行启发式提交，而其他事务参与者进行启发式回滚时，coordinator会抛出此异常。

### 事务模式

事务模型（Transaction Models）是指在事务处理系统中使用的一组原则和方法，用于定义事务的结构、范围、行为和执行方式。不同的事务模型反映了不同的设计和实现方法，以满足特定的应用需求。以下是三种常见的事务模型。

#### 扁平事务模型

扁平事务模型（**Flat Transaction Model**）是最简单和最常见的事务模型，其中每个事务都是独立的，并且没有任何嵌套或链接关系。扁平事务模型规定在任何给定时间只有一个事务在其他事务中处于活动状态。

我们是否可以在一个事务中同时开始另一个事务？有两种方法可以做到这一点：

* 在第一个事务结束之前，我们可以禁止开始另一个事务。

* 我们也可以暂停当前事务，并开始新事务，在新事务完成后，将恢复原始事务。

扁平事务模型广泛应用于各种数据库系统和应用程序。应用服务器必须支持扁平事务模型。

#### 链式事务模型

在链式事务模型（**Chained Transaction Model**）中，多个事务可以相互链接，使得一个事务的结束与下一个事务的开始紧密相连。当一个事务提交或回滚时，立即启动另一个事务，而不需要显式地发出BEGIN TRANSACTION命令。链式事务模型有助于提高事务处理的效率，尤其是在需要频繁执行事务的应用场景中。

#### 嵌套事务模型

在嵌套事务模型（**Nested Transaction Model**）中，事务可以嵌套在其他事务之内，形成一个层次结构。这意味着一个事务可以包含一个或多个子事务，子事务又可以包含它们自己的子事务。一个嵌套的子事务可以单独提交或中止。因此，复杂的事务可以被分解成更容易管理的子事务。子交易可以提交或回滚，而不需要整个交易提交或回滚。

[JTA（Java Transaction API ）](https://jcp.org/en/jsr/detail?id=907)规范不要求支持嵌套事务模型。大多数 JTA 实现只支持扁平事务模型。

## 分布式事务处理模型

分布式事务是一个涉及到由多个分布式应用程序执行的操作，以及可能涉及多个分布式数据库的事务。在分布式环境中保证事务遵守 ACID 原则是很困难的，需要协调和管理跨越不同系统的事务。对于复杂、异构的分布式系统来说，应用程序必须遵守同一个标准来协调事务工作，以进行分布式事务处理（**DTP**，*distributed transaction processing* ）。其中一个 DTP 标准是由 [Open Group](https://www.opengroup.org/) 开发的 [X/Open DTP](https://pubs.opengroup.org/onlinepubs/9294999599/toc.pdf)。Java EE 中的全局事务处理使用的就是 **X/Open DTP** 模型。**在企业 Java 应用的世界中，X/Open DTP 是事务处理的基石。**

### X/Open DTP

[X/Open](https://en.wikipedia.org/wiki/X/Open) 是一家成立于1984年的非营利性质的技术联盟，其目标是制定开放系统标准，以便于实现操作系统、数据库、网络和分布式计算等领域的互操作性。1996 年，X/Open 与 [Open Software Foundation](https://en.wikipedia.org/wiki/Open_Software_Foundation)合并，组成 [The Open Group](https://www.opengroup.org/)。

X/Open 在1991年开发了一个分布式事务处理（DTP）模型，其中包括传统的 TP monitors 所提供的许多功能。大多数关系型数据库、消息队列都支持基于 X/Open DTP 的规范。该模型将一个交易处理系统分为几个部分：交易管理器、数据库或其他资源管理器以及交易通信管理器

X/Open DTP 模型由**事务管理器**（TM）、**资源管理器**（RM）、**通信资源管理器**（CRM）和**应用程序**（AP）组成。X/Open DTP 标准规定了这些组件功能，以及组件之间的标准接口。

![DTP](https://cdn.mazhen.tech/images/202304191657264.png)

X/Open 的**资源管理器**用于描述任何共享资源的管理进程，但它最常用于表示关系数据库。在 X/Open DTP模型下，**应用程序**和**资源管理器**之间的接口是对于不同的 RM 是不一样的，但是可以使用**资源适配器**作为接口，提供**应用程序**和各种**资源管理器**类进行通信的通用方法，例如  JDBC 可以被认为是资源适配器。

**事务管理器**是 X/Open DTP 模型的核心，负责协调各分布式组件之间事务。**资源管理器**通过实现 [XA 规范](https://publications.opengroup.org/standards/dist-computing/c193)来参与分布式事务。**XA** 规范定义了**事务管理器**（TM）和**资源管理器**（RM）之间的双向接口。**事务管理器**实现了两阶段提交协议，确保所有的**资源管理器**都能同时提交完成事务，或在失败时回滚到原始状态。

**通信资源管理器** 为连接分布式的**事务管理器**提供了一种标准方法，以便在不同事务域之间传播事务信息，实现更广泛的分布式事务。**事务管理器**和**通信资源管理器**之间的标准接口由 [XA+接口](https://publications.opengroup.org/s243) 定义。**通信资源管理器**到**应用程序**的接口由三个不同的接口定义，即[TxRPC](https://pubs.opengroup.org/onlinepubs/009649499/toc.pdf)、[XATMI](https://pubs.opengroup.org/onlinepubs/009649399/toc.pdf) 和 [CPI-C](https://pubs.opengroup.org/onlinepubs/009658099/toc.pdf)。

### X/Open XA

[X/Open XA规范](https://publications.opengroup.org/standards/dist-computing/c193)定义了**事务管理器**（Transaction Manager）与**资源管理器**（Resource Manager）之间的协作机制，以便在分布式环境中实现两阶段提交2PC协议。X/Open XA规范主要包括以下几个组成部分：

* **XA接口**： 这是一组标准的函数和数据结构，用于定义**事务管理器**和**资源管理器**之间的通信方式。XA接口包括一系列函数，如xa_open()、xa_close()、xa_start()、xa_end()、xa_prepare()、xa_commit()、xa_rollback()等，这些函数分别对应分布式事务处理过程中的不同阶段。

* **XID**（Transaction Identifier）：唯一的事务标识符，用于跟踪和管理分布式环境中的事务。XID 包括三个主要部分：全局事务ID（Global Transaction ID）、分支限定符（Branch Qualifier）和格式ID（Format ID）。全局事务ID用于唯一标识一个分布式事务，分支限定符用于标识事务中的不同资源管理器，而格式ID用于指定XID的表示格式。

* **两阶段提交协议**：X/Open XA规范采用两阶段提交协议来实现分布式事务处理。

遵循X/Open XA规范的事务管理器和资源管理器可以跨平台、跨系统地协同工作，实现分布式事务处理的互操作性。

## Java Transaction API (JTA)

[Java Transaction API (JTA)](https://jcp.org/en/jsr/detail?id=907) 是Java平台上的一个事务处理规范，它为 Java 应用程序提供了一组统一的事务处理接口。JTA 是 Java EE 规范的一部分，旨在简化分布式事务处理。JTA 遵循 X/Open DTP模型，将事务管理器和资源管理器的接口抽象为 Java 接口。

JTA 规定了事务管理器和分布式事务系统中涉及的各方之间的 Java 接口：应用程序、资源管理器和应用服务器。

JTA包由两部分组成：

* 应用接口，由应用程序划定事务边界
* 事务管理器接口，由应用服务器控制事务边界的划分

![JTA](https://cdn.mazhen.tech/images/202304191811239.png)

上图显示了 JTA 的三个主要接口，包括 JTA **TransactionManager**、JTA **UserTransaction** 和 JTA XA **XAResource**。该图还显示了 JTA 与 Java事务服务（JTS）的关系。

JTA 组件被定义在 [javax.transaction](https://javaee.github.io/javaee-spec/javadocs/javax/transaction/package-summary.html)和 [javax.transaction.xa](https://docs.oracle.com/en/java/javase/17/docs/api/java.transaction.xa/javax/transaction/xa/package-summary.html) 两个包内。其中 **javax.transaction.xa**

### JTA 事务管理接口

JTA 支持事务管理服务的标准接口，应用服务器主要通过 [TransactionManager](https://javaee.github.io/javaee-spec/javadocs/javax/transaction/TransactionManager.html) 和 [Transaction](https://javaee.github.io/javaee-spec/javadocs/javax/transaction/Transaction.html) 接口来访问这些服务。

![JTA Transaction Manager](https://cdn.mazhen.tech/images/202304200945512.png)

应用服务器使用 **TransactionManager** 接口来管理用户应用程序的事务。 **TransactionManager** 将事务与线程相关联。**TransactionManager**上的**begin()**、**commit()**和**rollback()**方法被应用服务器调用，分别为当前线程开始、提交和回滚事务。**TransactionManager**还支持**setRollbackOnly()**方法，指定对当前线程的事务只支持回滚。**setTransactionTimeout()**方法还以秒为单位定义事务超时，**getStatus()** 方法返回当前线程事务的静态常量 **Status**。

调用 **TransactionManager.getTransaction()** 可以获得当前线程关联的事务对象 **Transaction**。通过调用 **TransactionManager.suspend()** 可以暂停当前事务并获得 **Transaction** 对象， **TransactionManager.resume()** 方法恢复当前事务。

**Transaction**接口表示具体的事务实例。**Transaction**由**TransactionManager**创建，提供了一些与事务相关的方法，如commit()，rollback()和getStatus()等。可以使用 setRollbackOnly() 调用告诉**Transaction**对象仅允许回滚。**enlistResource** 方法用于将 **XAResource** 对象添加到事务上下文中，**delistResource**方法用于将 **XAResource**对象从事务上下文中移除。

**Synchronization**接口用于在事务完成时接收回调通知。调用 **Transaction.registerSynchronization()** 可以将**Synchronization**注册到与当前线程关联的事务中。

**Status** 接口定义了一组静态常量，表示事务的状态。

### JTA 应用接口

JTA 的应用接口是 [UserTransaction](https://javaee.github.io/javaee-spec/javadocs/javax/transaction/UserTransaction.html) ，被应用程序用来控制事务边界。

![JTA Application Interface](https://cdn.mazhen.tech/images/202304201015744.png)

**UserTransaction.begin()**方法可以被应用程序调用，开始一个与应用程序当前线程相关联的事务。

**UserTransaction.commit()** 提交与当前线程关联的事务。**UserTransaction.rollback()** 回滚与当前线程关联的事务。通过调用**UserTransaction.setRollbackOnly()**，设置与当前线程相关的事务只能被回滚。

通过调用**UserTransaction.setTransactionTimeout()**可以设置与事务相关的超时，超时的单位是秒。事务状态**Status**可以通过 **UserTransaction.getStatus()** 获得。

EJB 可以依赖声明式和容器管理事务。但是如果希望 EJB 以编程方式管理自己的事务，就可以利用**UserTransaction**接口。Servlets 和 JSP 也可以利用 **UserTransaction** 接口来划分事务。**UserTransaction** 可以从JNDI查询中获得，或者直接从 EJB 容器环境中获得。

### JTA 和 X/Open XA

X/Open 制定的 [XA 规范](https://publications.opengroup.org/standards/dist-computing/c193) 定义了分布式资源管理器的接口，被 X/Open DTP 模型中的分布式事务管理器访问。JTA 使用 [XAResource](https://docs.oracle.com/en/java/javase/17/docs/api/java.transaction.xa/javax/transaction/xa/XAResource.html) 和 [Xid](https://docs.oracle.com/en/java/javase/17/docs/api/java.transaction.xa/javax/transaction/xa/Xid.html) 接口封装 XA。**TransactionManager** 使用 **XAResource** 接口来管理资源间的分布式事务。

![JTA resource management interfaces](https://cdn.mazhen.tech/images/202304201046370.png)

[Xid](https://docs.oracle.com/en/java/javase/17/docs/api/java.transaction.xa/javax/transaction/xa/Xid.html) 是分布式事务的标识符，可以从 Xid 获取标准的X/Open格式标识符、全局事务标识符字节和分支标识符。

 [XAResource](https://docs.oracle.com/en/java/javase/17/docs/api/java.transaction.xa/javax/transaction/xa/XAResource.html) 接口是事务管理器和资源管理器之间标准 X/Open 接口的 Java 映射。资源管理器的资源适配器必须实现 **XAResource** 接口，使资源能够参与进分布式事务。一个资源管理器的例子是关系数据库，对应的资源适配器就是 JDBC 接口。

**XAResource.start()**方法用于将分布式事务与资源关联。**XAResource.end()**将资源与事务分离。**XAResource**还提供了提交、准备提交、回滚、恢复和遗忘分布式事务的方法。事务超时也可以从XAResource中设置和获取。

## Java Transaction Service (JTS)

**CORBA Object Transaction Service (OTS)** 将分布式事务处理模型（DTP）扩展到了对象领域，它提供了一种在分布式对象系统中进行事务处理的方法。**OTS** 定义了一组标准的接口和协议，允许 CORBA 对象参与分布式事务。**Java Transaction Service (JTS)** 是 **OTS** 的 Java 映射， **JTA** 推荐使用 **JTS** 作为其底层事务系统的实现。

从事务管理器的角度来看，JTA 接口是以 high-level 的形式出现，而 JTS 是事务管理器内部使用的 low-level 接口。**应用服务器间的事务互操作性是通过底层使用  JTS 实现获得的。**

### CORBA

由 [Object Management Group（OMG）](https://www.omg.org/)定义的通用对象请求代理架构（Common Object Request Broker Architecture，CORBA）是一个由包括IBM、BEA和惠普在内的工业联盟制定的标准，它促进了可互操作应用程序的构建，这些应用程序基于分布式对象的概念。

CORBA 使用一个标准的通信模型，在这个模型上，用不同的语言组合实现的客户和服务器，以及在不同的硬件和操作系统平台上运行的客户和服务器可以进行交互。CORBA 体系结构主要包含以下几个部分：

* **对象请求代理（Object Request Broker，ORB）**，它使对象能够在分布式的异质环境中透明地发出和接收请求。这个组件是OMG参考模型的核心。
* **对象服务**，一组支持使用和实现对象功能的服务集合。这些服务是构建分布式应用程序所必需的，例如Object Transaction Service (OTS)。
* 通用设施，应用程序可能需要的其他有用服务。

CORBA 比 Java EE 的出现早了十年，并且不受限于单一的实现语言。在 Java EE 出现之前，CORBA 是企业应用程序的标准开发平台。 EJB 采用的底层分布式对象通信协议是由 CORBA 定义的。EJB 使用 CORBA 通信协议将它们的服务暴露给客户，也可以使用 CORBA 通信协议与其他 EJB 和基于 CORBA 的服务器环境通信。一些 CORBA 服务，如 CORBA 命名服务、CORBA 事务和 CORBA 安全，被 Java EE 标准所接受，作为创建可互操作的 EJB 服务的手段。

### ORB

**ORB**是 CORBA 的核心组件，负责在客户端和服务端之间传递请求和响应。ORB的主要功能包括：

* 为客户端提供透明访问：客户端可以像调用本地对象一样调用远程对象，而不用关心底层通信和数据交换的细节。
* 定位和激活服务对象：ORB负责在分布式系统中查找和激活服务对象，以便客户端能够与它们进行通信。
* 消息封装和解封装：ORB将客户端的请求封装为消息，并在服务端解封装，以便服务对象能够处理请求。响应也会经过类似的处理。
* 系统间通信：ORB处理不同系统间的通信，包括连接管理、错误处理和安全性。
* 跨平台和跨语言：通过 IDL，ORB 可以实现不同编程语言之间的对象互操作。

### GIOP 和 IIOP

GIOP 是一种通用的协议，用于定义分布式系统中不同 ORB之间的通信。GIOP 指定了在 ORB 之间传递的消息格式和通信规则。IIOP 是一种基于 TCP/IP 协议的 GIOP 实现。

GIOP 将 IDL 数据类型映射成二进制数据流，并通过网络发送。GIOP 使用通用数据表示（Common Data Representation ，CDR）语法来完成这一任务，以有效地在IDL数据类型和二进制数据流之间进行映射。

IIOP 将 GIOP 消息数据映射到 TCP/IP 连接行为，以及对输入/输出流的读/写。当一个 CORBA 服务器对象要被分发时，ORB 通过Interoperable Object Reference (IOR) 使网络上唯一识别该对象的信息可用。IOR 包含 CORBA 服务器对象进程的 IP 地址和 TCP 端口。CORBA 客户端利用IOR 建立和CORBA 服务器的连接。

### RMI/IIOP

Java 远程方法调用（JAVA REMOTE METHOD INVOCATION，RMI）框架是Java的分布式对象通信框架。RMI允许客户端和服务器将对象作为方法参数和返回值通过值或引用来传递。如果在方法参数或返回类型中使用的类的类型对客户端或服务器都是未知的，它可以被动态加载。RMI还为分布式垃圾收集提供了一种方法，以清理不再被任何分布式客户端引用的任何分布式服务器对象。

![rmi](https://cdn.mazhen.tech/images/202304201532758.png)

RMI 客户端与实现 Java 接口的对象对话，该接口与特定 RMI 服务器暴露的远程接口相对应。该接口实际上是由 RMI stub实现的，它接受来自 RMI 客户端的调用，并将其打包成可通过网络发送的序列化数据包。同样地，stub 将来自RMI服务器的序列化响应数据包解封为可由RMI客户端使用的Java对象。

Remote Reference Layer 从RMI stub 获取序列化的数据，并处理建立在传输协议之上的 RMI 特定的通信协议。Remote Reference Layer 的职责包括解决RMI服务器的位置，启动连接，以及激活远程服务器。

RMI目前支持两个网络传输协议。**JRMP** 是标准的 RMI 通信信息传递协议。CORBA的 IIOP 消息传输协议现在也可以通过 **RMI/IIOP** 标准扩展来实现。

![RMI/IIOP](https://cdn.mazhen.tech/images/202304201549570.gif)

**JRMP** 是一个非标准的协议，不能实现与跨语言的 CORBA 对象的通信。与 JRMP 不同，**RMI/IIOP** 可以在不同平台和编程语言之间进行通信，因为它使用了 CORBA 的 IIOP 协议。RMI/IIOP 使用 IDL 来定义远程对象的接口，这样不同编程语言的客户端都可以调用远程对象。RMI/IIOP 使用 CORBA 的对象传输方式，而不是 Java 序列化，这样可以实现跨平台和跨编程语言的对象传输。

### OTS

OTS 定义了事务服务实现的接口。OTS 的接口基本上可以分为客户端可用的接口和服务器可用的接口。这些接口之间有一些重叠，因为在某些情况下需要同时提供给客户和服务器。

![OTS](https://cdn.mazhen.tech/images/202304201717219.png)

简要地描述一下这些接口在OTS规范中的作用：

* **Current** 是应用开发者与事务实现的典型交互方式，允许事务的开始和结束。使用 **Current** 创建的事务会自动与调用的线程相关联。底层实现通常会使用 **TransactionFactory** 来创建top-level 事务。OTS 规范允许事务被嵌套。

* **Control** 接口提供对特定事务的访问，实际上包装了事务 **Coordinator** 和 **Terminator** 接口，分别用于 enlist 参与者和结束事务。把这个功能分成两个接口的原因之一是，为了更精细的控制可终止事务的实体。
* **Resource**/**SubtransactionAwareResource** 接口代表事务参与者，可以兼容任何两阶段提交协议的实现，包括 X/Open XA。
* 每个top-level 事务都有一个相关的**RecoveryCoordinator**，参与者可以使用它来进行故障恢复。
* **Transaction Context** 主要作用是存储和传递与当前事务相关的信息。通过使用 **Transaction Context**，OTS 中的事务参与者可以共享同一事务上下文，从而实现对事务的正确协调和管理。

使用 OTS 接口进行事务划分和传播时，有两种使用模式：

* Indirect/Implicit 模式，事务使用 **Current** 接口创建、提交和回滚事务。事务传播根据目标对象 POA 中的策略自动进行。
* Direct/Explicit 模式，事务使用 **TransactionFactory** 创建，并使用 **Control** 对象进行提交或回滚。事务传播是通过向每个 IDL 操作添加参数（例如，事务的控制对象）来完成。

大多数应用程序的首选 Indirect/Implicit 模式，Direct/Explicit 模式提供了更大的灵活性，但更难管理。

### JTS

**Java Transaction Service**（JTS）规范是 **OTS** 规范的Java语言映射。使用符合 JTS 的实现在理论上允许与其他 JTS 实现的互操作。

JTS API 通过规范提供 [IDL](https://github.com/eclipse-ee4j/orb/tree/master/omgapi/src/main/idl) 生成，主要的接口在 org.omg.CosTransactions 和 org.omg.CosTSPortability 包中。 Java 应用服务器通过 JTA 接口访问事务管理功能，JTA 通过 JTS 与事务管理的实现进行交互。同样，JTS 可以通过 JTA XA 接口访问资源，也可以访问启用 OTS 的非 XA 资源。JTS 实现可以通过 CORBA OTS 接口进行互操作。JTS 必须支持扁平事务模型。JTS 可以支持嵌套事务模型，但不是必需的。

![jts](https://cdn.mazhen.tech/images/202304201812890.png)

从 Transaction Manager 的角度来看，JTS 的实现是不需要公开。上图 Transaction Manager 框中的虚线说明了JTA 和 JTS 之间的专用接口，允许 JTA 与底层 OTS 实现进行交互。

JTS 使用 CORBA OTS 接口来实现互操作性和可移植性（即通过 CosTransactions 和 CosTSPortability），这些接口为利用 IIOP 在 JTS 之间生成和传播事务上下文的实现定义了标准机制。

总之，**JTA** 是暴露给用户和应用服务器使用的接口，应用服务器内部可以使用 **JTS** 作为其底层事务系统的实现，应用服务器间的事务互操作性是通过底层使用  **JTS** 实现获得的。
