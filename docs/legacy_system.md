# 遗留系统

<img src="https://cdn.mazhen.tech/images/202405141501496.webp" alt="遗留系统" style="zoom:50%;" />

最近在处理一个 [EJB](https://www.oracle.com/java/technologies/enterprise-javabeans-technology.html) 调用的问题，和底层的 [CORBA](https://www.corba.org/) 通信有关，都是很古老的技术名词。

二十多年前我刚参加工作的时候，EJB 带着神秘和时髦的色彩横空出世，可后来没几年就被 Spring Framework 祛魅，很少有人再使用 EJB 开发应用。

CORBA 则更加古老，估计现在很多程序员都没听说过，更别说开发过 CORBA 组件了。实际上 CORBA 是最早的分布式服务规范，早在 1991 年就发布了 1.0。可以说后来的 EJB，Web Services，甚至微服务，service mesh 都有 CORBA 的影子。

* CORBA 定义了 [IDL（Interface Definition Language）](https://www.corba.org/omg_idl.htm)，用它来描述对象的接口、方法、参数和返回类型等信息，根据 IDL 可以生成各种语言的实现，不同语言编写的对象可以进行交互。
* CORBA 定义了一系列服务，如[Naming Service](https://www.omg.org/spec/NAM/1.3)，[Transaction Service](https://www.omg.org/spec/TRANS/1.4)，[Security Service](https://www.omg.org/spec/SEC/1.8)等，作为分布式系统的基础服务。事务、安全等服务会随着远程调用进行传播。
* CORBA 的 [ORB（Object Request Broker）](https://www.corba.org/orb_basics.htm) 负责分布式系统中对象之间的通信。用户可以像调用本地对象一样调用远程对象上的方法，ORB 会处理网络通信和远程调用的细节。ORB 之间通过 [IIOP（Internet Inter-ORB Protocol）](https://en.wikipedia.org/wiki/General_Inter-ORB_Protocol)协议进行通信。

就这样，IDL、一系列服务，再加上ORB，构成了 CORBA 的完整体系。其实 CORBA 的理念很好，面向对象，跨语言跨平台，服务传播和网络通信对用户透明。

CORBA 作为一套成熟的工业规范，后来者自然会想办法吸收和兼容。1998年发布的 [JDK 1.2](https://web.archive.org/web/20070816170028/http://www.sun.com/smi/Press/sunflash/1998-12/sunflash.981208.9.xml)，内置了 Java IDL ，以及全面兼容 ORB 规范的 Java ORB 实现。这时 Java 已经准备在企业端开发领域大展拳脚，JDK 内置了对 CORBA 的支持，为  [J2EE](https://www.oracle.com/java/technologies/appmodel.html) （也就是后来的 [Java EE](https://www.oracle.com/java/technologies/java-ee-glance.html)，现在的 [Jakarta EE](https://jakarta.ee/)）做好了准备。

EJB 全面继承了 CORBA，[Java Transaction Service (JTS)](https://web.archive.org/web/20080418135325/http://java.sun.com/javaee/technologies/jts/) 是 CORBA 事务服务 [OTS](https://www.omg.org/spec/TRANS/1.4) 的 Java 映射，EJB 之间的远程调用走 [RMI/IIOP](https://docs.oracle.com/javase/7/docs/technotes/guides/rmi-iiop/index.html) 协议，事务、安全上下文会通过 IIOP 进行传播。理论上，部署在不同品牌应用服务器上的 EJB 之间可以互相调用，EJB 也可以和任何语言开发的 CORBA 对象进行交互，并且所有 EJB 和 CORBA 对象，可以运行在同一个事务、安全上下文中。

EJB 的目标是做真正的中间件，连接不同厂商的 J2EE 应用服务器，连接不同语言开发、运行在不同平台上的 CORBA 对象，并且它们可以加入到同一个分布式事务中，受到同样的安全策略保护。

理想很丰满，现实是 EJB 的理想从来没有被实现过。[Rod Johnson](https://twitter.com/springrod) 在总结了 J2EE 的优缺点后，干脆[抛弃了 EJB(without EJB)](https://www.oreilly.com/library/view/expert-one-on-onetm-j2eetm/9780764558313/) ，开发了轻量级 Spring Framework。Spring 太成功了，以至于对很多人来说，Java 开发 ≈ 使用 Spring 进行开发。

后来的 Web Services/SOA 又把 CORBA、EJB 的路重新走了一遍， 定义了和 IDL 类似的 [WSDL](https://en.wikipedia.org/wiki/Web_Services_Description_Language)，以及一系列的事务规范 [WS-Transaction](https://en.wikipedia.org/wiki/WS-Transaction)，[WS-Coordination](https://en.wikipedia.org/wiki/WS-Coordination)，[WS-Atomic Transaction](https://en.wikipedia.org/wiki/WS-Atomic_Transaction)。然后开发者又觉得大公司定义的规范太复杂，才有了轻量级的 REST，微服务。

Java 的 CORBA 实现在  JDK 9 中被标记为 `deprecated`， 并最终在2018年发布的 JDK 11中被[正式移除](https://openjdk.org/jeps/320)。EJB 和 CORBA 都没有成功，Java 宣告和 CORBA 分手，一段历史结束。

在2024年的今天，有着30多年历史的 CORBA 和20多年历史的 EJB 已经是遗留系统，不会再有大批聪明的年轻人愿意投入到这个技术领域。不过对于像我这样还在一线搬砖的大龄程序员，遗留系统也是一种选择。它们和自身的境况很像：激情已过，一天天的老去。我们互相扶持着，每天对它们进行修修补补，打着补丁，它们也回报勉强够养家的报酬，然后一起等待着被淘汰的一天。在 AI 革命，号称要取代码农的今天，竟然还能靠着 20 多年前学到的技能挣到工资，也算一个小小的奇迹吧。

在翻阅 [Java ORB](https://github.com/eclipse-ee4j/orb) 的源代码时，注意到了很多源文件上都标记了作者的名字，于是顺手在网上一搜，还真找到了作者的信息。

<img src="https://cdn.mazhen.tech/images/202405141407035.jpg" alt="Harold Carr" style="zoom:50%;" />

[Harold Carr](http://haroldcarr.com/)，84年至94年在惠普实验室从事分布式 C++ 工作，94年加入 Sun，设计了 Sun 的 CORBA ORB，JAX-WS 2.0，负责过 GlassFish 中的 RMI-IIOP 负载平衡和故障切换。Sun 被收购后他一直留在 Oracle，目前仍在 Oracle 实验室从事技术工作。同时他组过乐队录过专辑，还出版过诗集。有意思的人，有意思的经历。

回到开头的 EJB 问题，仍然没有头绪，继续在代码里找线索。生活充满了眼前的苟且，还能有诗和远方吗？
