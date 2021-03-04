---
title: "TungstenFabric Kubernetes Annotation"
subtitle: "annotation"
layout: post
author: "hujin"
header-style: text
tags:
  - kubernetes
  - tungstenfabric
  - annotation
---

## Annotation整理

名称	                                    |值	                                                                                                            |说明
-|-|-
opencontrail.org/fip-pool	            |"{'domain': 'default-domain', 'project': 'ArcherAdmin', 'network': 'public1420', 'name': 'floating-ip-pool'}"  |指定fip-pool
opencontrail.org/network                |'opencontrail.org/network': '{"domain":"default-domain", "project": "ArcherAdmin", "name":"test-net"}'         |指定network,如果设置在pod中，pod使用指定网络建网卡，如果指定在namespace中，则namespace中所有pod使用指定网络建网卡
opencontrail.org/isolation              |'opencontrail.org/isolation': 'true'                                                                           |设置namespace是否隔离
opencontrail.org/ip_fabric_forwarding   |"opencontrail.org/ip_fabric_forwarding" : "true"                                                               |使用fabric网络创建pod网卡
opencontrail.org/ip_fabric_snat         |"opencontrail.org/ip_fabric_snat" : "true"                                                                     |启用pod网络的fabric_snat
opencontrail.org/cidr                   |"opencontrail.org/cidr" : "1.0.0.0/24"                                                                         |使用自定义网络（非contrail网络）时指定


## 参考

- https://github.com/kubevirt/user-guide/pull/196/files/700980d5099b969c4fe2defb3773ec84e628a207
- https://www.juniper.net/documentation/en_US/day-one-books/topics/concept/contrail-as-a-cni.html
- https://www.juniper.net/documentation/en_US/contrail19/topics/concept/kubernetes-cni-contrail.html