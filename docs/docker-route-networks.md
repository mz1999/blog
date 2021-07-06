# Docker跨主机通信路由模式动手实验

容器的跨主机通信主要有两种方式：封包模式和路由模式。[上一篇文章](./docker-overlay-networks.md)演示了使用`VXLAN`协议的封包模式，这篇将介绍另一种方式，利用三层网络的路由转发实现容器的跨主机通信。

## 路由模式概述

宿主机将它负责的容器IP网段，以某种方式告诉其他节点，然后每个节点根据收到的`<宿主机-容器IP网段>`映射关系，配置本机路由表。

这样对于容器间跨节点的IP包，就可以根据本机路由表获得到达目的容器的网关地址，即目的容器所在的宿主机地址。接着在把IP包封装成二层网络数据帧时，将目的MAC地址设置为网关的MAC地址，IP包就可以通过二层网络送达目的容器所在的宿主机。

至于用什么方式将`<宿主机-容器IP网段>`映射关系发布出去，不同的项目采用了不同的实现方案。

[Flannel](https://github.com/coreos/flannel)是将这些信息集中存储在[etcd](https://github.com/etcd-io/etcd)中，每个节点从[etcd](https://github.com/etcd-io/etcd)自动获取数据，更新宿主机路由表。

[Calico](https://github.com/projectcalico/calico/)则使用[BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol)（Border Gateway Protocol）协议交换共享路由信息，每个宿主机都是运行在它之上的容器的`边界网关`。

如果宿主机之间跨了网段怎么办？宿主机之间的二层网络不通，虽然知道目的容器所在的宿主机，但没办法将目的MAC地址设置为那台宿主机的MAC地址。

`Calico`有两种解决方案：

* IPIP 模式，在跨网段的宿主机之间建立“隧道”
* 让宿主机之间的路由器“学习”到容器路由规则，每个路由器都知道某个容器IP网段是哪个宿主机负责的，容器间的IP包就能正常路由了。

## 动手实验

![2018-11-15 23.42.06](media/docker/2018-11-15%2023.42.06.png)


路由模式的实验比较简单，关键在于宿主机上路由规则的配置。为了简化实验，这些路由规则都是我们手工配置，而且两个节点之间二层网络互通，没有跨网段。

参照[Docker跨主机Overlay网络动手实验](./docker-overlay-networks.md)，创建“容器”，`veth pairs`，`bridge`，设置IP，激活虚拟设备。

然后在`node-1`上增加路由规则：

```
sudo ip route add 172.18.20.0/24 via 192.168.31.192
```

在`node-2`上增加路由规则：

```
sudo ip route add 172.18.10.0/24 via 192.168.31.183
```

当`docker1`访问`docker2`时，IP包会从`veth`到达`br0`，然后根据`node-1`上刚设置的路由规则，访问`172.18.20.0/24`网段的网关地址为`node-2`，这样，IP包就能路由到`node-2`了。

同时，`node-2`的路由表中包含这样的一条规则：

```
$ ip route

...
172.18.20.0/24 dev br0 proto kernel scope link src 172.18.20.1
```

到达`node-2`的IP包，会根据这条规则路由到网桥`br0`，最终到达`docker-2`。反过来从`docker2`访问`docker1`的过程也是类似。


## 总结

两种容器跨主机的通信方案我们都实验了一下，现在做个简单总结对比：

* **封包模式**对基础设施要求低，三层网络通就可以了。但封包、解包带来的性能损耗较大。
* **路由模式**性能好，但要求二层网络连通，或者在跨网段的情况下，要求路由器能配合“学习”路由规则。


至此，容器网络的三篇系列完成：

* [Docker单机网络模型动手实验](./docker-network-bridge.md)
* [Docker跨主机Overlay网络动手实验](./docker-overlay-networks.md)
* Docker跨主机通信路由模式动手实验（本篇）

