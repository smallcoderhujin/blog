---
title: "Neuvector 初探"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - neuvector

---

## 介绍

![neuvector_arch](/blog/img/neuvector_arch1.png)

NeuVector 是业界首个端到端的开源容器安全平台，唯一为容器化工作负载提供企业级零信任安全的解决方案。
NeuVector 可以提供实时深入的容器网络可视化、东西向容器网络监控、主动隔离和保护、容器主机安全以及
容器内部安全，容器管理平台无缝集成并且实现应用级容器安全的自动化，适用于各种云环境、跨云或者本地部署等容器生产环境。

我们重点看neuvector的网络功能。 neuvector的网络架构中分为controller和agent端
- controller：控制服务，提供api接口，启动并watch consul数据库
- agent：节点服务，负责监控节点上容器资源，根据用户规则执行响应操作

控制服务在提供统一的接口服务的同时，还运行consul服务，用来提供数据持久化功能，用户在调用neuvector接口后会将数据保存到key/value数据库consul中
控制服务和agent的通信有两种方式：
- watch consul:由于每个节点上的agent服务会watch consul的数据变动（类似etcd），其实是watch指定的key；在控制服务往consul中增加新的记录后，
agent在watch到后根据对应key调用对应的方法处理。 
- grpc：controller在某些功能中会直接通过grpc的方式调用agent，比如容器网络抓包功能

agent服务除了响应controller的grpc调用以外，在启动时会创建一个独立的协程来监听当前节点容器资源的变化，会获取当前节点使用的runtime来调用对应的monitor方法
runtime支持docker/containerd/crio，支持获取容器的事件包括：
- start
- stop
- delete
- copyin
- copyout
- network create
- network delete
- socket error

dp是一个内核模块，在agent中维护，通过socket和agent进行通信，用来具体执行用户的网络规则，对容器网卡的in/out/status流量进行处理，详细的会在网络策略时讲

后面会重点从源码角度对neuvector的网络抓包、进程管理、文件管理和网络策略进行分析

## 部署

这里我们通过helm快速部署

安装helm

    curl -fsSL -o get_helm.sh     https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
        
添加repo

    helm repo add neuvector https://neuvector.github.io/neuvector-helm/
    helm search repo neuvector/core
    
开始部署

    kubectl create namespace neuvector
    kubectl create serviceaccount neuvector -n neuvector
    
    helm install neuvector --namespace neuvector neuvector/core  \
    --set registry=harbor.archeros.cn/dev  --set tag=5.0.1 \
    --set=controller.image.repository=neuvector/controller \
    --set=enforcer.image.repository=neuvector/enforcer \
    --set manager.image.repository=neuvector/manager \
    --set cve.scanner.image.repository=neuvector/scanner \
    --set cve.updater.image.repository=neuvector/updater
    
访问webui
    
    [root@node1 ~]# kubectl get svc -n neuvector  |grep webui
    neuvector-service-webui           NodePort    10.68.204.173   <none>        8443:30080/TCP                  5d

    浏览器访问： https://<管理ip>:30080
    默认用户密码都是: admin

后面我们将从源码级别来逐个分析neuvector的网络功能
    


