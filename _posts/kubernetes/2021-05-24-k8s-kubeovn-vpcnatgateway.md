---
title: "KubeOVN - VPCNatGateway"
subtitle: "vpc nat gateway"
layout: post
author: "hujin"
header-style: text
tags:
  - k8s
  - cni
  - kube-ovn
  - vpc-nat-gateway
---


## 功能
用来给k8s中pod/service/ingress等网络资源访问外部网络

### 支持功能

- 指定外部网络出口物理网卡
- 指定多子网以及子网的网关
- 浮动IP功能
- 端口转发功能
- 静态路由规则
- SNAT功能

## 原理
![kube-ovn vpc-nat-gateway](/blog/img/vpc-nat-gateway.png)

- 创建一个nat gateway deployment，副本数是1
- 通过configmap获取pod对应的镜像文件路径，创建nat gateway pod；重建策略是recreate

pod中设置annotations：

- ovn.kubernetes.io/vpc_nat_gw： nat gateway的名称
- k8s.v1.cni.cncf.io/networks： 指定pod对应namespace的外部网络
- ovn.kubernetes.io/logical_switch：为nat gateway添加的子网列表
- ovn.kubernetes.io/ip_address：为nat gateway设置的网关IP列表
- 等待nat gateway pod变成Running后，触发nat gateway初始化，执行命令： 
    bash /kube-ovn/nat-gateway.sh init
    
init执行的命令：

    ip link set net1 up
    ip rule add iif net1 table $ROUTE_TABLE
    ip rule add iif eth0 table $ROUTE_TABLE
    
    # add static chain
    iptables -t nat -N DNAT_FILTER
    iptables -t nat -N SNAT_FILTER
    iptables -t nat -N EXCLUSIVE_DNAT # floatingIp DNAT
    iptables -t nat -N EXCLUSIVE_SNAT # floatingIp SNAT
    iptables -t nat -N SHARED_DNAT
    iptables -t nat -N SHARED_SNAT
    
    iptables -t nat -A PREROUTING -j DNAT_FILTER
    iptables -t nat -A DNAT_FILTER -j EXCLUSIVE_DNAT
    iptables -t nat -A DNAT_FILTER -j SHARED_DNAT
    
    iptables -t nat -A POSTROUTING -j SNAT_FILTER
    iptables -t nat -A SNAT_FILTER -j EXCLUSIVE_SNAT
    iptables -t nat -A SNAT_FILTER -j SHARED_SNAT
    
vpc nat gateway相关线程说明:

- addVpcWorker:监听vpc创建事件，并在ovn中创建logical router
- AddOrUpdateVpcNatGwWorker：监听vpc nat gateway创建或更新事件，创建或更新对应的nat gateway deployment
- initVpcNatGwWorker：初始化nat gateway deployment中对应的pod容器，执行命令见上文
- delVpcNatGwWorker：删除nat gateway deployment
- updateVpcEipWorker：找到nat gateway deployment对应的pod，获取annotations中eip相关字段，判断需要删除和新增的eip，并更新到annotations中
- updateVpcFloatingIpWorker：在nat gateway 中更新浮动IP的映射关系
- updateVpcDnatWorker：在nat gateway 中更新端口转发的映射关系
- updateVpcSnatWorker：在nat gateway 中更新子网和eip的snat的映射关系
- updateVpcSubnetWorker：在nat gateway 中新增、删除子网网卡，创建或者删除对应子网的路由信息

## 使用方式
开启VPC Nat Gatway功能

    kind: ConfigMap
    apiVersion: v1
    metadata:
      name: ovn-vpc-nat-gw-config
      namespace: kube-system
    data:
      image: 'kubeovn/vpc-nat-gateway:v1.7.0'  # Docker image for vpc nat gateway
      enable-vpc-nat-gw: true                  # 'true' for enable, 'false' for disable 
      nic: eth1                                # The nic that connect to underlay network, use as the 'master' for macvlan
    
创建VPC Nat Gateway

    kind: VpcNatGateway
    apiVersion: kubeovn.io/v1
      name: ngw
    spec:
      vpc: test-vpc-1                  # Specifies which VPC the gateway belongs to 
      subnet: sn                       # Subnet in VPC  
      lanIp: 10.0.1.254                # Internal IP for nat gateway pod, IP should be within the range of the subnet 
      eips:                            # Underlay IPs assigned to the gateway
        - eipCIDR: 192.168.0.111/24
           gateway: 192.168.0.254
        - eipCIDR: 192.168.0.112/24
           gateway: 192.168.0.254
      floatingIpRules: 
        - eip: 192.168.0.111
          internalIp: 10.0.1.5
      dnatRules:
        - eip: 192.168.0.112
          externalPort: 8888
          protocol: tcp
          internalIp: 10.0.1.10
          internalPort: 80
      snatRules:
        - eip: 192.168.0.112
           internalCIDR: 10.0.1.0/24
           
给VPC添加静态路由

    kind: Vpc
    apiVersion: kubeovn.io/v1
    metadata:
      name: test-vpc-1
    spec:
      staticRoutes:
        - cidr: 0.0.0.0/0
          nextHopIP: 10.0.1.254     # Should be the same as the 'lanIp' for vpc gateway
          policy: policyDst