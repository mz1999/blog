
# 在国产飞腾平台上编译安装nginx

[飞腾芯片](http://www.phytium.com.cn) + [银河麒麟OS](http://www.kylinos.cn/)是目前国产自主可控市场上的主流基础平台。飞腾芯片是[aarch64](https://en.wikipedia.org/wiki/ARM_architecture#AArch64)架构，是支持64位的[ARM](https://en.wikipedia.org/wiki/ARM_architecture)芯片。银河麒麟是基于Ubuntu的发行版。因此可以认为`飞腾芯片` + `银河麒麟OS`相当于 `ARM64` + `Ubuntu`。

本文介绍在飞腾平台上编译安装nginx的步骤。

* **下载nginx源码**

从[http://nginx.org/en/download.html](http://nginx.org/en/download.html)下载当前稳定版本的源码，例如

```
wget http://nginx.org/download/nginx-1.14.2.tar.gz
```

解压nginx源码：

```
tar -zxvf nginx-1.14.2.tar.gz
```

* **nginx配置文件的语法高亮**

将nginx源码目录下`contrib/vim/`的所有内容，copy到用户的`$HOME/.vim`目录，可以实现nginx配置文件在`vim`中的语法高亮。

```
mkdir ~/.vim
cp -r contrib/vim/*  ~/.vim/
```

再使用`vim`打开`nginx.conf`，可以看到配置文件已经可以语法高亮。

* **编译前的配置**

查看编译配置支持的参数

```
$ ./configure --help | more

--help                             print this message

  --prefix=PATH                      set installation prefix
  --sbin-path=PATH                   set nginx binary pathname
  --modules-path=PATH                set modules path
  --conf-path=PATH                   set nginx.conf pathname
  --error-log-path=PATH              set error log pathname
  --pid-path=PATH                    set nginx.pid pathname
  --lock-path=PATH                   set nginx.lock pathname

  --user=USER                        set non-privileged user for
                                     worker processes
  --group=GROUP                      set non-privileged group for
                                     worker processes

  --build=NAME                       set build name
  --builddir=DIR                     set build directory

  --with-select_module               enable select module
  --without-select_module            disable select module
  --with-poll_module                 enable poll module
  --without-poll_module              disable poll module

  ......

  --with-libatomic                   force libatomic_ops library usage
  --with-libatomic=DIR               set path to libatomic_ops library sources

  --with-openssl=DIR                 set path to OpenSSL library sources
  --with-openssl-opt=OPTIONS         set additional build options for OpenSSL

  --with-debug                       enable debug logging
```

`--with`开头的模块缺省不包括在编译结果中，如果想使用需要在编译配置时显示的指定。`--without`开头的模块则相反，如果不想包含在编译结果中需要显示设定。

例如我们可以这样进行编译前设置：

```
./configure --prefix=/home/adp/nginx  --with-http_ssl_module
```

设置了nginx的安装目录，需要`http_ssl`模块。

如果报错缺少`OpenSSL`，需要先安装`libssl`。在`/etc/apt/sources.list.d`目录下增加支持`ARM64`的apt源，例如国内的清华，创建`tsinghua.list`，内容如下：

```
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb-src http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
```

执行命令安装`OpenSSL`：

```
$ sudo apt-get update
$ sudo apt-get install libssl-dev
```

`configure`命令执行完后，会生成中间文件，放在目录`objs`下。其中最重要的是`ngx_modules.c`文件，它决定了最终那些模块会编译进`nginx`。

* 执行编译

在nginx目录下执行`make`编译：

```
$ make
```

编译成功的`nginx`二进制文件在`objs`目录下。如果是做nginx的升级，可以直接将这个二进制文件copy到nginx的安装目录中。

* 安装

在nginx目录下执行`make install`进行安装：

```
$ make install
```

安装完成后，我们到`--prefix`指定的目录中查看安装结果：

```
$ tree -L 1 /home/adp/nginx

nginx/
├── conf
├── html
├── logs
└── sbin
```

* 验证安装结果

编辑`nginx/conf/nginx.conf`文件，设置监听端口为`8080`：

```
http {
    ...

    server {
        listen       8080;
        server_name  localhost;
...
```

启动nginx

```
./sbin/nginx
```

访问默认首页：

```
$ curl -I http://localhost:8080

HTTP/1.1 200 OK
Server: nginx/1.14.2
Date: Tue, 02 Apr 2019 08:38:02 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 02 Apr 2019 08:30:04 GMT
Connection: keep-alive
ETag: "5ca31d8c-264"
Accept-Ranges: bytes
```

其他常用命令：

```
# 查看帮助
$ ./sbin/nginx  -?

# 重新加载配置
$ ./sbin/nginx -s reload

# 立即停止服务
$ ./sbin/nginx -s stop

# 优雅停止服务
$ ./sbin/nginx -s quit

# 测试配置文件是否有语法错误
$ ./sbin/nginx  -t/-T

# 打印nginx版本、编译信息
$ ./sbin/nginx  -v/-V
```