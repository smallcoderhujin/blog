---
title: "Istio Bookinfo"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - istio
  - bookinfo

---


## 架构图
<img src="https://istio.io/latest/zh/docs/examples/bookinfo/noistio.svg">

在k8s中通过deployment安装一些微服务并通过service暴露出来，包括
- productpage: python编写，服务主界面，或者叫index页面，页面中会访问其他微服务
- reviews： java编写，书籍评价微服务
- details： ruby编写，书籍详情信息微服务
- ratings： nodejs编写，数据推荐指数微服务，rating service的后端pod有三个，分别对应三个version

## 部署

### 部署原始应用（deployment/service等）

    kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

查看安装结果

    [root@controller01 ~]# kubectl get svc
    NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
    details       ClusterIP   10.96.192.251   <none>        9080/TCP   87s
    kubernetes    ClusterIP   10.96.0.1       <none>        443/TCP    119d
    productpage   ClusterIP   10.96.93.53     <none>        9080/TCP   86s
    ratings       ClusterIP   10.96.137.195   <none>        9080/TCP   87s
    reviews       ClusterIP   10.96.146.191   <none>        9080/TCP   86s

    [root@controller01 ~]# kubectl get pods -owide
    NAME                              READY   STATUS    RESTARTS   AGE   IP               NODE           NOMINATED NODE   READINESS GATES
    details-v1-66b6955995-n4rwz       2/2     Running   0          92s   10.244.114.172   controller01   <none>           <none>
    productpage-v1-5d9b4c9849-mjrcf   2/2     Running   0          91s   10.244.114.141   controller01   <none>           <none>
    ratings-v1-fd78f799f-ckf66        2/2     Running   0          92s   10.244.114.187   controller01   <none>           <none>
    reviews-v1-6549ddccc5-gtstf       2/2     Running   0          92s   10.244.114.175   controller01   <none>           <none>
    reviews-v2-76c4865449-sz5dr       2/2     Running   0          92s   10.244.114.188   controller01   <none>           <none>
    reviews-v3-6b554c875-jdpn6        2/2     Running   0          92s   10.244.114.143   controller01   <none>           <none>


###  此时服务可以通过k8s原生的service访问成功，且rating service有三个后端pod，所以平均切换
修改productpage service类型，改成nodeport（之前是clusterip）

    kubectl edit svc productpage
    apiVersion: v1
    kind: Service
    metadata:
      labels:
        app: productpage
        service: productpage
      name: productpage
    spec:
      clusterIP: 112.96.53.21
      clusterIPs:
      - 112.96.53.21
      ports:
      - name: http
        nodePort: 30607
        port: 9080
        protocol: TCP
        targetPort: 9080
      selector:
        app: productpage
      type: NodePort


通过http://[管理IP]:[svc-nodeport]/productpage来访问productpage服务

> 上面演示的是原生k8s的service功能，通过管理IP和nodeport的方式访问productpage服务，下面我们将通过istio的方式来访问微服务

### 启用istio 注入功能

允许default namespace自动注入istio sidecar

    kubectl label namespace default istio-injection=enabled
    
重新部署所有微服务，注入istio sidecar

    kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
    kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

创建gateway

> 这里创建gateway，保证流量可以进入ingressgateway，并进行转发

    [root@controller01 ~]# kubectl apply -f istio-1.11.2/samples/bookinfo/networking/bookinfo-gateway.yaml 
    gateway.networking.istio.io/bookinfo-gateway created
    virtualservice.networking.istio.io/bookinfo created


    [root@controller01 ~]# kubectl get gateway
    NAME               AGE
    bookinfo-gateway   109s

获取Istio Ingress Gateway的外部端口：

> 注意这里的31460 nodeport，需要通过这个访问bookinfo入口服务
    
    [root@controller01 ~]# kubectl get svc -n istio-system |grep gateway
    istio-egressgateway    ClusterIP      10.96.188.75    <none>        80/TCP,443/TCP                                                               4m8s
    istio-ingressgateway   LoadBalancer   10.96.156.24    <pending>     15021:31717/TCP,80:31460/TCP,443:30052/TCP,31400:32767/TCP,15443:30069/TCP   4m8s

创建destination rules, 配置路由访问规则：

    kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml
    
    [root@nested-istio-node1 networking]# kubectl get destinationrule 
    NAME          HOST          AGE
    details       details       15h
    productpage   productpage   15h
    ratings       ratings       15h
    reviews       reviews       15h
    
> 此时bookinfo的virtualservice只是将流量转发到productpage这个destination v1上，其他destination只是创建，未被引用

> 此时访问页面，页面中review微服务还是会三个版本平均切换的

##  智能路由

### 根据版本路由
    
    kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml
    
    cat samples/bookinfo/networking/virtual-service-all-v1.yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: productpage
    spec:
      hosts:
      - productpage
      http:
      - route:
        - destination:
            host: productpage
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
    spec:
      hosts:
      - reviews
      http:
      - route:
        - destination:
            host: reviews
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: ratings
    spec:
      hosts:
      - ratings
      http:
      - route:
        - destination:
            host: ratings
            subset: v1
    ---
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: details
    spec:
      hosts:
      - details
      http:
      - route:
        - destination:
            host: details
            subset: v1
    ---
    

> 这里创建了几个service，并将流量都转发到各自的v1 destinationrule上（demo中只有rating有多版本），访问服务，发现不管怎么刷页面，都看不到星星，因为v1版本没星星


### 根据用户路由

在vs中定义转发规则，如果发现header中有个jason用户信息，转发到v2，否则转发到v1后端pod
    
    kubectl delete -f samples/bookinfo/networking/virtual-service-all-v1.yaml
    kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml
    
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
    spec:
      hosts:
        - reviews
      http:
      - match:
        - headers:
            end-user:
              exact: jason
        route:
        - destination:
            host: reviews
            subset: v2
      - route:
        - destination:
            host: reviews
            subset: v1

> 再次访问，用jason用户登录就能看到黑星星，而其它方式看到的页面都是无星星

### 故障注入

注入错误让jason用户有个7s的延迟:
    
    kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-delay.yaml
    
    cat samples/bookinfo/networking/virtual-service-reviews-test-delay.yaml
    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: reviews
    spec:
      hosts:
      - reviews
      http:
      - match:
        - headers:
            end-user:
              exact: jason
        fault:
          delay:
            percentage:
              value: 100.0
            fixedDelay: 7s
        route:
        - destination:
            host: reviews
            subset: v2
      - route:
        - destination:
            host: reviews
            subset: v3

> 使用jason用户访问页面会有7s延迟，其他用户则转发到v3版本的reviews



### 链路切换
把100%流量切到v1

    kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml

> 页面不论刷几遍，都没有星星

v1 v3各50%流量

    kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml

> 刷新页面, 一会有红星，一会没星，50%的概率

把100%流量切到v3

    kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml
    
> 刷新页面，我们看到的都是红心了

