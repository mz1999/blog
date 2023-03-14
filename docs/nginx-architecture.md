# Nginx架构基础

## Nginx 进程模型

![process model](https://cdn.mazhen.tech/images/202211161536855.png)

Nginx其实有两种进程结构，一种是单进程结构，一种是多进程结构。单进程结构只适合我们做开发调试，在生产环境下，为了保持 Nginx 足够健壮，以及可以利用到 CPU 的多核特性，我们用到的是多进程架构的Nginx。

多进程架构的Nginx，有一个父进程 master process，master 会有很多子进程，这些子进程分为两类，一类是worker 进程，一类是 cache 相关的进程。

在一个四核的Linux服务器上查看Nginx进程：

```shell
$ ps -ef --forest | grep nginx
mazhen     20875   13073  0 15:25 pts/0    00:00:00          |   \_ grep --color=auto nginx
mazhen     20862       1  0 15:25 ?        00:00:00 nginx: master process ./sbin/nginx
mazhen     20863   20862  0 15:25 ?        00:00:00  \_ nginx: worker process
mazhen     20864   20862  0 15:25 ?        00:00:00  \_ nginx: worker process
mazhen     20865   20862  0 15:25 ?        00:00:00  \_ nginx: worker process
mazhen     20866   20862  0 15:25 ?        00:00:00  \_ nginx: worker process
mazhen     20867   20862  0 15:25 ?        00:00:00  \_ nginx: cache manager process
mazhen     20868   20862  0 15:25 ?        00:00:00  \_ nginx: cache loader process
```

可以看到，Nginx 的 master 进程创建了4个 worker 进程，以及用来管理磁盘内容缓存的缓存helper进程。

为什么Nginx使用的是多进程结构，而不是多线程结构呢？因为多线程结构，线程之间是共享同一个进程地址空间，当某一个第三方模块出现了地址空间的断错误时，会导致整个Nginx进程挂掉，而多进程模型就不会出现这样的问题，Nginx的第三方模块通常不会在 master 进程中加入自己的功能代码。

**master** 进程执行一些特权操作，比如读取配置以及绑定端口，它管理 worker 进程的，负责监控每个 worke进程是否在正常工作，是否需要重载配置文件，以及做热部署等。

**worker** 进程处理真正的请求，从磁盘读取内容或往磁盘中写入内容，以及与上游服务器通信。

**cache manager** 进程会周期性地运行，从磁盘缓存中删除条目，以保证缓存没有超过配置的大小。

**cache loader** 进程在启动时运行，用于将磁盘上的缓存加载到内存中，随后退出。

Nginx 采用了事件驱动的模型，它希望 worker 进程的数量和 CPU 一致，并且每一个 worker 进程与某一颗CPU绑定，worker 进程以非阻塞的方式处理多个连接，减少了上下文切换，同时更好的利用到了 CPU 缓存，减少缓存失效。

## 请求处理流程

![request process](https://cdn.mazhen.tech/images/202211171050526.png)

Nginx 使用的是非阻塞的事件驱动处理引擎，需要用状态机来把这个请求正确的识别和处理。Nginx 内部有三个状态机，分别是处理4层 TCP 流量的传输层状态机，处理7层流量的HTTP状态机和处理邮件的email 状态机。

worker 进程首先等待监听套接字上的事件，新接入的连接会触发事件，然后连接分配到一个状态机。

![state machine](https://cdn.mazhen.tech/images/202211171113018.png)

状态机本质上是告诉 Nginx 如何处理请求的指令集。解析出的请求是要访问静态资源，那么就去磁盘加载静态资源，更多的时候 Nginx 是作为负载均衡或者反向代理使用，这个时候请求会通过4层或7层协议，传输到上游服务器。对于每一个处理完成的请求，Nginx会记录 access 日志和 error 日志。

## Nginx 进程管理

Linux 上多进程之间进行通讯，可以使用共享内存和信号。Nginx 在做进程间的管理时，使用了信号。我们可以使用 kill 命令直接向 master 进程和 worker 进程发送信号，也可以使用 nginx 命令行。

master 进程接收处理的信号：

* **CHLD**  在 Linux 系统中，当子进程终止的时候，会向父进程发送 CHLD 信号。master 进程启动的 worker 进程，所以 master 是 worker 的父进程。如果 worker 进程由于一些原因意外退出，那么 master 进程会立刻收到通知，可以重新启动一个新的 worker进程。

* **TERM** 和 **INT**  立刻终止 worker 和 master 进程。
* **QUIT** 优雅的停止 worker 和 master 进程。worker 不会向客户端发送 reset 立即结束连接。
* **HUP** 重新加载配置文件
* **USR1** 重新打开日志文件，做日志文件的切割
* **USR2** 通知 master 开始进行热部署
* **WINCH** 在热部署过程中，通知旧的 master ，让它优雅关闭 worker 进程

我们也可以通过 `nginx -s` 命令向 master 进程发送信号。在 Nginx 启动过程中， Nginx 会把 master 的 PID 记录在文件中，这个文件的默认位置是 `$nginx/logs/nginx.pid` 。 当我们执行 `nginx -s` 命令时，`nginx` 命令会去读取 `nginx.pid` 文件中 master 进程的 PID，然后向 master 进程发送对应的信号。下面是 `nginx -s` 命令对应的信号：

* **reload** - HUP
* **reopen** - USR1
* **stop** - TERM
* **quit** -  QUIT

使用 `nginx -s`  和 直接使用 kill 命令向 master 进程发送信号，效果是一样的。

注意，**USR2** 和 **WINCH** 没有对应的 `nginx -s` 命令，只能通过 kill 命令直接向 master 进程发送。

worker 进程能接收的信号：

* TERM 和 INT
* QUIT
* USR1
* WINCH

worker 进程收到这些信号，会产生和发给 master 一样的效果。但我们通常不会直接向 worker 进程发送信号，而是通过 master 进程来管理 worker 进程，master 进程收到信号以后，会再把信号转发给 worker 进程。

## Nginx 配置更新流程

![reload](https://cdn.mazhen.tech/images/202211171726947.png)

当更改了 Nginx 配置文件后，我们都会执行 `nginx -s reload` 命令重新加载配置文件。Nginx 不会停止服务，在处理新的请求的同时，平滑的进行配置文件的更新。

执行 `nginx -s reload` 命令，会向 master 进程发送 SIGHUP 信号。当 master 进程接收 SIGHUP信号后，会做如下处理：

1. 检查配置文件语法是否正确。
2. master 加载配置，启动一组新的 worker 进程。这些 worker 进程马上开始接收新连接和处理网络请求。子进程可以共享使用父进程已经打开的端口，所以新的 worker 可以和老的worker监听同样的端口。
3. master 向旧的 worker 发送 QUIT 信号，让旧的 worker 优雅退出。
4. 旧的 worker 进程停止接收新连接，完成现有连接的处理后结束进程。

## Nginx 热部署流程

![hot deploy](https://cdn.mazhen.tech/images/202211171739417.png)

Nginx 支持热部署，在升级的过程中也实现了高可用性，不导致任何连接丢失，停机时间或服务中断。热部署的流程如下：

1. 备份旧的 nginx 二进制文件，将新的nginx二进制文件拷贝到 `$nginx_home/sbin`目录。
2. 向 master 进程发送 USR2 信号。
3. master 进程用新的nginx文件启动新的master进程，新的master进程会启动新的worker进程。
4. 向旧的 master 进程发送 WINCH 信号，让它优雅的关闭旧的 worker 进程。此时旧的 master 仍然在运行。
5. 如果想回滚到旧版本，可以向旧的 master 发送 HUP 信号，向新的master 发送QUIT信号。
6. 如果一切正常，可以向旧的 master 发送 QUIT 信号，关闭旧的 master。
