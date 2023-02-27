---
title: "Neuvector源码分析之 dp引流的实现"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - neuvector
  - deep packet inspection
  - dpi
  - tc

---


## 背景
在neuvector中通过监听runtime事件来动态维护节点上容器基础信息，在界面支持策略配置、模式管理都对容器流量产生影响。用户从学习模式调整成保护模式时可以对容器流量进行阻断操作，那么agent
中是如何实现阻断的？
这里我们将核心流程抽离出来，通过demo的方式来介绍具体的实现细节

## 原理
### 默认形态

![neuvector_pcap](/blog/img/neuvector_dp1.png)

开始之前先看下默认情况下容器的网络形态（以calico为例）:
- 容器网卡使用veth设备，一端在容器内部，一端在host上
- 在host上通过路由规则的方式将流量引到host上的veth设备，最终到达容器内部

### 引流形态

![neuvector_pcap](/blog/img/neuvector_dp2.png)

引流形态下流量会优先经过agent容器，根据策略规则过滤后转发到具体的容器内部。本次我们需要从host上直接ping container-ns中的网卡ip 10.10.10.88,

首先创建两个network namespace
- ns1：引流实现位置
- ns2：模拟普通容器的network namespace

命令
 
    ip netns add agent-ns
    ip netns add container-ns

在host上创建三组veth

- vex：流量入口
- vin：跨agent-ns和container-ns，在agent中将流量转发到这个设备就可以到达容器内部
- vbr：监控设备，也是dp工作的设备

命令：

    # 创建veth
    ip link add vex type veth peer name vex-peer
    ip link add vin type veth peer name vin-peer
    ip link add vbr type veth peer name vth
     
    # 将vex放入agent-ns
    ip link set vex netns agent-ns
    ip netns exec agent-ns ip link set vex up
     
    # 将vin放入agent-ns
    ip link set vin netns agent-ns
    ip netns exec agent-ns ip link set vin up
     
    # 将vin-peer放入container-ns
    ip link set vin-peer netns container-ns
     
    # 在container-ns修改网卡名称为eth0
    ip netns exec container-ns ip link set vin-peer name eth0
    ip addr add 10.10.10.88/24 dev eth0
    ip netns exec container-ns ip link set eth0 up
     
    # 将vbr和vth都放入agent-ns
    ip link set vbr netns agent-ns
    ip link set vth netns agent-ns
    ip netns exec agent-ns ip link set vbr up
    ip netns exec agent-ns ip link set vth up

    # 配置host上的vex-peer
    ip addr add 10.10.10.66/24 dev vex-peer
    ip link set vex-peer up

tc引流

    # 创建qdisc
    tc qdisc add dev vin ingress
    tc qdisc add dev vex ingress
    tc qdisc add dev vbr ingress

    # 创建filter
    tc filter add dev vex pref 10001 parent ffff: protocol ip u32 match u8 0 1 at -14 match u16 0x1a27 0xffff at -14 match u32 0xf3873a27 0xffffffff at -12 action pedit munge offset -14 u16 set 0x4e65 munge offset -12 u32 set 0x755600b4 pipe action mirred egress mirror dev vbr
     
    tc filter add dev vex pref 10002 parent ffff: protocol all u32 match u8 0 0 action mirred egress mirror dev vin
     
    tc filter add dev vbr pref 5 parent ffff: protocol all u32 match u16 0x4e65 0xffff at -14 match u32 0x755600b4 0xffffffff at -12 action pedit munge offset -14 u16 set 0x1a27 munge offset -12 u32 set 0xf3873a27 pipe action mirred egress mirror dev vin
     
    tc filter add dev vin pref 10002 parent ffff: protocol all u32 match u8 0 0 action mirred egress mirror dev vex
     
    tc filter add dev vin pref 10001 parent ffff: protocol ip u32 match u8 0 1 at -14 match u32 0x1a27f387 0xffffffff at -8 match u16 0x3a27 0xffff at -4 action pedit munge offset -8 u32 set 0x4e657556 munge offset -4 u16 set 0x00b4 pipe action mirred egress mirror dev vbr
     
    tc filter add dev vbr pref 172 parent ffff: protocol all u32 match u32 0x4e657556 0xffffffff at -8 match u16 0x00b4 0xffff at -4 action pedit munge offset -8 u32 set 0x1a27f387 munge offset -4 u16 set 0x3a27 pipe action mirred egress mirror dev vex

此时我们从host上ping 10.10.10.88，同时在agent-ns的vex和vbr抓包会看到数据包

- 这里从抓包结果会看到目标mac变了，这个是通过tc完成的
- 但是在vin上是抓不到包的（可以抓到广播包），因为dp没有开始工作

抓包结果：

    sh-4.2# tcpdump -enpli vex
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on vex, link-type EN10MB (Ethernet), capture size 262144 bytes
    17:31:40.761531 06:82:9c:02:54:91 > 1a:27:f3:87:3a:27, ethertype IPv4 (0x0800), length 98: 10.10.10.66 > 10.10.10.88: ICMP echo request, id 32052, seq 214, length 64
    17:31:41.761533 06:82:9c:02:54:91 > 1a:27:f3:87:3a:27, ethertype IPv4 (0x0800), length 98: 10.10.10.66 > 10.10.10.88: ICMP echo request, id 32052, seq 215, length 64
    17:31:42.761516 06:82:9c:02:54:91 > 1a:27:f3:87:3a:27, ethertype IPv4 (0x0800), length 98: 10.10.10.66 > 10.10.10.88: ICMP echo request, id 32052, seq 216, length 64
    ^C
    3 packets captured
    3 packets received by filter
    0 packets dropped by kernel
    sh-4.2# tcpdump -enpli vbr
    tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
    listening on vbr, link-type EN10MB (Ethernet), capture size 262144 bytes
    17:31:45.761528 06:82:9c:02:54:91 > 4e:65:75:56:00:b4, ethertype IPv4 (0x0800), length 98: 10.10.10.66 > 10.10.10.88: ICMP echo request, id 32052, seq 219, length 64
    17:31:46.761515 06:82:9c:02:54:91 > 4e:65:75:56:00:b4, ethertype IPv4 (0x0800), length 98: 10.10.10.66 > 10.10.10.88: ICMP echo request, id 32052, seq 220, length 64
    17:31:47.761531 06:82:9c:02:54:91 > 4e:65:75:56:00:b4, ethertype IPv4 (0x0800), length 98: 10.10.10.66 > 10.10.10.88: ICMP echo request, id 32052, seq 221, length 64

运行dp

    ip netns exec agent-ns dp -n 1

> 这里需要重新编译dp，对dp做一些改造，保证可以在host运行，主要是mmap那部分逻辑


调用dp接口

分别调用dp接口：
- ctrl_add_srvc_port： 配置引流设备名称等参数
- ctrl_cfg_internal_net：配置当前节点设备ip和类型
- ctrl_add_mac：在引流设备中添加需要引流的mac
- ctrl_cfg_mac：配置mac参数

调用脚本

    import socket
    import json
     
    DP_SERVER_PATH = '/tmp/dp_listen.sock'
    SOCK = None

    def connect_dp():
        global SOCK
        if SOCK:
            return SOCK
        socket_family = socket.AF_UNIX
        socket_type = socket.SOCK_DGRAM
     
        sock = socket.socket(socket_family, socket_type)
        sock.connect(DP_SERVER_PATH)
        SOCK = sock
        print("dp connected")
        return sock
     
     
    def send_msg(msg):
        sock = connect_dp()
        sock.sendall(json.dumps(msg).encode())
     
        # data = sock.recv(1024)
        # print("receive data:", data)
        print('send msg: %s success' % msg)
     
     
    def add_srvc_port():
        data = {
            "iface": "vth",
            "jumboframe": False
        }
        send_msg({'ctrl_add_srvc_port': data})
     
     
    def add_internal_net():
        data = {
            "flag": 3,
            "subnet_addr": [
                {"ip": "172.17.0.0", "mask": "255.255.0.0"},
                {"ip": "179.20.23.0", "mask": "255.255.255.0"},
                {"ip": "10.10.10.0", "mask": "255.255.255.0"},
                {"ip": "172.20.166.0", "mask": "255.255.255.0"},
                {"ip": "10.10.10.66", "mask": "255.255.255.0", "iptype": "devip"},
                {"ip": "10.10.10.88", "mask": "255.255.255.0", "iptype": "devip"},
     
            ]
        }
        send_msg({'ctrl_cfg_internal_net': data})
     
     
    def add_mac():
        data = {
            "iface": "vth",
            "mac": "1a:27:f3:87:3a:27",
            "ucmac": "4e:65:75:56:00:b4",
            "bcmac": "ff:ff:ff:00:00:b4",
            "oldmac": "",
            "pmac": "",
            "pips": None,
        }
        send_msg({'ctrl_add_mac': data})
     
     
    def cfg_mac():
        data = {
            "tap": False,
            "macs": ["1a:27:f3:87:3a:27"],
        }
        send_msg({'ctrl_cfg_mac': data})
     
    add_internal_net()
    add_srvc_port()
    add_mac()
    cfg_mac()

## 总结

- 可以看到neuvector的引流方式和容器使用的cni无关
- 引流的准备操作实际是agent执行的，dp只是对经过vth的数据包根据策略规则进行过滤，并转发
- 后期可以探索这种使用方式是否可以应用到虚拟机状态下

这里只是调用dp接口，dp内部实现将在后面详细分析