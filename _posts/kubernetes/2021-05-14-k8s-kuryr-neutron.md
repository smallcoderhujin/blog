---
title: "Kuryr CNI"
subtitle: "cni"
layout: post
author: "hujin"
header-style: text
tags:
  - k8s
  - cni
  - neutron
  - kuryr
---


## 背景

当k8s部署在openstack节点或者虚拟机中时，可以使用openstack提供的网络功能实现k8s cni，这里主要对接的是neutron和lbaas模块，社区提供了kuryr的实现方案。
Kuryr 是 OpenStack Neutron 的子项目，其主要目标是透过该项目来集成 OpenStack 与 Kubernetes 的网络。该项目在 Kubernetes 中实作了原生 Neutron-based 的网络，
因此使用 Kuryr-Kubernetes 可以让 OpenStack VM 与 Kubernetes Pods 能够选择在同一个子网络上运作，并且能够使用 Neutron L3 与 Security Group 来对网络进行路由，以及阻挡特定来源 Port，并且也提供基于 Neutron LBaaS 的 Service 集成。

![kuryr_kubernetes_arch](/blog/img/kuryr_k8s_arch.png)
![kuryr_kubernetes_arch](/blog/img/kuryr_k8s_pipline.png)

kuryr kubernetes支持两种部署模式：

- nested：嵌套部署，通过虚机网卡子接口的方式实现，依赖社区trunk port功能
- none-nested：非嵌套部署，pod网卡类似虚机网卡直接添加到OVS网桥中，通过ovn-agent管理流表

控制节点

通过kuryr controller来watch kubernetes资源创建、更新和删除

- pod
- service
- endpoints
- namespace
- ingress(待定)
- networkpolicy

在watch到资源创建后，在neutron中创建对应的网络资源
Kubernetes|Neutron |说明
-|-|-
Pod|	VM	|
Service|	LB	|
Ingress|	L7Router/LB|	
Endpoints|	LB|
NetworkPolicy|	SecurityGroup|
Namespaces|	Network|


计算节点

计算节点中有两个服务：

- kuryr cni：kuryr cni当前有python和golang两种实现，golang实现是标准的cni方式，社区慢慢往这个方向切换
- kuryr daemon服务：为cni提供api服务用来配置、更新和删除容器网卡

当kubelet收到pod创建事件时，kubelet会调用kuryr cni的方法cmdAdd，这里kuryrcni会调用kuryr daemon的addNetwork接口

kuryr daemon有三个线程：

- watcher：watch k8s中当前节点上的pod操作
- server：提供api供kuryr cni调用；对watch到的pod执行addNetwork：创建并配置pod网卡，delNetwork:删除pod网卡
- health check： 健康检查

PS: kuryr controller在watch 资源创建后，会在neutron中创建对应资源，同时更新k8s中资源，将neutron中的资源信息设置到annotations中


## 安装三节点OpenStack

准备
    
    cat  /etc/yum.repos.d/qemu-kvm-rhev.repo
    [qemu-kvm-rhev]
    name=oVirt rebuilds of qemu-kvm-rhev
    baseurl=http://resources.ovirt.org/pub/ovirt-3.5/rpm/el7Server/
    mirrorlist=http://resources.ovirt.org/pub/yum-repo/mirrorlist-ovirt-3.5-el7Server
    enabled=1
    skip_if_unavailable=1
    gpgcheck=0
    
    yum install -y centos-release-openstack-train
    yum update -y
    yum install -y openstack-packstack
    
生成部署配置文件并修改

    packstack --gen-answer-file=/root/answer.txt
    CONFIG_CONTROLLER_HOST=192.168.1.30
    CONFIG_COMPUTE_HOSTS=192.168.1.31,192.168.1.32
    CONFIG_NETWORK_HOSTS=192.168.1.30

开始部署

    packstack --answer-file=./answer.conf

ps: 部署时发现nova报qmeu版本需要2.8以上，这里参考文档https://www.huaweicloud.com/articles/023bb022255569c81600e6e372fa06c0.html安装2.8版本的qemu


## 部署Kubernetes
TODO: kubespray

修改apiserver配置，开放8080端口

    vim /etc/kubernetes/manifests/kube-apiserver.yaml
    - --insecure-bind-address=0.0.0.0
    - --insecure-port=8080


## 部署kuryr

### 控制节点

获取代码

    git clone https://opendev.org/openstack/kuryr-kubernetes.git
    cd kuryr-kubernetes
    git checkout remotes/origin/stable/train
    cd ..
    pip install -e kuryr-kubernetes

生成配置文件

    cd kuryr-kubernetes
    ./tools/generate_config_file_samples.sh
    mkdir -p /etc/kuryr/
    cp etc/kuryr.conf.sample /etc/kuryr/kuryr.conf

创建kuryr租户和用户

    openstack project create --domain Default --description "kuryr Project" kuryr
    openstack user create --domain Default --password kuryr kuryr --project kuryr
    openstack role add --project kuryr --user kuryr admin

创建k8s pod、service网络

    neutron net-create pod-net
    neutron subnet-create pod-net 10.244.0.0/16
    
    neutron net-create service-net
    neutron subnet-create service-net 10.96.0.0/16
    
    neutron security-group-create kuryr-sg
    neutron security-group-rule-create kuryr-sg --direction ingress
    neutron security-group-rule-create kuryr-sg --direction egress

修改kuryr controller配置

    vim /etc/kuryr/kuryr.conf
    [DEFAULT]use_stderr = true
    bindir = /usr/local/libexec/kuryr
    
    [kubernetes]
    api_root = http://172.16.41.130:8080
    enabled_handlers = vif,kuryrport  # lb, lbaasspec
    
    [neutron]
    auth_url = http://172.16.41.130:5000/v3
    username = kuryr
    user_domain_name = Default
    password = kuryr
    project_name = kuryr
    project_domain_name = Default
    auth_type = password
    
    [neutron_defaults]
    ovs_bridge = br-int
    pod_security_groups = {id_of_secuirity_group_for_pods}
    pod_subnet = {id_of_subnet_for_pods}
    project = {id_of_project}
    service_subnet = {id_of_subnet_for_k8s_services}
    
启动kuryr controller服务（需要做成systemd服务）

    kuryr-k8s-controller --config-file /etc/kuryr/kuryr.conf


### 计算节点

获取代码

    git clone https://opendev.org/openstack/kuryr-kubernetes.git
    cd kuryr-kubernetes
    git checkout remotes/origin/stable/train
    cd ..
    pip install -e kuryr-kubernetes
    pip install 'oslo.privsep>=1.20.0' 'os-vif>=1.5.0'

创建配置文件

    vim /etc/kuryr/kuryr.conf
    [DEFAULT]
    use_stderr = true
    bindir = /usr/local/libexec/kuryr
    
    [kubernetes]
    api_root = http://172.16.41.130:8080

创建软链接kuryr-cni

    mkdir -p /opt/cni/bin
    ln -s $(which kuryr-cni) /opt/cni/bin/

创建cni配置文件
    
    rm -rf /etc/cni/net.d/
    mkdir -p /etc/cni/net.d/
    vim /etc/cni/net.d/10-kuryr.conf
    {
        "cniVersion": "0.3.1",
        "name": "kuryr",
        "type": "kuryr-cni",
        "kuryr_conf": "/etc/kuryr/kuryr.conf",
        "debug": true
    }
    
重启kubelet
    
    systemctl daemon-reload && systemctl restart kubelet.service
    
启动kuryr daemon服务

    kuryr-daemon --config-file /etc/kuryr/kuryr.conf -d


## 自测

创建虚机

    [root@controller01 ~]# glance image-list
    +--------------------------------------+--------+
    | ID                                   | Name   |
    +--------------------------------------+--------+
    | 5d87d4cf-384f-4644-9e81-3aafaec567f9 | cirros |
    | e9e97933-c2e7-40a7-bd39-9520be75f937 | cirros |
    +--------------------------------------+--------+
    [root@controller01 ~]# nova flavor-list
    +----+-----------+------------+------+-----------+------+-------+-------------+-----------+-------------+
    | ID | Name      | Memory_MiB | Disk | Ephemeral | Swap | VCPUs | RXTX_Factor | Is_Public | Description |
    +----+-----------+------------+------+-----------+------+-------+-------------+-----------+-------------+
    | 1  | m1.tiny   | 512        | 1    | 0         | 0    | 1     | 1.0         | True      | -           |
    | 2  | m1.small  | 2048       | 20   | 0         | 0    | 1     | 1.0         | True      | -           |
    | 3  | m1.medium | 4096       | 40   | 0         | 0    | 2     | 1.0         | True      | -           |
    | 4  | m1.large  | 8192       | 80   | 0         | 0    | 4     | 1.0         | True      | -           |
    | 5  | m1.xlarge | 16384      | 160  | 0         | 0    | 8     | 1.0         | True      | -           |
    +----+-----------+------------+------+-----------+------+-------+-------------+-----------+-------------+
    [root@controller01 ~]# neutron net-list
    neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
    +--------------------------------------+-------------+----------------------------------+----------------------------------------------------+
    | id                                   | name        | tenant_id                        | subnets                                            |
    +--------------------------------------+-------------+----------------------------------+----------------------------------------------------+
    | 032a6394-4efe-4ef5-949e-e9717bcaeed6 | public      | 298316f4f2574e898ba50b89cdae58c3 | 8255a4e7-4902-4bdf-9feb-5b552371c924 172.24.4.0/24 |
    | acd5d8f3-8b1b-495a-98f2-78a8bb23290b | service-net | 497fd0c4ce624c18a8007c4f72f021f6 | a9f5361a-9e71-457c-bd3d-6e52b638f52c 10.96.0.0/16  |
    | b3102426-e8d5-4a9e-b18b-6cdd28fd5af4 | pod-net     | 497fd0c4ce624c18a8007c4f72f021f6 | eddcc6a4-0c65-413b-9aaf-31548b31e10e 10.244.0.0/16 |
    | b5962808-53ee-4ab4-a19b-f0396535302f | private     | 1fbe0c13d018486d8c53a112621b9fc1 | 39e76bea-fc39-432c-bb28-3b1e40a350f9 10.0.0.0/24   |
    +--------------------------------------+-------------+----------------------------------+----------------------------------------------------+
    [root@controller01 ~]# nova boot  --image e9e97933-c2e7-40a7-bd39-9520be75f937 --nic net-id=b3102426-e8d5-4a9e-b18b-6cdd28fd5af4 hujin1  --flavor 1
    [root@controller01 ~]# nova list
    +--------------------------------------+--------+--------+------------+-------------+---------------------+
    | ID                                   | Name   | Status | Task State | Power State | Networks            |
    +--------------------------------------+--------+--------+------------+-------------+---------------------+
    | 88e871ff-05c8-45c4-9c09-3cbc6e61eb61 | hujin1 | ACTIVE | -          | Running     | pod-net=10.244.2.62 |
    +--------------------------------------+--------+--------+------------+-------------+---------------------+

创建pod

    [root@controller01 ~]# cat busybox.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: busybox
      namespace: default
    spec:
      containers:
      - image: busybox:latest
        command:
          - sleep
          - "3600"
        imagePullPolicy: IfNotPresent
        name: busybox
      restartPolicy: Always
    [root@controller01 ~]# kubectl get pods -owide
    NAME      READY   STATUS    RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
    busybox   1/1     Running   19         19h   10.244.2.195   compute02   <none>           <none>

pod和虚机通信
    
    [root@controller01 ~]# nova list
    +--------------------------------------+--------+--------+------------+-------------+---------------------+
    | ID                                   | Name   | Status | Task State | Power State | Networks            |
    +--------------------------------------+--------+--------+------------+-------------+---------------------+
    | 88e871ff-05c8-45c4-9c09-3cbc6e61eb61 | hujin1 | ACTIVE | -          | Running     | pod-net=10.244.2.62 |
    +--------------------------------------+--------+--------+------------+-------------+---------------------+
    [root@controller01 ~]# kubectl get pods -owide
    NAME      READY   STATUS    RESTARTS   AGE   IP             NODE        NOMINATED NODE   READINESS GATES
    busybox   1/1     Running   19         19h   10.244.2.195   compute02   <none>           <none>
    [root@controller01 ~]# kubectl exec -it busybox -- sh
    / # ping 10.244.2.62
    PING 10.244.2.62 (10.244.2.62): 56 data bytes
    64 bytes from 10.244.2.62: seq=0 ttl=64 time=1.193 ms
    64 bytes from 10.244.2.62: seq=1 ttl=64 time=0.961 ms
    64 bytes from 10.244.2.62: seq=2 ttl=64 time=0.594 ms
    64 bytes from 10.244.2.62: seq=3 ttl=64 time=0.685 ms
    ^C
    --- 10.244.2.62 ping statistics ---
    4 packets transmitted, 4 packets received, 0% packet loss
    round-trip min/avg/max = 0.594/0.858/1.193 ms   

pod和虚机互通实现

    [root@compute02 qemu]# ovs-vsctl show
    0e110214-9c8c-438d-9bee-d1f4ad0f89d7
        Manager "ptcp:6640:127.0.0.1"
            is_connected: true
        Bridge br-int
            fail_mode: secure
            datapath_type: system
            Port "tap59fdfbe4-10"
                Interface "tap59fdfbe4-10"
            Port "tap6e153f27-29"
                Interface "tap6e153f27-29"
            Port "ovn-b6c0da-0"
                Interface "ovn-b6c0da-0"
                    type: geneve
                    options: {csum="true", key=flow, remote_ip="172.16.41.131"}
            Port br-int
                Interface br-int
                    type: internal
            Port "ovn-11a523-0"
                Interface "ovn-11a523-0"
                    type: geneve
                    options: {csum="true", key=flow, remote_ip="172.16.41.130"}
            Port "tap2d79fedf-f5"
                Interface "tap2d79fedf-f5"
        ovs_version: "2.12.0"
    [root@compute02 qemu]# ovs-ofctl show br-int
    OFPT_FEATURES_REPLY (xid=0x2): dpid:0000ee3ac6759a4e
    n_tables:254, n_buffers:0
    capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
    actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
     7(tap2d79fedf-f5): addr:2a:c3:86:4f:bc:f2
         config:     0
         state:      0
         current:    10GB-FD COPPER
         speed: 10000 Mbps now, 0 Mbps max
     9(tap6e153f27-29): addr:fe:16:3e:bc:48:d9
         config:     0
         state:      0
         current:    10MB-FD COPPER
         speed: 10 Mbps now, 0 Mbps max
     12(tap59fdfbe4-10): addr:8e:f5:d0:5f:9d:4d
         config:     0
         state:      0
         current:    10GB-FD COPPER
         speed: 10000 Mbps now, 0 Mbps max
     13(ovn-b6c0da-0): addr:ea:25:f6:13:cc:a7
         config:     0
         state:      0
         speed: 0 Mbps now, 0 Mbps max
     14(ovn-11a523-0): addr:da:7e:b4:cd:6c:c2
         config:     0
         state:      0
         speed: 0 Mbps now, 0 Mbps max
     LOCAL(br-int): addr:ee:3a:c6:75:9a:4e
         config:     PORT_DOWN
         state:      LINK_DOWN
         speed: 0 Mbps now, 0 Mbps max
    OFPT_GET_CONFIG_REPLY (xid=0x4): frags=normal miss_send_len=0
    [root@compute02 qemu]# cat flows |grep reg15=0x5,metadata=0x4
     cookie=0x0, duration=23100.073s, table=33, n_packets=1094, n_bytes=107044, idle_age=158, priority=100,reg15=0x5,metadata=0x4 actions=load:0x1->NXM_NX_REG13[],load:0x3->NXM_NX_REG11[],load:0x2->NXM_NX_REG12[],resubmit(,34)
     cookie=0x0, duration=23100.073s, table=34, n_packets=0, n_bytes=0, idle_age=23100, priority=100,reg10=0/0x1,reg14=0x5,reg15=0x5,metadata=0x4 actions=drop
     cookie=0xb623d20f, duration=23100.061s, table=44, n_packets=0, n_bytes=0, idle_age=23100, priority=34000,udp,reg15=0x5,metadata=0x4,dl_src=fa:16:3e:18:1f:10,nw_src=10.244.0.1,tp_src=67,tp_dst=68 actions=ct(commit,zone=NXM_NX_REG13[0..15]),resubmit(,45)
     cookie=0xc28406ec, duration=22063.406s, table=44, n_packets=0, n_bytes=0, idle_age=22063, priority=2002,ct_state=-new+est-rpl+trk,ct_label=0x1/0x1,ip,reg15=0x5,metadata=0x4 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
     cookie=0x22d5455a, duration=22063.406s, table=44, n_packets=47, n_bytes=4606, idle_age=22012, priority=2002,ct_state=-new+est-rpl+trk,ct_label=0/0x1,ip,reg15=0x5,metadata=0x4 actions=resubmit(,45)
     cookie=0xc28406ec, duration=22063.406s, table=44, n_packets=2, n_bytes=196, idle_age=22054, priority=2002,ct_state=+new-est+trk,ip,reg15=0x5,metadata=0x4 actions=load:0x1->NXM_NX_XXREG0[97],resubmit(,45)
     cookie=0x43b1967d, duration=23100.063s, table=44, n_packets=0, n_bytes=0, idle_age=23100, priority=2001,ct_state=+est+trk,ct_label=0x1/0x1,ipv6,reg15=0x5,metadata=0x4 actions=drop
     cookie=0x4a7b7997, duration=23100.062s, table=44, n_packets=0, n_bytes=0, idle_age=23100, priority=2001,ct_state=+est+trk,ct_label=0/0x1,ip,reg15=0x5,metadata=0x4 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0x1->NXM_NX_CT_LABEL[0]))
     cookie=0x43b1967d, duration=23100.062s, table=44, n_packets=0, n_bytes=0, idle_age=23100, priority=2001,ct_state=+est+trk,ct_label=0x1/0x1,ip,reg15=0x5,metadata=0x4 actions=drop
     cookie=0x4a7b7997, duration=23100.061s, table=44, n_packets=0, n_bytes=0, idle_age=23100, priority=2001,ct_state=+est+trk,ct_label=0/0x1,ipv6,reg15=0x5,metadata=0x4 actions=ct(commit,zone=NXM_NX_REG13[0..15],exec(load:0x1->NXM_NX_CT_LABEL[0]))
     cookie=0x43b1967d, duration=23100.063s, table=44, n_packets=0, n_bytes=0, idle_age=23100, priority=2001,ct_state=-est+trk,ipv6,reg15=0x5,metadata=0x4 actions=drop
     cookie=0x43b1967d, duration=23100.062s, table=44, n_packets=1036, n_bytes=101528, idle_age=22063, priority=2001,ct_state=-est+trk,ip,reg15=0x5,metadata=0x4 actions=drop
     cookie=0xa1320a97, duration=23100.062s, table=48, n_packets=0, n_bytes=0, idle_age=23100, priority=90,ip,reg15=0x5,metadata=0x4,dl_dst=fa:16:3e:57:d9:64,nw_dst=255.255.255.255 actions=resubmit(,49)
     cookie=0xa1320a97, duration=23100.062s, table=48, n_packets=55, n_bytes=5390, idle_age=158, priority=90,ip,reg15=0x5,metadata=0x4,dl_dst=fa:16:3e:57:d9:64,nw_dst=10.244.2.195 actions=resubmit(,49)
     cookie=0xa1320a97, duration=23100.061s, table=48, n_packets=0, n_bytes=0, idle_age=23100, priority=90,ip,reg15=0x5,metadata=0x4,dl_dst=fa:16:3e:57:d9:64,nw_dst=224.0.0.0/4 actions=resubmit(,49)
     cookie=0xe6b695a, duration=23100.062s, table=48, n_packets=0, n_bytes=0, idle_age=23100, priority=80,ip,reg15=0x5,metadata=0x4,dl_dst=fa:16:3e:57:d9:64 actions=drop
     cookie=0xe6b695a, duration=23100.062s, table=48, n_packets=0, n_bytes=0, idle_age=23100, priority=80,ipv6,reg15=0x5,metadata=0x4,dl_dst=fa:16:3e:57:d9:64 actions=drop
     cookie=0xf4e4ab82, duration=23100.063s, table=49, n_packets=58, n_bytes=5516, idle_age=158, priority=50,reg15=0x5,metadata=0x4,dl_dst=fa:16:3e:57:d9:64 actions=resubmit(,64)
     cookie=0x0, duration=23100.073s, table=64, n_packets=3, n_bytes=126, idle_age=176, priority=100,reg10=0x1/0x1,reg15=0x5,metadata=0x4 actions=push:NXM_OF_IN_PORT[],load:0->NXM_OF_IN_PORT[],resubmit(,65),pop:NXM_OF_IN_PORT[]
     cookie=0x0, duration=23100.073s, table=65, n_packets=61, n_bytes=5642, idle_age=158, priority=100,reg15=0x5,metadata=0x4 actions=output:7

## 参考
- packstack部署： https://www.rdoproject.org/install/packstack/
- kuryr部署： https://docs.openstack.org/kuryr-kubernetes/latest/installation/manual.html
- kuryr设计文档： https://docs.openstack.org/kuryr-kubernetes/latest/devref/kuryr_kubernetes_design.html
