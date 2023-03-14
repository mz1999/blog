# Nginx反向代理配置

反向代理（reverse proxy）是指用代理服务器来接受外部的访问请求，然后将请求转发给内网的上游服务器，并将从上游服务器上得到的结果返回外部客户端。作为反向代理是 Nginx 的一种常见用法。

![reverse proxy](https://cdn.mazhen.tech/images/202211151520191.webp)

这里的负载均衡是指选择一种策略，尽量把请求平均地分布到每一台上游服务器上。下面介绍负载均衡的配置项。

## upstream

作为反向代理，一般都需要向上游服务器的集群转发请求。[upstream](https://nginx.org/en/docs/http/ngx_http_upstream_module.html#upstream) 块定义了一个上游服务器的集群，便于反向代理中的 proxy_pass使用。

```nginx
http {
    ...
    upstream backend {
        server 127.0.0.1:8080;
    }
    ...
}
```

upstream 定义了一组上游服务器，并命名为 `backend`。

## proxy_pass

[proxy_pass](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) 指令设置代理服务器的协议和地址。协议可以指定 "http "或 "https"。地址可以指定为域名或IP地址，也可以配置为 upstream 定义的上游服务器：

```nginx
http {
    server {
        listen       6888;
        server_name  localhost;

        location / {
            proxy_pass http://backend;
        }
    }
}
```

## proxy_set_header

在传递给上游服务器的请求头中，可以使用[proxy_set_header](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) 重新定义或添加字段。一般我们使用 proxy_set_header 向上游服务器传递一些必要的信息。

```nginx
location / {
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://backend;
}
```

上面的配置使用 proxy_set_header 添加了三个 HTTP header：

* Host

Host 是表明请求的主机名。默认情况下，Nginx 向上游服务器发送请求时，请求头中的 Host 字段是上游真实服务器的IP和端口号。如果我们想让传递给上游服务器的 Host 字段，包含的是用户访问反向代理时使用的域名，就需要通过 `proxy_set_header` 设置 Host 字段，值可以为 **$host** 或 **$http_host**，区别是前者只包含IP，而后者包含IP和端口号。

* X-Real-IP

经过反向代理后，上游服务器无法直接拿到客户端的 ip，也就是说，在应用中使用`request.getRemoteAddr()` 获得的是 Nginx 的地址。通过 `proxy_set_header X-Real-IP $remote_addr;`，将客户端的 ip 添加到了 HTTP header中，让应用可以使用 `request.getHeader(“X-Real-IP”)` 获取客户端的真实ip。

* X-Forwarded-For

如果配置了多层反向代理，当一个请求经过多层代理到达上游服务器时，上游服务器通过 X-Real-IP 获得的就不是客户端的真实IP了。那么这个时候就要用到 **X-Forwarded-For** ，设置 X-Forwarded-For 时是增加，而不是覆盖，从客户的真实IP为起点，穿过多层级代理 ，最终到达上游服务器，都会被记录下来。

## proxy_cache

Nginx 作为反向代理支持的所有特性和内置变量都可以在 [ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html) 的文档页面找到：

![ngx_http_proxy_module](https://cdn.mazhen.tech/images/202211160945215.png)

其中一个比较重要的特性是 proxy cache，对访问上游服务器的请求进行缓存，极大减轻了对上游服务的压力。

配置示例：

```nginx
http {
    ...
    proxy_cache_path /tmp/nginx/cache levels=1:2 keys_zone=myzone:10m inactive=1h max_size=10g use_temp_path=off;
    server {
        ...
        location / {
            ...
            proxy_cache myzone;
            proxy_cache_key $host$uri$is_args$args;
            proxy_cache_valid 200 304 302 12h;
        }
    }
}
```

配置说明：

* `proxy_cache_path` 缓存路径，要把缓存放在哪里
  * `levels=1:2`：缓存的目录结构
  * `keys_zone=myzone:10m`：定义一块用于存放缓存key的共享内存区，命名为myzone，并分配 10MB 的内存；配至10MB的zone 大约可以存放 80000个key。
  * `inactive=1d`：不活跃的缓存文件 1 小时后将被清除
  * `max_size=10g`：缓存所占磁盘空间的上限
  * `use_temp_path=off`：不另设临时目录

* `proxy_cache myzone;`：代表要使用上面定义的 myzone
* `proxy_cache_key`：用于生成缓存键，区分不同的资源。key 是决定缓存命中率的因素之一。
  * `$host`：request header中的 Host字段
  * `$uri`：请求的uri
  * `$is_args` 反映请求的 URI 是否带参数，若没有即为空值。
  * `$args`：请求中的参数

* `proxy_cache_valid`：控制缓存有效期，可以针对不同的 HTTP 状态码可以设定不同的有效期。示例针对 200，304，302 状态码的缓存有效期为12小时。

检验缓存配置的效果。

首先查看缓存路径，没有存放任何内容：

```shell
$ tree /tmp/nginx/cache/
/tmp/nginx/cache/

0 directories, 0 files
```

然后访问Nginx反向代理服务器：

```shell
❯ curl -v http://172.21.32.84:6888/

...
```

再次查看缓存路径：

```shell
$ tree /tmp/nginx/cache/
/tmp/nginx/cache/
└── 6
    └── ed
        └── 5e9596b7783c532f541535dd1a60eed6

2 directories, 1 file
```

经过请求后，缓存路径中已经有内容，并且目录结构是我们配置的 `level=1:2`。
