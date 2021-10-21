---
title: "DockerCE20.10版本打包流程"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - docker
  - docker-ce

---


## 背景
Docker-CE分支在v20.10版本之后将会停止更新，原先的docker-ce将拆分成docker/cli和moby/moby两个项目，其中docker/cli就是docker的客户端，也就是我们常用的docker命令行工具所属的项目；moby/moby项目就是原先docker engine的部分

## 环境准备

在编译docker源码之前，需要安装docker-ce

    yum install docker-ce -y

## 获取项目代码

根据需求获取最新的docker/cli和moby/moby项目代码和切换版本

    cd /root
    git clone https://github.com/docker/cli.git
    cd cli && git checkout v20.10.7
    git clone https://github.com/moby/moby.git
    cd moby && git checkout v20.10.7
    git clone https://github.com/docker/scan-cli-plugin.git

> 根据实际情况适当翻墙或者使用国内加速优化方式

### 【可选】编译二进制文件

切换分支

    cd cli
    git chekout 20.10
    
    cd moby
    git checkout 20.10
    
编译cli

    make -f docker.Makefile binary
    
> 在build目录下是编译好的文件

编译moby

在git clone之前添加代理

    hack/dockerfile/install/containerd.installer
	hack/dockerfile/install/dockercli.installer
	hack/dockerfile/install/proxy.installer
	hack/dockerfile/install/rootlesskit.installer
	hack/dockerfile/install/runc.installer
	hack/dockerfile/install/shfmt.installer
	hack/dockerfile/install/tini.installer
	hack/dockerfile/install/tomlv.installer
	hack/dockerfile/install/vndr.installe

编译

    make binary
    
> 在bundles/binary-daemon/目录下是编译好的文件

验证，停止节点上的docker

    systemctl stop docker.service

将之前编译的docker和dockerd替换
    
    mv /usr/bin/docker /home/backup
    cp cli/build/docker /usr/bin/docker
    chmod +x /usr/bin/docker
    
    mv /usr/bin/dockerd /home/backup
    cp moby/bundles/binary-daemon/dockerd /usr/bin/dockerd
    chmod +x /usr/bin/dockerd
    
启动docker

    systemctl start docker.service
    
    docker version

启动测试容器

    docker pull alpine
    docker run alpine echo "hello from alpine"
    
    
### 编译RPM包

获取打包项目代码

    git clone https://github.com/docker/docker-ce-packaging.git
    cd docker-ce-packaging
    
【可选】根据需求切换分支

    git checkout v20.10.0-beta1
    
> 注意：这里的版本建议和docker/cli等项目逇版本保持一致
    
创建代码目录

    mkdir -p src/github.com/docker/
    
将上面git clone下来的代码放到对应目录

    cp -r /root/cli src/github.com/docker/
    cp -r /root/moby src/github.com/docker/docker # 这里一定要改成docker名称，否则会出现一系列错误
    cp -r /root/scan-cli-plugin src/github.com/docker/
    
在对应项目里根据需求切换分支或者修改代码

【可选】设置docker项目https代理
    
    vi ./src/github.com/docker/docker/hack/dockerfile/install/runc.installer
    vi ./src/github.com/docker/docker/hack/dockerfile/install/containerd.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/dockercli.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/proxy.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/rootlesskit.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/runc.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/shfmt.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/tini.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/tomlv.installer
	vi ./src/github.com/docker/docker/hack/dockerfile/install/vndr.installer
	vi ./src/github.com/docker/docker/vendor/github.com/docker/libnetwork/network.go
    
    vi rpm/SPECS/docker-ce-cli.spec
    ...
    %build
    export https_proxy=http://10.51.30.48:1080
    ...
    
    
开始编译(根据需求选择对应的版本和系统)

    cd rpm
    VERSION=20.10.7 make centos-7
    
生成的文件在

    [root@localhost rpm]# pwd
    /root/docker-ce-packaging/rpm
    [root@localhost rpm]# ll rpmbuild/centos-7/RPMS/x86_64
    total 65428
    -rw-r--r-- 1 root root 23793492 Oct 20 18:47 docker-ce-20.10.7.chinac-3.el7.x86_64.rpm
    -rw-r--r-- 1 root root 31183248 Oct 20 18:51 docker-ce-cli-20.10.7.chinac-3.el7.x86_64.rpm
    -rw-r--r-- 1 root root  8424840 Oct 20 18:52 docker-ce-rootless-extras-20.10.7.chinac-3.el7.x86_64.rpm
    -rw-r--r-- 1 root root  3591600 Oct 20 18:53 docker-scan-plugin-0.8.0-3.el7.x86_64.rpm  


## Troubleshooting

### cli/compose/schema/schema.go:4:2: cannot find package "embed" in any of:
低版本的golang会有这个问题，需要修改编译时指定的golang版本

    vi common.tk
    ...
    GO_VERSION=1.16.9
> 注意：不是升级本地的golang，是容器中的，所以只需要改下这个文件

## 参考
- https://openpower.ic.unicamp.br/post/building-docker-for-power/
- https://www.fatalerrors.org/a/pull-and-compile-docker-ce.html
