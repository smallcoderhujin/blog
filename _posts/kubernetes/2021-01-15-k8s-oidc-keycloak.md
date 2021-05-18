---
title: "Kuberntes OIDC Keycloak"
subtitle: "Keycloak"
layout: post
author: "hujin"
header-style: text
tags:
  - kubernetes
  - oidc
  - keycloak
---

# 架构图
![keyloak.png](http://langyxxl.dynv6.net:50008/data/User/admin/home/%E6%8A%80%E6%9C%AF/K8S/keyloak.png)

OIDC是一种 OAuth2 认证方式， 被某些 OAuth2 提供者支持，例如 Azure 活动目录、Salesforce 和 Google。 协议对 OAuth2 的主要扩充体现在有一个附加字段会和访问令牌一起返回， 这一字段称作 ID Token（ID 令牌）。 ID 令牌是一种由服务器签名的 JSON Web 令牌（JWT）

- 登录到你的身份服务（Identity Provider）
- 你的身份服务将为你提供 access_token、id_token 和 refresh_token
- 在使用 kubectl 时，将 id_token 设置为 --token 标志值，或者将其直接添加到 kubeconfig 中
- kubectl 将你的 id_token 放到一个称作 Authorization 的头部，发送给 API 服务器
- API 服务器将负责通过检查配置中引用的证书来确认 JWT 的签名是合法的
- 检查确认 id_token 尚未过期 x
- 确认用户有权限执行操作
- 鉴权成功之后，API 服务器向 kubectl 返回响应
- kubectl 向用户提供反馈信息

# 部署keycloak

## 准备

    # 安装java
    yum install java-11-openjdk
    # 获取源码包
    wget https://github.com/keycloak/keycloak/releases/download/12.0.1/keycloak-12.0.1.tar.gz
    tar xzvf keycloak-12.0.1.tar.gz
    cd keycloak-12.0.1

##  启用admin用户

    sh bin/add-user-keycloak.sh --user admin


## 创建证书，注意修改ip地址，证书创建到/root/ssl目录

    vim /root/makessl.sh

    #!/bin/bash

    mkdir -p ssl

    cat << EOF > ssl/ca.cnf
    [req]
    req_extensions = v3_req
    distinguished_name = req_distinguished_name

    [req_distinguished_name]

    [ v3_req ]
    basicConstraints = CA:TRUE
    EOF

    cat << EOF > ssl/req.cnf
    [req]
    req_extensions = v3_req
    distinguished_name = req_distinguished_name

    [req_distinguished_name]

    [ v3_req ]
    basicConstraints = CA:FALSE
    keyUsage = nonRepudiation, digitalSignature, keyEncipherment
    subjectAltName = @alt_names

    [alt_names]
    IP.1 = 172.16.41.106   # 修改成当前机器的外网IP,否则会报错误：x509: cannot validate certificate for 172.16.41.106 because it doesn't contain any IP SANs
    EOF

    openssl genrsa -out ssl/ca-key.pem 2048
    openssl req -x509 -new -nodes -key ssl/ca-key.pem -days 365 -out ssl/ca.pem -subj "/CN=keycloak-ca" -extensions v3_req -config ssl/ca.cnf

    openssl genrsa -out ssl/keycloak.pem 2048
    openssl req -new -key ssl/keycloak.pem -out ssl/keycloak-csr.pem -subj "/CN=keycloak" -config ssl/req.cnf
    openssl x509 -req -in ssl/keycloak-csr.pem -CA ssl/ca.pem -CAkey ssl/ca-key.pem -CAcreateserial -out ssl/keycloak.crt -days 365 -extensions v3_req -extfile ssl/req.cnf


## 导入证书（密码尽量保持一致）：

    cd ssl
    openssl pkcs12 -export -out keycloak.p12 -inkey keycloak.pem -in keycloak.crt -certfile ca.pem
    keytool -importkeystore -deststorepass 'passw0rd' -destkeystore keycloak.jks -srckeystore keycloak.p12 -srcstoretype PKCS12

## 将证书拷贝至configuration目录

    cp keycloak.jks ../keycloak-12.0.1/standalone/configuration

## 检查证书

    # 确认证书的Alias Name，会在下面的standalone.xml中引用
    cd ../keycloak-12.0.1/standalone/configuration
    keytool -list -keystore keycloak.jks -v

## 修改keycloak配置

    vim /root/keycloak-12.0.1/standalone/configuration/standalone.xml

    <management>
    <management>
    <security-realms>
        <security-realm name="ApplicationRealm">
            <server-identities>
             <ssl>
                <keystore path="keycloak.jks" relative-to="jboss.server.config.dir" keystore-password="passw0rd" alias="server" key-password="passw0rd" generate-self-signed-certificate-host="localhost"/>
              </ssl>
            </server-identities>
            ...
        </security-realm>
        <security-realm name="UndertowRealm">
            <server-identities>
                <ssl>
                    <keystore path="keycloak.jks" relative-to="jboss.server.config.dir" keystore-password="passw0rd" />
                </ssl>
            </server-identities>
        </security-realm>

##  运行程序, 指定外网IP地址
    sh bin/standalone.sh -Djboss.bind.address=172.16.41.106 -Djboss.bind.address.management=172.16.41.106

## 访问页面

    登录：https://172.16.41.106:8443/auth/admin


# 初始化keycloak

## 创建realm kubernetes
页面菜单栏顶部有个“v”图标，点击并创建realm“Add realm”

## 创建client kubernetes
![create realm kubernetes](https://developer.ibm.com/developer/default/articles/cl-lo-openid-connect-kubernetes-authentication2/images/image001.png)

![secret key](https://developer.ibm.com/developer/default/articles/cl-lo-openid-connect-kubernetes-authentication2/images/image002.png)

注意：
- access type要设置成confidential，否则后面无法获取secret key
- valid redirect url随意填写，这里填的：http://localhost:32768


## 创建用户

创建用户

    菜单栏选择“Users”后点击“Add user”，创建用户：theone（名字随意，保持一致即可）

设置密码

![](https://developer.ibm.com/developer/default/articles/cl-lo-openid-connect-kubernetes-authentication2/images/image003.png)

激活用户

    访问https://172.16.41.106:8443/auth/admin/kubernetes/console
    使用theone用户登录，登录后404 没有权限，可以忽略；登录后要求重新设置密码，还可以设置成之前的密码，没有关系


# 配置kubernetes

拷贝证书

    将/root/ssl/ca.pem或者keycloak.crt证书文件，从keycloak节点拷贝到k8s master节点的/etc/kubernetes/ssl/目录下

修改kube apiserver配置

    vim /etc/kubernetes/manifests/kube-apiserver.yaml
    spec:
    containers:
    - command:
    ...
    - --oidc-issuer-url=https://172.16.41.106:8443/auth/realms/kubernetes
    - --oidc-client-id=kubernetes
    - --oidc-username-claim=preferred_username
    - --oidc-username-prefix=-
    - --oidc-ca-file=/etc/kubernetes/pki/ca.pem

此时kube api server会自动重启，注意查看容器日志

    kubectl logs kube-apiserver-xxx -f -n kube-system

权限设置

    #cat rolebing.yaml

    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      annotations:
        rbac.authorization.kubernetes.io/autoupdate: "true"
      labels:
        kubernetes.io/bootstrapping: rbac-defaults
      name: cluster-readonly
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - apiGroup: rbac.authorization.k8s.io
      kind: User
      name: theone   # 替换成之前在keycloak中创建的用户名

# 测试脚本
    import requests
    from pprint import pprint
    import json


    auth_url = 'https://179.18.3.180:6443'
    headers = {'Connection': 'Keep-Alive'}

    def get_token():
        url = 'https://172.16.41.106:8443/auth/realms/kubernetes/protocol/openid-connect/token'
        req = requests.post(url, data={'client_id': 'kubernetes',
                                       'client_secret': 'aa3e7e64-d947-493e-93ca-d43faae6585e',
                                       'response_type': 'code token',
                                       'grant_type': 'password',
                                       'username': 'theone',
                                       'password': 'test',
                                       'scope': 'openid'},
                           verify=False)
        r = json.loads(req.content)
        pprint(r)
        return r


    def list_ingress(tk):
        req_url = '%s/api/v1/pods' % auth_url
        headers['Authorization'] = 'Bearer %s' % tk
        print(req_url, headers)
        req = requests.get(req_url, headers=headers, verify=False)
        pprint(json.loads(req.content))

    tokes = get_token()
    list_ingress(tokes['id_token'])

参考
- https://developer.ibm.com/zh/depmodels/cloud/articles/cl-lo-openid-connect-kubernetes-authentication/
- https://developer.ibm.com/zh/articles/cl-lo-openid-connect-kubernetes-authentication2/