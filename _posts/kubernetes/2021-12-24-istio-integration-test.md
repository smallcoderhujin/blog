---
title: "Istio集成测试"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - istio
  - integration-test

---


## 简介
istio集成测试使用go test，会自动读取源码目录下面名为 *_test.go 的文件，生成并运行测试用的可执行文件。istio集成测试脚本中根据case定义一般会先部署istio集群，再部署对应的echo instance，最后执行具体的case。

## 准备
修改master节点apiserver参数
这里需要支持第三方token(third-party-token)，默认k8s使用first-party-jwt

    cat /etc/kubernetes/manifests/kube-apiserver.yaml
    ...
    - --service-account-signing-key-file=/etc/kubernetes/ssl/sa.key
    - --service-account-key-file=/etc/kubernetes/ssl/sa.pub
    - --service-account-issuer=api
    - --service-account-api-audiences=api,vault,factors
    ...
    
安装metallb组件

由于istio集成测试时会部署loadbalancer类型的service，在独立的k8s环境中没有上有的LB提供服务，因此需要引入metallb组件

metallb分为l2模式和bgp模式，这里我们使用l2模式

开启strictARP

    # see what changes would be made, returns nonzero returncode if different
    kubectl get configmap kube-proxy -n kube-system -o yaml | \
    sed -e "s/strictARP: false/strictARP: true/" | \
    kubectl diff -f - -n kube-system
    
    # actually apply the changes, returns nonzero returncode on errors only
    kubectl get configmap kube-proxy -n kube-system -o yaml | \
    sed -e "s/strictARP: false/strictARP: true/" | \
    kubectl apply -f - -n kube-system

创建metallb-system namespace

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml

下载configmap，并修改address参数，预留一段k8s管理网络IP段给Loadbalancer类型的service使用

    wget https://github.com/metallb/metallb/blob/v0.11.0/manifests/example-layer2-config.yaml
    vi example-layer2-config.yaml
    mv example-layer2-config.yaml l2-config.yaml
    kubectl apply -f l2-config.yaml

安装metallb speaker和controller等资源

    kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml


提前下载的镜像


    gcr.io/istio-testing/app:1.12-dev
    gcr.io/istio-testing/operator:1.12-dev
    gcr.io/istio-testing/proxyv2:1.12-dev
    gcr.io/istio-testing/pilot:1.12-dev
    gcr.io/istio-testing/app_sidecar_ubuntu_bionic:1.12-dev
    gcr.io/istio-testing/fake-gce-metadata:1.0
    gcr.io/istio-testing/ext-authz:0.7

    jimmidyson/configmap-reload:v0.5.0
    envoyproxy/ratelimit:6f5de117
    openzipkin/zipkin-slim:2.23.0
    gcr.io/istio-release/pilot:1.6.11
    gcr.io/istio-release/pilot:1.7.6
    gcr.io/istio-release/pilot:1.8.6
    gcr.io/istio-release/pilot:1.9.5
    gcr.io/istio-release/pilot:1.10.0
    gcr.io/istio-release/proxyv2:1.11.3


下载istio源码，当前测试的版本是release-1.12

    git clone https://github.com/istio/istio.git -b release-1.12
    cd istio
    
## 集成测试
go test命令行参数介绍

- -p 允许并行执行通过调用 t.Parallel 的测试函数的最大次数
- -vet 在 “go test “期间对 “go vet ” 的调用，以使用逗号分隔的vet检查列表, off表示不执行go vet
- -v 显示测试的详细命令
- -count 运行每个测试和基准测试的次数（默认 1）
- -timeout 执行二进制文件超时时间，超过会报panic
- -tags

### telemetry集成测试

    go test -p 1 -vet=off -v -count=1 -tags=integ ./tests/integration/telemetry/... -timeout 30m \
    --istio.test.istio.istiodlessRemotes --istio.test.ci --istio.test.work_dir=/logs/artifacts \
    --istio.test.tag=1.12-dev --istio.test.pullpolicy=IfNotPresent \
    --istio.test.hub=harbor.huayun.org/huayun-kubernetes/istio-testing
    
telemetry失败case:
- TestVMTelemetry: 依赖谷歌的GCP项目 <https://github.com/istio/istio/issues/35923>，需要临时删除这个case： git rm -r tests/integration/telemetry/stackdriver/vm/

### security集成测试

    go test -p 1 -vet=off -v -count=1 -tags=integ ./tests/integration/security/... -timeout 30m \
    --istio.test.work_dir=/logs/artifacts --istio.test.tag=1.12-dev \
    --istio.test.pullpolicy=IfNotPresent --istio.test.skip TestAuthorization_JWT \
    --istio.test.skip TestAuthorization_EgressGateway \
    --istio.test.skip TestRequestAuthentication \
    --istio.test.skip TestIngressRequestAuthentication \
    --istio.test.hub=harbor.huayun.org/huayun-kubernetes/istio-testing

security失败case:
- TestAuthorization_JWT:
- TestAuthorization_EgressGateway:
- TestRequestAuthentication:
- TestIngressRequestAuthentication:

### pilot集成测试

    go test -p 1 -vet=off -v -count=1 -tags=integ ./tests/integration/pilot/... -timeout 30m \
    --istio.test.work_dir=/logs/artifacts --istio.test.tag=1.12-dev \
    --istio.test.pullpolicy=IfNotPresent --istio.test.skip TestCustomGateway \
    --istio.test.skip TestTproxy \
    --istio.test.skip TestTraffic \
    --istio.test.hub=harbor.huayun.org/huayun-kubernetes/istio-testing
    
pilot失败case
- TestCustomGateway
- TestTproxy
- TestTraffic

### helm集成测试

    go test -p 1 -vet=off -v -count=1 -tags=integ ./tests/integration/helm/... -timeout 30m \
    --istio.test.work_dir=/logs/artifacts --istio.test.tag=1.12-dev \
    --istio.test.pullpolicy=IfNotPresent \
    --istio.test.hub=harbor.huayun.org/huayun-kubernetes/istio-testing

### operator集成测试

    go test -p 1 -vet=off -v -count=1 -tags=integ ./tests/integration/operator/... -timeout 30m \
    --istio.test.work_dir=/logs/artifacts --istio.test.tag=1.12-dev \
    --istio.test.pullpolicy=IfNotPresent \
    --istio.test.hub=harbor.huayun.org/huayun-kubernetes/istio-testing

### 总结

| 组件  |case数量(total:nopass)|执行时间（m）|	备注|
|---    |-- |-- |-- |
|helm   |7:7    |6  |   |
|operator|3:0   | 6 |   |
|pilot  |60:3   |18 |   |		
|security|48:4   |	12  |	都和jwt相关|
|telemetry|	32:1|	30|	依赖谷歌的GCP项目，无法执行通过|
|总计	|150：15	|72	|    |

注意：执行完成后需要执行清理操作，防止残留

## 清理脚本

    for ns in default ingress-nginx metallb-system;
    do
    kubectl delete cm istio-ca-root-cert -n $ns;
    done
    
    for ns in 1- service- se- app- istio- gce-metadata default- stable- external- echo- test-ns canary;
    do
    for i in `kubectl get ns |grep $ns |awk '{print $1}'`;do kubectl delete all --all -n $i --force && kubectl delete cm -n $i istio-ca-root-cert & done;
    done
    
    for ns in 1- service- se- app- istio- gce-metadata default- stable- external- echo- test-ns canary;
    do
    for i in `kubectl get ns |grep $ns |awk '{print $1}'`;do kubectl delete namespace $i --force;done;
    done

## 参考
- 集成测试官方文档：https://github.com/istio/istio/tree/master/tests/integration
- metallb部署：https://metallb.universe.tf/installation/
- jwt配置：https://imroc.cc/istio/troubleshooting/istio-token-setup-failed-for-volume-istio-token/
