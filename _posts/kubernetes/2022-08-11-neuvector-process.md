---
title: "Neuvector源码分析之 进程管理"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - neuvector
  - 进程管理

---


## 功能介绍
![neuvector_pcap](/blog/img/neuvector_process.png)

进程管理是针对容器资源的进程进行管理，用户在界面可以先查询资源，在资源的进程列表中维护进程信息
在agent中进程管理的实现实际就是kill，只是在kill之前根据探测到的资源进程信息和用户下发的规则对比，得到哪些进程需要kill

通过进程管理，可以防止异常命令的执行，只要在容器中执行非正常命令，直接就被kill掉，有效保护容器安全

## 源码

接口：

    r.PATCH("/v1/process_profile/:name", handlerProcessProfileConfig)