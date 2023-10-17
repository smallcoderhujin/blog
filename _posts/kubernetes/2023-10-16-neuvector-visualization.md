---
title: "Neuvector源码之 网络流量拓扑"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - neuvector
  - 网络流量拓扑

---

## 背景

容器环境通信频繁，包含大量的东西向和南北向通信，通过neuvector的网络活动功能，可以全局查看集群中容器间的流量/访问host的流量/访问外网以及来自外网的流量。
网络活动功能比较丰富，包含：
- 网络抓包
- 模式切换（学习/告警/保护）
- 实时流量查看（方向/协议/端口/数据统计等）
- 容器的自动发现和展示

这里我们只深入介绍下实时流量的实现，并讨论下移植到虚拟机场景的可能性

## 架构图

![neuvector file](/blog/img/neuvector_flow.png)

kubernetes的网络是通过cni标准来实现的，实现方式多种多样。neuvector提供一种不依赖特定cni的通用的方式来获取容器流量并展示

![neuvector file](/blog/img/neuvector_flow1.png)

我们从几个维度来看neuvector的流量拓扑功能：

- 分成三个模块，分别是controller/enforce/dp
- controller负责提供api接口，汇总监控数据并行成map拓扑数据
- enforce负责监控节点容器的生命周期，汇总dp的数据，并添加相关容器信息，上报给controller
- dp是最底层负责解析数据报文，创建连接session并提供连接监控数据 


## 源码
这里先看下容器的探测
![neuvector file](/blog/img/neuvector_flow2.png)

拓扑展示资源包括：
- container：容器资源，以group为单位，一个group中可能包含多个容器，类似pod
- node：物理节点
- external：外部网络


这里通过容器资源的获取来具体介绍流程情况：
- 每个节点的enforce容器负责监听当前节点容器runtime的事件，runtime包括：docker/containerd/crio
- 监听的事件包括：创建、停止、删除、拷贝进容器、从容器拷贝出
- 这里会根据节点中proc目录下每个容器的cgroup信息来处理pod中多个容器，这里资源展示以pod单位
- 在监听到事件后更新本地内存数据，同时更新到consul数据库中
- 在controller容器中会watch consul数据库的变化，在发现有容器数据新建后更新本地的内存数据
- 在用户请求api获取容器资源列表时，直接读取内存数据并返回

然后我们再看看connection如何获取的：
![neuvector file](/blog/img/neuvector_flow3.png)

- 当容器创建或者更新时，enforce会调用dp接口下发相关的网络信息
- 当两个容器在相互通信时，所有报文都会被容器所在节点的dp模块监听，dp模块在发现一个新的session创建后，会通过socket连接通知节点上的enforce 容器
- enforce容器会解析dp发送的消息，同时根据报文mac地址识别出mac对应的源容器，目标容器一般是无法识别的，需要上送到controller中进一步识别
- 在enforce容器中会一个定时任务用来将本地的连接数据通过grpc发送到controller模块
- 在controller中reportconnection方法会更新本地的wlGraph内存数据，并将连接保存到数据库中
- 在用户请求api获取容器资源间连接列表或者指定两个资源间的连接时，直接读取内存数据并返回


最后看看dp中如何获取连接信息的：
![neuvector file](/blog/img/neuvector_flow4.png)

- 当一个新的容器创建完成后，dp会通过socket连接到容器网卡上
- dp会接受容器网卡的tx和rx报文，其中主要分析rx报文
- 首先会根据报文特征查询对应的session，并更新session中连接信息，主要是一个技术类的参数
- dp中有一个定时器，每6s将dp中所有进程的连接数据发送到enforce容器
> 注意：当前只支持ipv4连接的上报


## 虚拟机流量拓扑
进一步分析源码，发现这套机制完全可以应用到虚拟机场景，不过有几个问题需要解决：
- 虚拟机的探测：需要监控libvirt事件，检测虚拟机的生命周期
- 虚拟机网卡的探测：需要定时探测虚拟机网卡，包括新增和卸载网卡，并将网卡信息下发给dp，dp和虚拟机网卡建立socket连接
- 保护模式理论上也是可以支持的，但是需要根据虚拟机网络的具体实现在链路中嵌入dp那一套流程（参考另一篇博客）
- 同网络通信是正常支持的，对于vxlan多网络场景，cidr可能相同的情况下，部分根据ip/mac地址获取虚拟机信息的流程需要优化

虚拟机的流量拓扑已经基本实现，这里有需要可以进一步讨论

## 总结
本次没有针对源码逐步分析，从架构和流程上分析了流量拓扑的实现原理，同时分析虚拟机场景下使用这种方案的可能行。
实际发现流量探测的时效性还是挺高的，基本秒级响应，同时对于异常（告警/攻击/保护）流量可以实时告警。
个人感觉唯一不足的地方是支持的协议不太多，但常用是够的，后面可以看看如何自定义。

支持的协议：

协议|备注
---|---
cassandra|-
couchbase|-
dhcp|-
dns|-
echo|-
grpc|-
http|-
kafka|-
mongodb|-
mysql|-
ntp|-
postgresql|-
redis|-
spark|-
sqlinjection|-
ssh|-
ssl|-
tds|-
tftp|-
tns|-
zookeeper|-