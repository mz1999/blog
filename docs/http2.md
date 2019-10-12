---
marp: true
title: HTTP 2.0简介
theme: uncover
paginate: true
_paginate: false
---

# HTTP 2.0简介

马震

---

### HTTP简史
* `HTTP`(HyperText Transfer Protocol，超文本传输协议)是互联网上最普遍采用的一种应用协议
* 由欧洲核子研究委员会`CERN`的英国工程师[Tim Berners-Lee](https://en.wikipedia.org/wiki/Tim_Berners-Lee)在1991年发明
* [Tim Berners-Lee](https://en.wikipedia.org/wiki/Tim_Berners-Lee)也是WWW的发明者

---
### HTTP简史
* `HTTP/0.9`：只有一行的协议
    * 请求只有一行，包括`GET`方法和要请求的文档的路径
    * 响应是一个超文本文档，没有首部，也没有其他元数据，只有`HTML`
    * 服务器与客户端之间的连接在每次请求之后都会关闭
* `HTTP/0.9`的设计目标传递超文本文档

---
### HTTP简史

* `HTTP/0.9`演示


```
$> telnet apache.org 80

Trying 95.216.24.32...
Connected to apache.org.
Escape character is '^]'.

GET /foundation/

<!DOCTYPE html>
...
Connection closed by foreign host.
```

---
### HTTP简史
* 1996年`HTTP`工作组发布了`RFC 1945`，这就是`HTTP/1.0`
* 提供请求和响应的各种元数据
* 不局限于超文本的传输，响应可以是任何类型：`HTML`文件、图片、音频等
* 支持内容协商、内容编码、字符集、认证、缓存等
* 从**超文本**到**超媒体**传输

--- 
### HTTP简史

* `HTTP/1.0`演示

```
$> telnet apache.org 80
Trying 95.216.24.32...
Connected to apache.org.
GET /foundation/ HTTP/1.0
Accept: */*

HTTP/1.1 200 OK
Server: Apache/2.4.18 (Ubuntu)
Content-Length: 46012
Connection: close
Content-Type: text/html

<!DOCTYPE html>
...
Connection closed by foreign host.
```

---
### HTTP简史

* 1997年1月定义`HTTP/1.1`标准的`RFC 2068`发布
* 1999年6月`RFC 2616`发布，取代了`RFC 2068`
* 性能优化
  * 持久连接
    * 除非明确告知，默认使用持久连接
  * 分块编码传输
  * 请求管道，支持并行请求处理（缺乏支持）
  * 增强的缓存机制

---
### HTTP简史

* `HTTP/1.1`演示
```
>$ telnet www.baidu.com 80

Trying 14.215.177.38...
Connected to www.a.shifen.com.

GET /s?wd=http2 HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Connection: keep-alive
Host: www.baidu.com

HTTP/1.1 200 OK
Connection: Keep-Alive
Content-Type: text/html;charset=utf-8
```

---

```
Date: Sun, 06 Oct 2019 12:49:28 GMT
Server: BWS/1.1
Transfer-Encoding: chunked

ffa
<!DOCTYPE html>
...

1be7
...

0

GET /img/bd_logo1.png HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml
Accept-Language: zh-CN,zh;q=0.9,en-US;q=0.8,en;q=0.7
Connection: close
Host: www.baidu.com

HTTP/1.1 200 OK
```

---

```
Content-Length: 7877
Content-Type: image/png
Date: Sun, 06 Oct 2019 13:05:06 GMT
Etag: "1ec5-502264e2ae4c0"
Expires: Wed, 03 Oct 2029 13:05:06 GMT
Last-Modified: Wed, 03 Sep 2014 10:00:27 GMT
Server: Apache
Set-Cookie: BAIDUID=0D01C3C9C00A6019C16F79CAEB1EFE91:FG=1
Connection: close

.
.
.
.
.
.



Connection closed by foreign host.
```

---
### HTTP简史
* 2015年正式发布`HTTP/2`
  * 主要目标：改进传输性能，低延迟和高吞吐量
  * 保持原有的高层协议语义不变
* 根据[报告](https://w3techs.com/technologies/details/ce-http2/all/all)，截止2019年10月，全球已经有 **41.1%** 的网站开启了`HTTP/2`

---
### HTTP/1.1 持久连接
![tcp-connection1 w:850](./media/http2/tcp-connection1.png)

---
### HTTP/1.1 持久连接
![tcp-connection1 w:850](./media/http2/tcp-connection2.png)

---
### HTTP/1.1 持久连接
* 非持久`HTTP`连接的固定时间成本
  * 至少两次网络往返： 握手、请求和响应
* 服务处理速度越快，固定延迟的影响就越大
* 持久连接避免`TCP`连接时的三次握手，消除`TCP`的慢启动

---
### HTTP/1.1 管道
* 多次请求必须满足**先进先出**(FIFO)的顺序

![keep-alive w:600](./media/http2/http-keep-alive.png)

---
### HTTP/1.1 管道
* 尽早发送请求，不被每次响应阻塞

![keep-alive w:750](./media/http2/http-pipeline.png)

---
### HTTP/1.1 管道

* `HTTP/1.1`的局限性
  * 只能严格串行地返回响应，不允许一个连接上的多个响应交错到达
* 管道的问题
  * 并行处理请求时，服务器必须缓冲管道中的响应，占用服务器资源
  * 由于失败可能导致重复处理，非幂等的方法不能pipeline化
  * 由于中间代理的兼容性，可能会破坏管道
* 管道的应用非常有限