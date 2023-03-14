# Nginx的基本使用

## Nginx 常用命令

Nginx的指令格式为 `nginx [options argument]`。

* **查看帮助**

```shell
./sbin/nginx  -?
```

* **使用指定的配置文件**

```shell
./sbin/nginx -c filename
```

* **指定运行目录**

```shell
./sbin/nginx -p /home/mazhen/nginx/
```

* **设置配置指令**，覆盖配置文件中的指令

```shell
./sbin/nginx -g directives
```

* **向 Nginx 发送信号**

我们可以向 Nginx 进程发送信号，控制运行中的 Nginx。一种方法是使用 `kill` 命令，也可以使用 `nginx -s` ：

```shell
# 重新加载配置
$ ./sbin/nginx -s reload

# 立即停止服务
$ ./sbin/nginx -s stop

# 优雅停止服务
$ ./sbin/nginx -s quit

# 重新开始记录日志文件
$ ./sbin/nginx -s reopen
```

* **测试配置文件是否有语法错误**

```shell
./sbin/nginx  -t/-T
```

* **打印nginx版本**

```shell
./sbin/nginx  -v/-V
```

## 热部署

在不停机的情况下升级正在运行的 Nginx 版本，就是热部署。

首先查看正在运行的 Nginx：

```shell
$ ps aux | grep nginx
mazhen      4376  0.0  0.0   9896  2372 ?        Ss   16:47   0:00 nginx: master process ./sbin/nginx
mazhen      4402  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4403  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4404  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4405  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4407  0.0  0.0  12184  2316 pts/0    S+   16:51   0:00 grep --color=auto nginx
```

备份现有 Nginx 的二进制文件：

```shell
cp nginx nginx.old
```

将构建好的最新版 Nginx 的二进制文件拷贝到 `$nginx/sbin` 目录：

```shell
cp ~/works/nginx-1.22.1/objs/nginx ~/nginx/sbin/ -f
```

给正在运行的Nginx的 master 进程发送信号，通知它我们要开始进行热部署：

```shell
kill -USR2  4376
```

这时候 Nginx master 进程会使用新的二进制文件，启动新的 master 进程。新的 master 会生成新的 worker，同时，老的worker并没有退出，也在运行中，但不再监听 80/443 端口，请求会平滑的过度到新 worker 中。

```shell
$ ps aux | grep nginx
mazhen      4376  0.0  0.0   9896  2536 ?        Ss   16:47   0:00 nginx: master process ./sbin/nginx
mazhen      4402  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4403  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4404  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4405  0.0  0.0  10324  2104 ?        S    16:50   0:00 nginx: worker process
mazhen      4454  0.0  0.0   9768  6024 ?        S    16:59   0:00 nginx: master process ./sbin/nginx
mazhen      4455  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4456  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4457  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4458  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4461  0.0  0.0  12184  2436 pts/0    S+   16:59   0:00 grep --color=auto nginx
```

向老的 Nginx master 发送信号，让它优雅关闭 worker 进程。

```shell
kill -WINCH 4376
```

这时候再查看 Nginx 进程：

```shell
$ ps aux | grep nginx
mazhen      4376  0.0  0.0   9896  2536 ?        Ss   16:47   0:00 nginx: master process ./sbin/nginx
mazhen      4454  0.0  0.0   9768  6024 ?        S    16:59   0:00 nginx: master process ./sbin/nginx
mazhen      4455  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4456  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4457  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4458  0.0  0.0  10216  1980 ?        S    16:59   0:00 nginx: worker process
mazhen      4475  0.0  0.0  12184  2292 pts/0    S+   17:07   0:00 grep --color=auto nginx
```

老的 worker 已经优雅退出，所有的请求已经切换到了新升级的 Nginx 中。

老的 master 仍然在运行，如果需要，我们可以向它发送 reload 信号，回退到老版本的 Nginx。

## 日志切割

首先使用 `mv` 命令，备份旧的日志：

```shell
mv access.log  bak.log
```

Linux 文件系统中，改名并不会影响已经打开文件的写入操作，因为内核 inode 不变，这样操作不会出现丢日志的情况。

然后给运行中的 Nginx 发送 reopen 信号：

```shell
./nginx -s reopen
```

Nginx 会重新生成 `access.log` 日志文件。

一般会写一个 `bash` 脚本，通过配置 crontab，每日进行日志切割。
