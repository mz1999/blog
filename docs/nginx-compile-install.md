# Nginx的编译和安装

[Nginx](https://nginx.org/) 是最流行的Web服务器，根据 [W3Techs](https://w3techs.com) 最新的统计，[世界上三分之一的网站在使用Nginx](https://w3techs.com/technologies/overview/web_server)。

## 准备工作

### Linux 版本

Nginx 需要 Linux 的内核为 2.6 及以上的版本，因为Linux 内核从 2.6 开始支持 epoll。可以使用 `uname -a` 查看 Linux 内核版本：

```shell
$ uname -a
Linux mazhen-laptop 5.15.0-52-generic #58-Ubuntu SMP Thu Oct 13 08:03:55 UTC 2022 x86_64 x86_64 x86_64 GNU/Linux
```

从输出看到内核版本为 5.15.0，满足要求。现在已经很难找到内核 2.6 以下的服务器了吧。

### 安装依赖

为了编译 Nginx 源码，需要安装一些依赖包。本文以 Ubuntu 为例。

1. **GCC编译器**
GCC（GNU Compiler Collection）是必需的编译工具。使用下面的命令安装：

```shell
sudo apt install build-essential
```

`build-essential` 是所谓的 `meta-package`，包含了 g++/GNU 编译器集合，GNU调试器，以及一些编译程序所需的工具和库。

2. **PCRE库**
PCRE库支持正则表达式。如果我们在配置文件nginx.conf中使用了正则表达式，那么在编译Nginx时就必须把PCRE库编译进 Nginx，因为 Nginx的 HTTP 模块需要靠它来解析正则表达式。另外，pcre-devel 是使用PCRE做二次开发时所需要的开发库，包括头文件等，这也是编译Nginx所必须使用的。使用下面的命令安装：

```shell
sudo apt install libpcre3 libpcre3-dev  
```

3. **zlib库**

zlib库用于对 HTTP 包的内容做 gzip 格式的压缩，如果我们在 nginx.conf 中配置了gzip on，并指定对于某些类型（content-type）的 HTTP 响应使用 gzip 来进行压缩以减少网络传输量，则在编译时就必须把 zlib 编译进 Nginx。zlib-devel 是二次开发所需要的库。使用下面的命令安装：

```
sudo apt install zlib1g-dev
```

4. **OpenSSL库**

如果我们需要 Nginx 支持 SSL 加密传输，需要安装 OpenSSL 库。另外，如果我们想使用MD5、SHA1等散列函数，那么也需要安装它。使用下面的命令安装：

```shell
sudo apt install openssl libssl-dev 
```

## 下载Nginx源码

从 [http://nginx.org/en/download.html](http://nginx.org/en/download.html)下载当前稳定版本的源码。

![nginx](https://cdn.mazhen.tech/images/202211111506229.png)

当前稳定版为 1.22.1：

```shell
wget https://nginx.org/download/nginx-1.22.1.tar.gz
```

## Nginx配置文件的语法高亮

为了 Nginx 的配置文件在 vim 中能语法高亮，需要经过如下配置。

解压 Nginx 源码：

```sehll
tar -zxvf nginx-1.22.1.tar.gz
```

将 Nginx 源码目录 `contrib/vim/` 下的所有内容，复制到 `$HOME/.vim` 目录：

```shell
mkdir ~/.vim
cp -r contrib/vim/*  ~/.vim/
```

现在使用 `vim` 打开 `nginx.conf`，可以看到配置文件已经可以语法高亮了。

## 编译前的配置

编译前需要使用 configure 命令进行相关参数的配置。

使用 `configure --help` 查看编译配置支持的参数：

```shell
$ ./configure --help | more
  --help                             print this message

  --prefix=PATH                      set installation prefix
  --sbin-path=PATH                   set nginx binary pathname
  --modules-path=PATH                set modules path
  --conf-path=PATH                   set nginx.conf pathname
  --error-log-path=PATH              set error log pathname
  --pid-path=PATH                    set nginx.pid pathname
  --lock-path=PATH                   set nginx.lock pathname

  ......

  --with-libatomic                   force libatomic_ops library usage
  --with-libatomic=DIR               set path to libatomic_ops library sources

  --with-openssl=DIR                 set path to OpenSSL library sources
  --with-openssl-opt=OPTIONS         set additional build options for OpenSSL

  --with-debug                       enable debug logging
```

`--with`开头的模块缺省不包括在编译结果中，如果想使用需要在编译配置时显示的指定。`--without`开头的模块则相反，如果不想包含在编译结果中需要显示设定。

例如我们可以这样进行编译前设置：

```shell
./configure --prefix=/home/mazhen/nginx  --with-http_ssl_module
```

设置了Nginx的安装目录，以及需要`http_ssl`模块。

`configure`命令执行完后，会生成中间文件，放在目录`objs`下。其中最重要的是`ngx_modules.c`文件，它决定了最终那些模块会编译进`nginx`。

## 编译和安装

* 执行编译

在nginx目录下执行`make`编译：

```shell
make
```

编译成功的`nginx`二进制文件在`objs`目录下。如果是做nginx的升级，可以直接将这个二进制文件copy到nginx的安装目录中。

* 安装

在nginx目录下执行`make install`进行安装：

```shell
make install
```

安装完成后，我们到 `--prefix` 指定的目录中查看安装结果：

```shell
$ tree -L 1 /home/mazhen/nginx
nginx/
├── conf
├── html
├── logs
└── sbin
```

## 验证安装结果

编辑 `nginx/conf/nginx.conf` 文件，设置监听端口为`8080`：

```nginx
http {
    ...

    server {
        listen       8080;
        server_name  localhost;
...
```

启动 nginx

```shell
./sbin/nginx
```

访问默认首页：

```shell
$ curl -I http://localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.22.1
Date: Fri, 11 Nov 2022 08:04:46 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Tue, 08 Nov 2022 09:54:09 GMT
Connection: keep-alive
ETag: "636a2741-267"
Accept-Ranges: bytes
```
