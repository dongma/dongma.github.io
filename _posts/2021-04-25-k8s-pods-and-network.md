---
layout: post
title: k8s核心组件及pod组件间通信原理
---

> 介绍`k8s`的核心组件如`Pod`、`Controller`、`StatefulSet`等组件以及组件间通信原理`Service`及`Ingress`服务。

###  Docker实例及Pods间的通信原理

在通信协议中“网络栈”包括有：网卡（`network interface`）、回环设备（`loopback device`）、路由表（`routing table`）和`iptables`规则。在`docker`中启动一个容器可使用宿主机的网络栈（`-net=host`），指定`-net`后默认不开启`network namespace`空间：

```shell
$ docker run –d –net=host --name nginx-host nginx
```

`nginx`服务启动后默认监听主机`80`端口，容器启动后会创建一个`docker0`的网桥。`docker`实例通过`Veth Pair`与宿主机建立连接关系，其中`Veth`的一端在容器内，另一段插在宿主机的`docker0`网桥上。

同一台宿主机上的容器实例间的网络是互通的，请求路由是通过宿主机向外转发。`ping 172.17.0.3`时匹配`0.0.0.0`的路由网关，意味着这是一条直连规则，匹配该规则的都走主机的`eth0`网卡。

在容器内`ping other-ip`时需将`other-ip`转换为`mac`地址（通`arp`地址解析获取硬件地址），容器内无法完成此操作容器通过默认路由在宿主机解析，获取请求`mac`地址 然后从容器经过`docker0`中 `Veth Pair`另外一端通过宿主机将请求转发出去。

<!-- more -->

在`docker`的默认配置下，一台宿主机上的`docker0`网桥，和其他宿主机上的`docker0`网桥，没有任何关联，它们互相之间也没办法连通。所以，连接在这些网桥上的容器，自然也没办法进行通信了。

#### 1. 容器跨主机网络（Overlay Network）

`flannel` 项目是`coreOS`公司主推的容器网络方案，事实上，`flannel`项目本身只是一个框架，真正为我们提供容器网络功能的，是 `flannel`的后端实现。有`3`种方式，基于`vxlan`、`host-gw`和`udp`进行实现。`flannel UDP`模式提供的其实是一个三层的`Overlay`网络。

`node 1`上有一个容器`container-1`，它的`IP`地址是`100.96.1.2`，对应的`docker0`网桥的地址是`100.96.1.1/24`。

`node 2`上有一个容器`container-2`，它的`IP`地址是`100.96.2.3`，对应的`docker0`网桥的地址是`100.96.2.1/24`。

```shell
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0 proto kernel scope link src 100.96.1.0
100.96.1.0/24 dev docker0 proto kernel scope link src 100.96.1.1
10.168.0.0/24 dev eth0 proto kernel scope link src 10.168.0.2
```

`node`跨主机通信引入了`flannel`组件，从`node1`请求`node2`在每个组件对请求包进行分发（`docker0`、`flannel`）,`flannel`包含子网在`node2`地址在`subnet`范围内，则对请求包进行分发最终达到`node2`上的`container`。

`flannel`项目里一个非常重要的概念子网（`subnet`），在由`flannel`管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个“子网”。在我们的例子中，`node 1` 的子网是`100.96.1.0/24`，`container-1` 的`IP`地址是`100.96.1.2`。`node 2`的子网是 `100.96.2.0/24`，`container-2`的 IP地址是`100.96.2.3`。

`TUN`设备的原理，这正是一个从用户态向内核态的流动方向（`Flannel`进程向`TUN`设备发送数据包），所以 `Linux`内核网络栈就会负责处理这个`IP`包，具体的处理方法，就是通过本机的路由表来寻找这个`IP`包的下一步流向。

课后问题：我觉得不合适，`mac`地址为硬件地址 当与请求节点直连时 可通过`mac`实现，但当目的`node`不在`subnet`时，还是要需要`rarp`地址逆解析）转换为`ip` 然后将数据包分发到目的`node`。

`VXLAN`，即 `Virtual Extensible LAN`（虚拟可扩展局域网），是 `Linux `内核本身就支持的一种网络虚似化技术。所以说，`VXLAN` 可以完全在内核态实现上述封装和解封装的工作，从而通过与前面相似的“隧道”机制，构建出覆盖网络（`Overlay Network`）。

设计思想是在现有的三层网络之上，“覆盖”一层虚拟的、由内核 `VXLAN` 模块负责维护的二层网络，使得连接在这个 `VXLAN `二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（`LAN`）里那样自由通信。

#### 2. kubernetes的网络模型与CNI网络插件
`kubernetes`使用`cni`作为`pod`的容器间通信的网桥（与`docker0`功能相同），初始化`pod`网络流程：

创建`Infra`容器调用`cni`插件初始化`infra`容器网络（插件位置：`/opt/cni/bin/flannel`），开始`dockershim`设置的一组 `CNI`环境变量（枚举值`ADD`、`DELETE`），用于表示将容器的`VethPair`插入或从`cni0`网桥移除。
与此同时，`cni bridge`插件检查`cni`网桥在宿主机上是否存在，若不存在则进行创建。接着，`cni bridge`插件在`network namespace`创建`VethPair`，将其中一端插入到宿主机的`cni0`网桥，另一端直接赋予容器实例`eth0`，`cni`插件把容器`ip`提供给`dockershim` 被`kubelet`用于添加到`pod`的`status`字段。

接下来，`cni bridge`调用`cni ipam`插件 从`ipam.subnet`子网中给容器`eth0`网卡分配`ip`地址同时设置`default route`配置，最后`cni bridge`插件为`cni`网桥设置`ip`地址。

三层网络特点：通过`ip route`得到数据传输路由，跨节点传输`ip`包时会将`route`中`geteway`的`mac`地址作为`ip`包的请求头用于数据包传输，到达目标`node`时 进行拆包，然后根据`ip`包去除`dest`地址并根据当前`node`的`route`列表，将数据包转发到相应`container`中。
优缺点：避免了额外的封包、拆包操作 性能较好，但要求集群宿主机间是二层连通的；
隧道模式：隧道模式通过`BGP`维护路由关系，其会将集群节点的`ip` 对应`gateway` 保存在当前节点的路由中，在请求发包时数据包`mac`头地址指定为路由`gateway`地址。
优缺点：需维护集群中所有`container`的连接信息，当集群中容器数量较大时`BGP`会爆炸增长，此时可切换至集群中某几个节点维护网络关系，剩余的节点从主要节点同步路由信息。

`k8s`使用`NetworkPolicy`定义`pod`的隔离机制，使用`ingress`和`egress`定义访问策略（限制可请求的`pod`及`namespace`、`port`端口），其本质上是`k8s`网络插件在宿主机上生成了`iptables`路由规则；

