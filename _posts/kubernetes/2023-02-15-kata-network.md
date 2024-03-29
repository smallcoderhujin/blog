---
title: "Kata容器网络实现"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - kata

---

## 背景
最近看到安全容器相关的文章，想着看看kata的网络实现

## 架构图
![kata调用流程](/blog/img/kata1.png)
在k8s中配置使用containerd作为runtime-endpoint实现，在containerd中配置runtime支持runc和kata



## 实现
![kata网络](/blog/img/kata2.png)

- 默认情况下，containerd容器在创建sandbox的时候，创建对应的netns出来
- 在创建容器时，cni负责创建和配置容器网卡，也就是在对应的netns中创建和配置容器网卡
- 在创建kata容器时，kata-runtime会在容器中创建一个qemu虚拟机，使用tap0_kata网卡作为虚拟机的虚拟网卡
- 这里kata-runtime支持多种方式将流量从容器原本的veth设备mirror到虚拟机的tap设备上
- 支持的mirror方式有: macvtap none和tcfilter，默认使用tcfilter


## 源码

containerd代码：创建netns

    // RunPodSandbox creates and starts a pod-level sandbox. Runtimes should ensure
    // the sandbox is in ready state.
    func (c *criService) RunPodSandbox(ctx context.Context, r *runtime.RunPodSandboxRequest) (_ *runtime.RunPodSandboxResponse, retErr error) {
        config := r.GetConfig()
        log.G(ctx).Debugf("Sandbox config %+v", config)
        ...
        if _, err := c.client.SandboxStore().Create(ctx, sandboxInfo); err != nil {
		    return nil, fmt.Errorf("failed to save sandbox metadata: %w", err)
        }
        ...
        if podNetwork {
            netStart := time.Now()
            // If it is not in host network namespace then create a namespace and set the sandbox
            // handle. NetNSPath in sandbox metadata and NetNS is non empty only for non host network
            // namespaces. If the pod is in host network namespace then both are empty and should not
            // be used.
            var netnsMountDir = "/var/run/netns"
            if c.config.NetNSMountsUnderStateDir {
                netnsMountDir = filepath.Join(c.config.StateDir, "netns")
            }
            sandbox.NetNS, err = netns.NewNetNS(netnsMountDir)
            if err != nil {
                return nil, fmt.Errorf("failed to create network namespace for sandbox %q: %w", id, err)
            }
            // Update network namespace in the store, which is used to generate the container's spec
            sandbox.NetNSPath = sandbox.NetNS.GetPath()
           ...


kata-runtime代码：创建qemu虚拟机并attach网卡

可以看到在创建完虚拟机后执行了AddEndpoints操作，也就是attach了网卡

    // startVM starts the VM.
    func (s *Sandbox) startVM(ctx context.Context, prestartHookFunc func(context.Context) error) (err error) {
        span, ctx := katatrace.Trace(ctx, s.Logger(), "startVM", sandboxTracingTags, map[string]string{"sandbox_id": s.id})
        defer span.End()
    
        s.Logger().Info("Starting VM")
    
        ...
    
        if err := s.network.Run(ctx, func() error {
            if s.factory != nil {
                vm, err := s.factory.GetVM(ctx, VMConfig{
                    HypervisorType:   s.config.HypervisorType,
                    HypervisorConfig: s.config.HypervisorConfig,
                    AgentConfig:      s.config.AgentConfig,
                })
                if err != nil {
                    return err
                }
    
                return vm.assignSandbox(s)
            }
    
            return s.hypervisor.StartVM(ctx, VmStartTimeout)
        }); err != nil {
            return err
        }
    
        ...
    
        // 1. Do not scan the netns if we want no network for the vmm.
        // 2. In case of vm factory, scan the netns to hotplug interfaces after vm is started.
        // 3. In case of prestartHookFunc, network config might have been changed. We need to
        //    rescan and handle the change.
        if !s.config.NetworkConfig.DisableNewNetwork && (s.factory != nil || prestartHookFunc != nil) {
            if _, err := s.network.AddEndpoints(ctx, s, nil, true); err != nil {
                return err
            }
        }
    
        s.Logger().Info("VM started")
    
        ...
        

kata-runtime代码：attach网卡流程

会在指定的network namespace中执行addSingleEndpoint方法

    // Add adds all needed interfaces inside the network namespace.
    func (n *LinuxNetwork) AddEndpoints(ctx context.Context, s *Sandbox, endpointsInfo []NetworkInfo, hotplug bool) ([]Endpoint, error) {
        span, ctx := n.trace(ctx, "AddEndpoints")
        katatrace.AddTags(span, "type", n.interworkingModel.GetModel())
        defer span.End()
    
        if endpointsInfo == nil {
            if err := n.addAllEndpoints(ctx, s, hotplug); err != nil {
                return nil, err
            }
        } else {
            for _, ep := range endpointsInfo {
                if err := doNetNS(n.netNSPath, func(_ ns.NetNS) error {
                    if _, err := n.addSingleEndpoint(ctx, s, ep, hotplug); err != nil {
                        n.eps = nil
                        return err
                    }
    
                    return nil
                }); err != nil {
                    return nil, err
                }
            }
        }

addSingleEndpoint

> 支持热挂载网卡

> 支持多种网卡类型，这里默认使用tuntap

> 网卡支持限速

> 支持多网卡（看到有单独的attach_interface接口）

    func (n *LinuxNetwork) addSingleEndpoint(ctx context.Context, s *Sandbox, netInfo NetworkInfo, hotplug bool) (Endpoint, error) {
        ...
            if socketPath != "" {
                networkLogger().WithField("interface", netInfo.Iface.Name).Info("VhostUser network interface found")
                endpoint, err = createVhostUserEndpoint(netInfo, socketPath)
            } else if netInfo.Iface.Type == "macvlan" {
                networkLogger().Infof("macvlan interface found")
                endpoint, err = createMacvlanNetworkEndpoint(idx, netInfo.Iface.Name, n.interworkingModel)
            } else if netInfo.Iface.Type == "macvtap" {
                networkLogger().Infof("macvtap interface found")
                endpoint, err = createMacvtapNetworkEndpoint(netInfo)
            } else if netInfo.Iface.Type == "tap" {
                networkLogger().Info("tap interface found")
                endpoint, err = createTapNetworkEndpoint(idx, netInfo.Iface.Name)
            } else if netInfo.Iface.Type == "tuntap" {
                if netInfo.Link != nil {
                    switch netInfo.Link.(*netlink.Tuntap).Mode {
                    case 0:
                        // mount /sys/class/net to get links
                        return nil, fmt.Errorf("Network device mode not determined correctly. Mount sysfs in caller")
                    case 1:
                        return nil, fmt.Errorf("tun networking device not yet supported")
                    case 2:
                        networkLogger().Info("tuntap tap interface found")
                        endpoint, err = createTuntapNetworkEndpoint(idx, netInfo.Iface.Name, netInfo.Iface.HardwareAddr, n.interworkingModel)
                    default:
                        return nil, fmt.Errorf("tuntap network %v mode unsupported", netInfo.Link.(*netlink.Tuntap).Mode)
                    }
                }
            } else if netInfo.Iface.Type == "veth" {
                networkLogger().Info("veth interface found")
                endpoint, err = createVethNetworkEndpoint(idx, netInfo.Iface.Name, n.interworkingModel)
            } else if netInfo.Iface.Type == "ipvlan" {
                networkLogger().Info("ipvlan interface found")
                endpoint, err = createIPVlanNetworkEndpoint(idx, netInfo.Iface.Name)
            } else {
                return nil, fmt.Errorf("Unsupported network interface: %s", netInfo.Iface.Type)
            }
        }
        ...
        networkLogger().WithField("endpoint-type", endpoint.Type()).WithField("hotplug", hotplug).Info("Attaching endpoint")
        if hotplug {
            if err := endpoint.HotAttach(ctx, s.hypervisor); err != nil {
                return nil, err
            }
        } else {
            if err := endpoint.Attach(ctx, s); err != nil {
                return nil, err
            }
        }
        ...

kata-runtime代码：引流实现

创建网卡, 这里第一张网卡名称是tap0_kata，虚拟机内部是eth0

    func createTuntapNetworkEndpoint(idx int, ifName string, hwName net.HardwareAddr, internetworkingModel NetInterworkingModel) (*TuntapEndpoint, error) {
        ...
    
        netPair, err := createNetworkInterfacePair(idx, ifName, internetworkingModel)
        if err != nil {
            return nil, err
        }
    
        endpoint := &TuntapEndpoint{
            NetPair: netPair,
            TuntapInterface: TuntapInterface{
                Name: fmt.Sprintf("eth%d", idx),
                TAPIface: NetworkInterface{
                    Name:     fmt.Sprintf("tap%d_kata", idx),
                    HardAddr: fmt.Sprintf("%s", hwName), //nolint:gosimple
                },
            },
            EndpointType: TuntapEndpointType,
        }
        ...

在attach方法中会调用xConnectVMNetwork，这里面会实现容器网络的引流

    // Attach for tun/tap endpoint adds the tap interface to the hypervisor.
    func (endpoint *TuntapEndpoint) Attach(ctx context.Context, s *Sandbox) error {
        span, ctx := tuntapTrace(ctx, "Attach", endpoint)
        defer span.End()
    
        h := s.hypervisor
        if err := xConnectVMNetwork(ctx, endpoint, h); err != nil {
            networkLogger().WithError(err).Error("Error bridging virtual endpoint")
            return err
        }
    
        return h.AddDevice(ctx, endpoint, NetDev)
    }

引流驱动有macvtap 和tcfilter，其中默认使用tcfilter

    func setupTCFiltering(ctx context.Context, endpoint Endpoint, queues int, disableVhostNet bool) error {
        ...
        if err := addQdiscIngress(tapAttrs.Index); err != nil {
            return err
        }
    
        if err := addQdiscIngress(attrs.Index); err != nil {
            return err
        }
    
        if err := addRedirectTCFilter(attrs.Index, tapAttrs.Index); err != nil {
            return err
        }
    
        if err := addRedirectTCFilter(tapAttrs.Index, attrs.Index); err != nil {
            return err
        }
    ...
    
## demo

kata部署可以参考网上的文章，这里通过创建一个kata容器，查看容器内的流量转发

创建kata容器

    [root@node1 kata]# cat nginx-kata.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: my-nginx-kata
    spec:
      selector:
        matchLabels:
          run: my-nginx
      replicas: 1
      template:
        metadata:
          labels:
            run: my-nginx
        spec:
          runtimeClassName: kata
          containers:
          - name: my-nginx
            image: httpd:alpine
            ports:
            - containerPort: 80
          - name: my-redis
            image: redis


查看容器

    [root@node1 kata]# kubectl get pods -owide
    NAME                             READY   STATUS    RESTARTS   AGE   IP               NODE    NOMINATED NODE   READINESS GATES
    my-nginx-kata-8675cd7c89-wzgr2   2/2     Running   0          45s   10.244.166.150   node1   <none>           <none>

查看网络命名空间

> 看到在netns中有两张网卡，eth0和tap0_kata

> eth0网卡是veth类型，类似普通容器的网卡

> tap0_kata网卡是tap类型，启动qemu虚拟机时使用的虚拟网卡

    [root@node1 kata]# ip netns exec cni-d27eff58-b9c9-a258-3a1e-a34528d9796f ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host
           valid_lft forever preferred_lft forever
    2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN group default qlen 1000
        link/ipip 0.0.0.0 brd 0.0.0.0
    4: eth0@if29: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1430 qdisc noqueue state UP group default qlen 1000
        link/ether fe:68:1c:e3:47:da brd ff:ff:ff:ff:ff:ff link-netnsid 0
        inet 10.244.166.150/32 brd 10.244.166.150 scope global eth0
           valid_lft forever preferred_lft forever
        inet6 fe80::fc68:1cff:fee3:47da/64 scope link
           valid_lft forever preferred_lft forever
    5: tap0_kata: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1430 qdisc mq state UNKNOWN group default qlen 1000
        link/ether 76:c7:1b:ab:30:64 brd ff:ff:ff:ff:ff:ff
        inet6 fe80::74c7:1bff:feab:3064/64 scope link
           valid_lft forever preferred_lft forever

tc引流规则

> 对于eth0网卡的入口流量通过tc规则mirror到tap0_kata网卡上

> 对于tap0_kata的入口流量通过tc规则mirror到eth0网卡上

> 这样一个双向的流量通道就建立了

    [root@node1 kata]# ip netns exec cni-d27eff58-b9c9-a258-3a1e-a34528d9796f tc -s qdisc show dev eth0
    qdisc noqueue 0: root refcnt 2
     Sent 0 bytes 0 pkt (dropped 0, overlimits 0 requeues 0)
     backlog 0b 0p requeues 0
    qdisc ingress ffff: parent ffff:fff1 ----------------
     Sent 480 bytes 5 pkt (dropped 0, overlimits 0 requeues 0)
     backlog 0b 0p requeues 0
     
    [root@node1 kata]# ip netns exec cni-d27eff58-b9c9-a258-3a1e-a34528d9796f tc -s filter show dev eth0 ingress
    filter protocol all pref 49152 u32
    filter protocol all pref 49152 u32 fh 800: ht divisor 1
    filter protocol all pref 49152 u32 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw  (rule hit 5 success 5)
      match 00000000/00000000 at 0 (success 5 )
            action order 1: mirred (Egress Redirect to device tap0_kata) stolen
            index 1 ref 1 bind 1 installed 439 sec used 437 sec
            Action statistics:
            Sent 480 bytes 5 pkt (dropped 0, overlimits 0 requeues 0)
            backlog 0b 0p requeues 0
     
    [root@node1 kata]# ip netns exec cni-d27eff58-b9c9-a258-3a1e-a34528d9796f tc -s filter show dev tap0_kata ingress
    filter protocol all pref 49152 u32
    filter protocol all pref 49152 u32 fh 800: ht divisor 1
    filter protocol all pref 49152 u32 fh 800::800 order 2048 key ht 800 bkt 0 terminal flowid ??? not_in_hw  (rule hit 12 success 12)
      match 00000000/00000000 at 0 (success 12 )
            action order 1: mirred (Egress Redirect to device eth0) stolen
            index 2 ref 1 bind 1 installed 451 sec used 165 sec
            Action statistics:
            Sent 768 bytes 12 pkt (dropped 0, overlimits 0 requeues 0)
            backlog 0b 0p requeues 0

## 总结
- 从上面的代码分析，大致了解了kata的流量路径，类似在一个普通容器基础上启动了一个qemu进程
- 通过tc mirror机制，将容器veth网卡流量镜像到tap网卡（通过macvtap子接口方式也类似）
