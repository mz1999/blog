# 应用服务器整合第三方连接池

数据库连接池是应用服务器的基本功能，但有时用户因为性能、监控等需求，想使用第三方的连接池。如果只是使用第三方连接池管理数据库连接，那么直接在应用中引入就可以了，但如果用户同时还需要应用服务器的分布式事务和安全服务，就没那么简单了。首先了解一下基本概念。

## Connection

从 JDBC driver 的角度来看，**Connection** 表示客户端会话。应用程序可以通过以下两种方式获取连接：

* **DriverManager** 最初的JDBC 1.0 API中被引入，当应用程序首次尝试通过指定URL连接到数据源时，`DriverManager`将自动加载在 CLASSPATH 中找到的任何 JDBC driver。
* **DataSource** 是在 JDBC 2.0 可选包API中引入的接口。它允许应用程序对底层数据源的细节是透明的。`DataSource` 对象的属性被设置为表示特定数据源。当调用其 `getConnection`方法时，`DataSource` 实例将返回到该数据源的连接。通过简单地更改DataSource对象的属性，可以将应用程序定向到不同的数据源；无需更改应用程序代码。同样，可以更改 `DataSource` 实现而不更改使用它的应用程序代码。

JDBC 还定义了 `DataSource` 接口的两个重要扩展：  

* **ConnectionPoolDataSource** - 支持物理连接的缓存和重用，从而提高应用程序的性能和可扩展性  
* **XADataSource** - 提供可以参与分布式事务的连接

![datasource](https://cdn.mazhen.tech/images/202303101440425.png)

从 **DriverManager** 和 **DataSource** 都可以获得 **Connection**。

**DataSource**、**ConnectionPoolDataSource** 和 **XADataSource** 都继承自 **CommonDataSource**，但它们之间没有继承关系。

从 **ConnectionPoolDataSource** 获得的是 **PooledConnection**，**PooledConnection** 并没有继承 **Connection**，但可以获得**Connection**。

从 **XADataSource** 获得的是 **XAConnection**，**XAConnection** 继承了 **PooledConnection**，除了能获得 **Connection**，还可以获得 **XAResource**。

## Application Server DataSource

应用服务器会为其客户端提供了一个 **DataSource** 接口的实现，并通过 JNDI 暴露给用户。这个 DataSource 包装了 jdbc driver 连接数据库的能力，并在此基础上提供连接池、事务和安全等服务。

在配置应用服务器的 DataSource 时，一般需要指定 Connection 的获取方式：

* java.sql.Driver

* javax.sql.DataSource

* javax.sql.ConnectionPoolDataSource

* javax.sql.XADataSource

这四种连接获取方式都是 JDBC driver 提供的能力，Driver 和 DataSource 是最基本方式。如果应用服务器的 DataSource 想要具备连接池化、分布式事务等服务，除了自身要实现这些功能以外，还需要底层 driver 提供相应的能力配合。

### ConnectionPoolDataSource

以连接池为例，JDBC driver 提供了 **ConnectionPoolDataSource** 的实现，应用服务器使用它来构建和管理连接池。客户端在使用相同的 JNDI 和 DataSource API 的同时获得更好的性能和可扩展性。

<img src="https://cdn.mazhen.tech/images/202303101503461.png" alt="connection pool" style="zoom: 33%;" />

应用服务器维护维护一个从 **ConnectionPoolDataSource** 对象返回的 **PooledConnection** 对象池。应用服务器的实现还可以向 PooledConnection 对象注册**ConnectionEventListener**，以获得连接事件的通知，如连接关闭和错误事件。

<img src="https://cdn.mazhen.tech/images/202303160947605.png" alt="ConnectionPoolDataSource" style="zoom: 50%;" />

我们看到，应用程序客户端通过 JNDI 查找一个 DataSource 对象，并请求从 DataSource 获得一个连接。当连接池没有可用连接时，DataSource 的实现从 JDBC driver 的 ConnectionPoolDataSource 中请求一个新的 PooledConnection 。应用服务器的 DataSource 实现会向 PooledConnection 注册一个ConnectionEventListener，随后获得一个新的 Connection 对象。应用客户端在完成操作后调用 `Connection.close()`，会生成一个 ConnectionEvent 实例，该实例会返回给应用服务器的数据源实现。在收到连接关闭的通知后，应用服务器可以将连接对象放回连接池中。

注意 **ConnectionPoolDataSource** 本身不是连接池，它是供连接池使用的 datasource。一般 JDBC driver 提供的 ConnectionPoolDataSource 实现并没有内置连接池功能，需要配合应用服务器或其他第三方连接池一起使用。可以参考  [MySQL Connector 的文档](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-usagenotes-j2ee-concepts-connection-pooling.html)。

### XADataSource

同样，如果想要分布式事务支持，应用服务器的 DataSource  需要依赖 driver 提供的 **XADataSource** 实现，同时通过 XAResource 和 Transaction Manager 交互。

<img src="https://cdn.mazhen.tech/images/202303101754420.png" alt="XADataSource" style="zoom:33%;" />

**XADataSource** 对象返回 **XAConnection** ，该对象扩展了 PooledConnection ，增加了对分布式事务的参与能力。应用服务器的 DataSource 实现在XAConnection 对象上调用 getXAResource() 以获得传递给事务管理器的 **XAResource** 对象。事务管理器使用 XAResource 来管理分布式事务。

就像池化连接一样，这种分布式事务管理的标准API对应用程序客户端也是透明的。因此，应用服务器可以使用不同 JDBC driver 实现的XADataSource， 来组装可扩展的分布式事务支持的数据访问方案。

![XADataSource](https://cdn.mazhen.tech/images/202303161008074.png)

## 直接整合外部连接池

如果想在应用服务器中直接整合第三方的连接池实现是比较困难的，下面分析一下原因。

JTA 规范要求连接必须能够同时处理多个事务，这个功能被称为事务多路复用或事务交错。我们看一个例子：

``` java
 1. UserTransaction ut = getUserTransaction(); 
 2. DataSource ds = getDataSource(); 
 3.  
 4. ut.begin(); 
 5. 
 6. Connection c1 = ds.getConnection(); 
 7.  // do some SQL 
 8. c1.close(); 
 9. 
10. ut.commit(); 
```

在第8行，连接将释放回连接池，另外一个线程就可以通过 `getConnection()` 获得刚释放的连接。但此时 c1 上的事务还没有提交，如果被其他线程获取，就有可能加入另一个事务，这就是为什么连接必须能够一次支持多个事务。

大多数数据库都不支持事务多路复用，那么一种变通的做法是**让事务独占连接**，在 JTA 事务完成之前，连接不要释放连接回池中。

因此，需要应用服务器的连接池实现能感知到事务，在第8行不会释放连接，而是连接被标记为关闭。在第10行事务提交后，标记为已关闭的所有连接才释放回连接池。

现实中，应用服务器管理的连接池都是能够感知事务的存在，并通过 XAResource 和 Transaction Manager 进行交互：

![JTA Transaction](https://cdn.mazhen.tech/images/202303102116711.png)

另外，应用服务器都实现了对 JCA（Java EE Connector Architecture）规范的支持。JCA 将应用服务器的事务、安全和连接管理等功能，与事务资源管理器集成，定义了一个标准的 SPI(Service Provider Interface) ，因此，一般应用服务器的连接池都在 JCA 中实现，JDBC DataSource 作为一种资源，被 JCA 统一管理。

<img src="https://cdn.mazhen.tech/images/202303102216362.png" alt="jca" style="zoom:50%;" />

由于外部连接池不能感知事务的存在，所以没办法做到事务对连接的独占，因此应用服务器不能直接整合第三方的连接池。

## 解决方案

如果外部连接池实现了 XADataSource，那么我们可以把它当作普通的  JDBC driver，在配置应用服务器的 DataSource 时使用。需要注意两点：

* 为外部连接池配置真正的 JDBC driver 时，要使用 driver的 XADataSource 作为连接的获取方式

* 外部连接池作为特殊的 driver，已经内置了池化功能，连接池的相关参数最好和应用服务器的DataSource保持一致，因为连接池的实际大小受到外部连接池的约束。

这个解决方案的问题是，应用服务器和外部连接池都对连接做的池化，实际上是建立了两个连接池，存在很大的浪费。变通的做法是，设置应用服务器连接池的空闲连接数为0，尽可能的将连接释放回外部连接池。当然更优的做法是，对外部连接池进行适当改造，让它能感知事务的存在，例如 [Agroal](https://github.com/agroal/agroal) 连接池能够和 Transaction Manager进行整合。
