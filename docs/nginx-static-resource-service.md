# Nginx静态资源服务的配置

## 配置文件语法

Nginx的配置文件是一个文本文件，由指令和指令块构成。

### 指令

**指令**以分号 `;` 结尾，指令和参数间以空格分割。

**指令块**作为容器，将相关的指令组合在一起，用大括号 `{}` 将它们包围起来。

```nginx
http {
    include       mime.types;
    default_type  application/octet-stream;

    server {
        listen       8080;
        server_name  localhost;
        location / {
            root   html;
            index  index.html index.htm;
        }
  }
}
```

上面配置中的http、server、location等都是指令块。指令块配置项之后是否如参数（例如 location /），取决于解析这个块配置项的模块。

指令块配置项是可以嵌套的。内层块会继承父级块包含的指令的设置。有些指令可以出现在多层指令块内，你可以通过在内层指令块包含该指令，来覆盖从父级继承的设置。

### Context

一些 top-level 指令被称为 **context**，将适用于不同流量类型的指令组合在一起。

* **events** – 通用的连接处理
* **http** – HTTP流量
* **mail** – Mail 流量
* **stream** – TCP 和 UDP 流量

放在这些 context 之外的指令是在 main context中。

在每个流量处理 context 中，可以包括一个或多个 **server** 块，用来定义控制请求处理的虚拟服务器。

对于HTTP流量，每个 server 指令块是对特定域名或IP地址访问的控制。通过一个活多个 **location** 定义如何处理特定的URI。

对于 Mail 和 TCP/UDP 流量，server 指令块是对特定 TCP 端口流量的控制。

## 静态资源服务

将个人网站的静态资源 clone 到 nginx 根目录：

```shell
git clone https://github.com/mz1999/mazhen.git
```

在 `conf/nginx.conf` 文件中配置监听端口和 `location`：

```nginx
http {
    server {
        listen       8080;
        server_name  localhost;
        
        #charset koi8-r;
        #access_log  logs/host.access.log  main;
        
        location / {
            alias   mazhen/;
            #index  index.html index.htm;
        }

}
```

location 的语法格式为：

```
location [ = | ~ | ~* | ^~ ] uri { ... }
```

`location` 会尝试根据用户请求中的 URI 来匹配上面的 `uri` 表达式，如果可以匹配，就选择这个 `location` 块中的配置来处理用户请求。

`location` 指定文件路径有两种方式：**root**和**alias**。

`root` 与`alias` 会以不同的方式将请求映射到服务器的文件上，它们的主要区别在于如何解释 `location` 后面的 `uri` 。

* **root**的处理结果是，root＋location uri。
* **alias**的处理结果是，使用 `alias` 替换 `location uri`。 `alias` 作为一个目录别名的定义。

例如：

```nginx
location /i/ {
    root /data/w3;
}
```

如果一个请求的 URI 是 `/i/top.gif` ，Nginx 将会返回服务器上的 `/data/w3/i/top.gif` 文件。

```nginx
location /i/ {
    alias /data/w3/images/;
}
```

如果一个请求的 URI 是 `/i/top.gif`，Nginx 将会返回服务器上的 `/data/w3/images/top.gif`文件。alias 会把 `location` 后面配置的 `uri` 替换为 `alias` 定义的目录。

最后要注意，使用 `alias` 时，目录名后面一定要加 `/`。

## 开启gzip

Nginx 的 `ngx_http_gzip_module` 模块是一个过滤器，它使用 "gzip "方法压缩响应。可以在 http context 下配置 gzip：

```nginx
http {
    ...
    gzip  on;
    gzip_min_length 1000;
    gzip_comp_level 2;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
  
    server {
        ...
    }
}
```

* gzip_min_length：设置允许压缩的页面最小字节数
* gzip_comp_level： 设置 gzip 压缩比，1 压缩比最小处理速度最快，9 压缩比最大但处理最慢
* gzip_types：匹配MIME类型进行压缩。

更多的配置项，可以参考[官方文档](https://nginx.org/en/docs/http/ngx_http_gzip_module.html)。

## autoindex

Nginx 的 `ngx_http_autoindex_module` 模块处理以斜线字符 `/` 结尾的请求，并产生一个目录列表。通常情况下，当 `ngx_http_index_module` 模块找不到index文件时，请求会被传递给 `ngx_http_autoindex_module` 模块。

autoindex 的配置很简单：

```nginx
location / {
    alias   mazhen/;
    autoindex on;
}
```

注意，只有 index 模块找不到index文件时，请求才会被 autoindex 模块处理。我们可以把 `mazhen` 目录下的 index 文件删掉，或者为 index 指令配置一个不存在的文件。

## limit_rate

由于带宽的限制，我们有时候需要限制某些资源向客户端传输响应的速率，例如可以对大文件限速，避免传输大文件占用过多带宽，从而影响其他更重要的小文件（css，js）的传输。我们可以使用 `set` 指令配合内置变量 `$limit_rate` 实现这个功能：

```nginx
location / {
    ...
    set $limit_rate 1k;
}
```

上面的指令限制了Nginx向客户端发送响应的速率为 1k/秒。

`$limit_rate`是Nginx的内置变量，Nginx的文档详细列出了每个模块的内置变量。以 [ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html) 为例，在 [Nginx文档首页](https://nginx.org/en/docs/)的 Modules reference 部分，点击进入 [ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html) ：

![http_core_module](https://cdn.mazhen.tech/images/202211151058224.png)

在  [ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html) 文档目录的最下方，点击 [Embedded Variables](https://nginx.org/en/docs/http/ngx_http_core_module.html#variables) ，会跳转到 [ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html) 内置变量列表：

![Embedded Variables](https://cdn.mazhen.tech/images/202211151116755.png)

这里有 http module 所有内置变量的说明，包括我们刚才使用 [$limit_rate](https://nginx.org/en/docs/http/ngx_http_core_module.html#var_limit_rate)。

## access log

Nginx 的 access log 功能由 [ngx_http_log_module](https://nginx.org/en/docs/http/ngx_http_log_module.html) 模块提供。ngx_http_log_module 提供了两个指令：

* **log_format** 指定日志格式
* **access_log** 设置日志写入的路径

举例说明：

```nginx
http {
    ...
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    server {
        ...
        access_log  logs/mazhen.access.log  main;
    }
}
```

log_format 使用内置变量定义日志格式，示例中的 log_format 可以使用 http module 定义的内置变量。log_format 还指定了这个日志格式的名称为 main，这样让我们定义多种格式的日志，为不同的 server 配置特定的日志格式。

access_log 设置了日志路径为 `logs/mazhen.access.log`，并指定了日志格式为 main。示例中的 access_log 定义在 server 下，那所有发往这个 server 的请求日志都使用 main 格式，被记录在  `logs/mazhen.access.log`文件中。
