---
title: "Cinder"
subtitle: "csi"
layout: post
author: "hujin"
header-style: text
tags:
  - k8s
  - csi
  - cinder

---

## 背景
由于openstack cinder提供丰富的后端存储服务，当k8s部署在openstack的虚拟机中时，使用cinder 的csi提供存储服务可以非常方便对接不同的后端存储

## 要求
- 分别部署openstack和k8s集群
- openstack cinder使用v3 api
- openstack nova和cinder在keystone中的endpoint url对应的ip地址，k8s的虚机可以访问到
- openstack开启metadata服务
- openstack虚机中需要安装cloud-init
> /var/lib/cloud/data/instance-id虚机中这个目录必须是虚机的uuid，这个是通过cloud-init注入的
- 虚机挂盘后要求在/dev/disk/by-id目录下有对应volume的设备存在

## 部署

获取k8s-openstack-provider源码（非必须）

    git clone https://github.com/kubernetes/cloud-provider-openstack
    cd cloud-provider-openstack

编译（需要docker-ce 17.05以上的版本， docker需要翻墙）

    export ARCH=amd64 # Defaults to amd64
    编译： make build-cmd-cinder-csi-plugin
    生成镜像：make image-cinder-csi-plugin

修改kubelet配置

    vim /etc/kubernetes/kubelet.env
    KUBELET_CLOUDPROVIDER="--cloud-provider=external"

配置openstack信息并使用base64加密

    vim cloud.conf
    [Global]
    username = ArcherAdmin
    password = ArcherAdmin@123
    domain-name = Default
    auth-url = http://172.118.23.20:45357/v3
    tenant-id = ad88dd5d24ce4e2189a6ae7491c33e9d
    region = RegionOne

    [Metadata]
    search-order = configDrive,metadataService

使用base64对openstack配置加密

    cat cloud.conf | base64 -w 0

获取社区yaml文件

    https://github.com/kubernetes/cloud-provider-openstack/tree/master/manifests/cinder-csi-plugin

将之前openstack的配置对应的base64结果更新到csi-secret-cinderplugin.yaml文件中

创建cinder csi资源

    kubectl apply -f cinder-csi-plugin

查看cinder csi pod状态

    [root@node1 csi-cinder]# kubectl get pods -n kube-system
    NAME                                       READY   STATUS        RESTARTS   AGE
    csi-cinder-controllerplugin-0              5/5     Running       25         4h12m
    csi-cinder-nodeplugin-cwzpr                2/2     Running       0          7m28s
    csi-cinder-nodeplugin-wxl6f                2/2     Running       0          7m29s

创建storeageclass和pvc

    cat sc.yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: csi-sc-cinderplugin
    provisioner: cinder.csi.openstack.org


    cat pvc.yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: csi-pvc-cinderplugin
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: csi-sc-cinderplugin

确认sc和pvc状态

    [root@node1 resource-yamls]# kubectl get sc
    NAME                  PROVISIONER                RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
    csi-sc-cinderplugin   cinder.csi.openstack.org   Delete          Immediate           false                  26h
    csi-sc-hujin          arstor.csi.huayun.io       Delete          Immediate           false                  4d9h
    [root@node1 resource-yamls]# kubectl get pvc
    NAME                   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
    csi-pvc-cinderplugin   Bound    pvc-e5aec543-ff37-40ad-85d5-ce038975e14c   1Gi        RWO            csi-sc-cinderplugin   12m
    [root@node1 resource-yamls]#

创建pod

    cat pod.yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: nginx
    spec:
      containers:
      - image: nginx:alpine
        imagePullPolicy: IfNotPresent
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        volumeMounts:
          - mountPath: /var/lib/www/html
            name: csi-data-cinderplugin
      volumes:
      - name: csi-data-cinderplugin
        persistentVolumeClaim:
          claimName: csi-pvc-cinderplugin
          readOnly: false

从pv中获取volume id

    [root@node1 resource-yamls]# kubectl describe pv pvc-e5aec543-ff37-40ad-85d5-ce038975e14c
    Name:            pvc-e5aec543-ff37-40ad-85d5-ce038975e14c
    Labels:          <none>
    Annotations:     pv.kubernetes.io/provisioned-by: cinder.csi.openstack.org
    Finalizers:      [kubernetes.io/pv-protection external-attacher/cinder-csi-openstack-org]
    StorageClass:    csi-sc-cinderplugin
    Status:          Bound
    Claim:           default/csi-pvc-cinderplugin
    Reclaim Policy:  Delete
    Access Modes:    RWO
    VolumeMode:      Filesystem
    Capacity:        1Gi
    Node Affinity:   <none>
    Message:
    Source:
        Type:              CSI (a Container Storage Interface (CSI) volume source)
        Driver:            cinder.csi.openstack.org
        FSType:            ext4
        VolumeHandle:      ef9037a7-9a67-408a-9f92-adbd2badc5db
        ReadOnly:          false
        VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1616250534343-8081-cinder.csi.openstack.org
    Events:                <none>
    [root@node1 resource-yamls]#

在openstack中查看volume状态

    [root@controller01 ~]# cinder list |grep ef9037a7
    | ef9037a7-9a67-408a-9f92-adbd2badc5db |   in-use  | pvc-e5aec543-ff37-40ad-85d5-ce038975e14c |  1   | basic-replica2 |  false   | 94a932d0-79da-4ed2-a228-a4e96264d1c0 |

确认pod状态

    [root@node1 csi-cinder]# kubectl get pods     -owide
    NAME    READY   STATUS    RESTARTS   AGE   IP             NODE    NOMINATED NODE   READINESS GATES
    nginx   1/1     Running   0          35s   10.244.28.16   node3   <none>           <none>

    [root@node1 resource-yamls]# kubectl exec -it nginx -- sh
    / # ls /var/lib/www/html/
    lost+found
    / #

## 测试场景

### 命令行删除deployment pod
删除后正常新建pod，且使用原来的volume

### k8s虚机网络故障
deployment中其中一个pod所在节点down机后，新建pod失败，报volume in-use
statefulset中其中一个pod所在节点down机后，不新建pod

### k8s虚机重启
同节点down机

### 虚机迁移
热迁移不影响pod使用
冷迁移：同节点down机

### k8s虚机对应计算节点故障
此时原来的pod一直是删除状态，新建的pod也无法正常创建，必须等待down机的节点恢复
这是一种安全的做法

### 虚机挂在最大磁盘个数
这是一个bug，升级qemu后可以解决


## 参考
- https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/cinder-csi-plugin/using-cinder-csi-plugin.md
- https://www.jianshu.com/p/87b02040991c
- https://silenceper.com/kubernetes-book/csi/how-to-write-csi-driver.html