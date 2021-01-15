---
title: "Kuberntes Webhook"
subtitle: "keystone"
layout: post
author: "hujin"
header-style: text
tags:
  - 笔记
  - kubernetes
  - webhook
  - keystone
---

## 背景
k8s本身不提供用户和项目管理功能，需要依赖第三方组件来实现。
由于k8s和openstack的融合需求，需要提供一种统一的用户鉴权机制，本次重点介绍k8s如何使用keystone进行鉴权。通过调用keystone获取token，并使用这个token调用k8s的接口获取资源信息。



## 架构图
![k8s-keystone-auth](https://superuser.openstack.org/wp-content/uploads/2019/03/fig2.png)

主要流程：

- 客户端通过rest api调用openstack keystone获取token
- 通过获取到的token调用k8s api
- kube-apiserver 根据配置调用webhook程序，校验token并获取token对应用户的权限

## 部署
webhook和odic一样也是集成外部认证系统的一种方式，当client发起api-server请求时会触发webhook服务TokenReview调用，webhook会检查用户的凭证信息，如果是合法则返回authenticated": true等信息。api-server会等待webhook服务返回，如果返回的authenticated结果为true，则表明认证成功，否则拒绝访问。

### 配置openstack

在openstack中创建资源

    openstack role create k8s-admin
    openstack user create demo_admin --project demo --password secret
    openstack role add --user demo_admin --project demo k8s-admin

创建demo-rc,用来后面获取token

    export OS_AUTH_URL="http://172.16.41.80:35357/v3"
    export OS_USERNAME="demo_admin"
    export OS_PASSWORD="secret"
    export OS_PROJECT_NAME="demo"
    export OS_USER_DOMAIN_NAME=Default
    export OS_PROJECT_DOMAIN_NAME=Default
    export OS_REGION_NAME=RegionTwo
    export OS_IDENTITY_API_VERSION=3
    export OS_IMAGE_API_VERSION=2
    export PYTHONIOENCODING='utf-8'

### kubernetes中创建k8s-keystone-auth webhook

创建keystone权限configmap

    cat keystone_configmap.yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: k8s-auth-policy
      namespace: kube-system
    data:
      policies: |
        [
          {
            "users": {
              "projects": ["demo"],
              "roles": ["member"]
            },
            "resource_permissions": {
              "*/pods": ["get", "list", "watch"]
            }
          }
        ]

    kubectl apply -f keystone_configmap.yaml

创建证书，给keystone-webhook容器使用

    openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj /CN=k8s-keystone-auth.kube-system/
    kubectl --namespace kube-system create secret tls keystone-auth-certs --cert=cert.pem --key=key.pem

创建keystone rbac

    cat keystone-rbac.yaml
    kind: ClusterRole
    apiVersion: rbac.authorization.k8s.io/v1
    metadata:
      labels:
        k8s-app: k8s-keystone-auth
      name: k8s-keystone-auth
    rules:
      # Allow k8s-keystone-auth to get k8s-auth-policy configmap
    - apiGroups: [""]
      resources: ["configmaps"]
      verbs: ["get", "watch", "list"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: k8s-keystone-auth
      labels:
        k8s-app: k8s-keystone-auth
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: k8s-keystone-auth
    subjects:
    - kind: ServiceAccount
      name: k8s-keystone
      namespace: kube-system
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: k8s-keystone
      namespace: kube-system


部署 k8s-keystone-auth，注意替换`keystone-url`

    cat keystone-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: k8s-keystone-auth
      namespace: kube-system
      labels:
        app: k8s-keystone-auth
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: k8s-keystone-auth
      template:
        metadata:
          labels:
            app: k8s-keystone-auth
        spec:
          serviceAccountName: k8s-keystone
          containers:
            - name: k8s-keystone-auth
              image: k8scloudprovider/k8s-keystone-auth:latest
              args:
                - ./bin/k8s-keystone-auth
                - --tls-cert-file
                - /etc/pki/tls.crt
                - --tls-private-key-file
                - /etc/pki/tls.key
                - --policy-configmap-name
                - k8s-auth-policy
                - --keystone-url
                - http://172.16.41.80:35357/v3
              volumeMounts:
                - mountPath: /etc/pki
                  name: certs
                  readOnly: true
              ports:
                - containerPort: 8443
          volumes:
          - name: certs
            secret:
              secretName: keystone-auth-certs


创建keystone_service，这里建议创建后给service绑定浮动IP，方便用来给外部调用用

    cat keystone-service.yaml
    kind: Service
    apiVersion: v1
    metadata:
      name: k8s-keystone-auth-service
      namespace: kube-system
    spec:
      selector:
        app: k8s-keystone-auth
      ports:
        - protocol: TCP
          port: 8443
          targetPort: 8443

验证k8s-keystone-auth服务

    # input:
    token = `source demo-rc;openstack token issue -f yaml -c id |awk '{print $2}'`
    kubectl run curl --rm -it --restart=Never --image curlimages/curl -- \
      -k -XPOST https://k8s-keystone-auth-service.kube-system:8443/webhook -d '
    {
      "apiVersion": "authentication.k8s.io/v1beta1",
      "kind": "TokenReview",
      "metadata": {
        "creationTimestamp": null
      },
      "spec": {
        "token": "'$token'"
      }
    }'

    # output:
    {
      "apiVersion": "authentication.k8s.io/v1beta1",
      "kind": "TokenReview",
      "metadata": {
        "creationTimestamp": null
      },
      "spec": {
        "token": "gAAAAABf_7dmngBBu9cThzYmRs8Hkqv9Gm1FBRL0duTFAv7xl5dwwHA3yhKnCo_cqvsnKt90ukdmV5crq1s6EgBjh_e5cvhGBejFdRADViH9Vmmr6KI2L9I8gG4Dkj52whKNVqxZ-2R81rOj_Amqj83Iwa5TEWURXVfKNaL9ktLPR3-qY4TkjWU"
      },
      "status": {
        "authenticated": true,
        "user": {
          "username": "demo_admin",
          "uid": "b9b167f5e86b48839a879e111e20a0b1",
          "groups": [
            "bf37908a629b4d2ca02dfc840da027cd"
          ],
          "extra": {
            "alpha.kubernetes.io/identity/project/id": [
              "bf37908a629b4d2ca02dfc840da027cd"
            ],
            "alpha.kubernetes.io/identity/project/name": [
              "demo"
            ],
            "alpha.kubernetes.io/identity/roles": [
              "k8s-admin"
            ],
            "alpha.kubernetes.io/identity/user/domain/id": [
              "default"
            ],
            "alpha.kubernetes.io/identity/user/domain/name": [
              "Default"
            ]
          }
        }
      }

### 配置kube-apisever

    mkdir /etc/kubernetes/webhooks
    cat <<EOF > /etc/kubernetes/webhooks/webhookconfig.yaml
    ---
    apiVersion: v1
    kind: Config
    preferences: {}
    clusters:
      - cluster:
          insecure-skip-tls-verify: true
          server: https://178.119.220.88:8443/webhook
        name: webhook
    users:
      - name: webhook
    contexts:
      - context:
          cluster: webhook
          user: webhook
        name: webhook
    current-context: webhook
    EOF

> 178.119.220.88是k8s-keystone-auth的浮动IP


    vim /etc/kubernetes/manifests/kube-apiserver.yaml
    spec:
      containers:
      - command:
      ...
      - --authentication-token-webhook-config-file=/etc/kubernetes/webhooks/webhookconfig.yaml
      - --authorization-webhook-config-file=/etc/kubernetes/webhooks/webhookconfig.yaml
      - --authorization-mode=Node,RBAC,Webhook

      volumeMounts:
      ...
      - mountPath: /etc/kubernetes/webhooks
        name: webhooks
        readOnly: true
    volumes:
    ...
    - hostPath:
        path: /etc/kubernetes/webhooks
        type: DirectoryOrCreate
      name: webhooks

> 修改完成后kube-apiserver会自动重启，注意查看日志是否有报错


## 测试

脚本：

    import eventlet
    eventlet.monkey_patch()

    from keystoneclient.v3 import client as keystone_client
    import requests
    from pprint import pprint
    import json


    def get_openstack_token():
        keystone = keystone_client.Client(username='demo_admin', password='secret',
                                          auth_url='http://172.16.41.80:35357/v3',
                                          tenant_name='demo',
                                          project_domain_name='Default',
                                          user_domain_name='Default',
                                          project_name='demo')
        print(keystone.auth_token)
        return keystone.auth_token


    def check_token(token):
        data = {"apiVersion": "authentication.k8s.io/v1beta1",
                "kind": "TokenReview",
                "metadata": {
                    "creationTimestamp": None
                },
                "spec": {
                    "token":  token
                }}
        headers = {'Content-Type': 'application/json', 'Connection': 'Keep-Alive'}
        req = requests.post('https://178.119.220.88:8443/webhook',
                            data=json.dumps(data), headers=headers, verify=False, timeout=5)
        print(req.content)


    def list_ingress(tk):
        auth_url = 'https://179.18.3.180:6443'
        headers = {'Connection': 'Keep-Alive',
                   'Authorization': 'Bearer %s' % tk}
        req_url = '%s/api/v1/namespaces' % auth_url
        print('='*20, req_url, headers)
        req = requests.get(req_url, headers=headers, verify=False)
        pprint(json.loads(req.content))

    token = get_openstack_token()
    list_ingress(token)


## 参考文档：
- https://github.com/kubernetes/cloud-provider-openstack/blob/master/docs/keystone-auth/using-keystone-webhook-authenticator-and-authorizer.md
- https://k2r2bai.com/2018/05/30/kubernetes/keystone-auth/
- https://zhuanlan.zhihu.com/p/97797321