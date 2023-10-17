---
title: "Neuvector源码之 合规性检测"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - neuvector
  - 合规性检测
  - compliance

---

## 背景
Neuvector安全基线支持 CIS Benchmark 标准，可对容器、镜像、Register、主机、kubernetes 进行安全标准检查，多维度展现容器资产的基线合规情况并帮助建立容器运行环境下的最佳基线配置，减少攻击面
NeuVector 的合规性审核包括 CIS 基线测试、自定义检查、机密审核以及 PCI、GDPR 和其他法规的行业标准模板扫描。本文将通过源码的方式分析合规性检测的具体实现

## 架构图
![neuvector bench](/blog/img/neuvector_bench.png)

从架构图中我们可以看到涉及两个模块，分别是:
- controller:负责提供API接口，保存业务数据
- agent（enforce）：具体执行合规性检测的模块




## 源码

首先在rest.go中查看bench相关的接口，我们主要看下：

	r.GET("/v1/bench/host/:id/docker", handlerDockerBench)
	r.POST("/v1/bench/host/:id/docker", handlerDockerBenchRun)
	r.GET("/v1/bench/host/:id/kubernetes", handlerKubeBench)
	r.POST("/v1/bench/host/:id/kubernetes", handlerKubeBenchRun)

这里我们通过docker合规性检测来了解具体的实现细节，在handlerDockerBenchRun方法中获取node id，并查询到这个node节点的agent信息，
方便后面的rpc调用

    func RunDockerBench(agentID string) error {
        client, err := findEnforcerServiceClient(agentID)
        if err != nil {
            return err
        }
    
        ctx, cancel := context.WithTimeout(context.Background(), defaultReqTimeout)
        defer cancel()
    
        _, err = client.RunDockerBench(ctx, &share.RPCVoid{})
        return err
    }

切换到agent的代码中，我们看到RunDockerBench方法实际做的事情是调用了RerunDocker方法，在RerunDocker方法中执行的逻辑非常简单，就是reset
host timer和container timer, 同时设置下数据库中当前节点bench任务的状态为scheduled
    
    func (b *Bench) RerunDocker() {
        log.Info("")
    
        if err := b.dockerCheckPrerequisites(); err != nil {
            log.WithFields(log.Fields{"error": err}).Error("Cannot run Docker CIS benchmark")
            b.logBenchFailure(benchPlatDocker, share.BenchStatusNotSupport)
            b.putBenchReport(Host.ID, share.BenchDockerHost, nil, share.BenchStatusNotSupport)
        } else {
            b.hostTimer.Reset(hostTimerStart)
            b.conTimer.Reset(containerTimerStart)
            b.putBenchReport(Host.ID, share.BenchDockerHost, nil, share.BenchStatusScheduled)
        }
    }

![neuvector bench](/blog/img/neuvector_bench2.png)
通过上图我们发现，在agent启动时会启动一个后台任务BenchLoop，用来定时指定容器、host、k8s平台、自定义的合规性检测

    func (b *Bench) BenchLoop() {
        var masterScript, workerScript, remediation string
        b.taskScanner = newTaskScanner(b, scanWorkerMax)
        //after the host bench, it will schedule a container bench automaticly even if no container
        for {
            select {
            case <-b.hostTimer.C:
                b.doDockerHostBench()
    
            case <-b.kubeTimer.C:
                ...
    
                b.doKubeBench(masterScript, workerScript, remediation)
            case <-b.conTimer.C:
                containers := b.cloneAllNewContainers()
                if Host.CapDockerBench {
                    b.doDockerContainerBench(containers)
                } else {
                    b.putBenchReport(Host.ID, share.BenchDockerContainer, nil, share.BenchStatusFinished)
                }
    
                ...
            case <-b.customConTimer.C:
                ...
    
                b.doContainerCustomCheck(wls)
            case <-b.customHostTimer.C:
                b.doHostCustomCheck()
            }
        }
    }

这里在reset container timer后，会立即触发定时任务的执行，也就是这里的doDockerContainerBench方法

    func (b *Bench) doDockerContainerBench(containers map[string]string) error {
        b.putBenchReport(Host.ID, share.BenchDockerContainer, nil, share.BenchStatusRunning)
        if out, err := b.runDockerContainerBench(containers); err != nil {
            b.logBenchFailure(benchPlatDocker, share.BenchStatusDockerContainerFail)
            b.putBenchReport(Host.ID, share.BenchDockerContainer, nil, share.BenchStatusDockerContainerFail)
            return err
        } else {
            log.Info("Running benchmark checks done")
    
            list := b.getBenchMsg(out)
            b.assignDockerBenchMeta(list)
    
            b.putBenchReport(Host.ID, share.BenchDockerContainer, list, share.BenchStatusFinished)
    
            // Going through each container, write report and log
            ...
        }
    }

这里我们重点关注几个方法，分别是
- runDockerContainerBench： 具体执行容器合规性检测，后面重点介绍
- getBenchMsg：将执行结果格式化
- assignDockerBenchMeta：执行结果格式化，主要是格式化合规项名称和profile
- putBenchReport：将格式化后的结果保存到数据库中，等待获取结果的api调用时从数据库中获取

下面我们看看runDockerContainerBench方法：

    func (b *Bench) runDockerContainerBench(containers map[string]string) ([]byte, error) {
        ...

        if err := b.replaceDockerDaemonCmdline(srcContainerBenchSh, dstContainerBenchSh, cs); err != nil {
            log.WithFields(log.Fields{"error": err}).Error("Replace container docker daemon cmdline error")
            return nil, fmt.Errorf("Replace container docker daemon cmdline error, error=%v", err)
        }
    
        args := []string{system.NSActRun, "-f", dstContainerBenchSh, "-m", global.SYS.GetMountNamespacePath(1)}
        var errb, outb bytes.Buffer
    
        log.WithFields(log.Fields{"args": args}).Debug("Running bench script")
        cmd := exec.Command(system.ExecNSTool, args...)
        cmd.SysProcAttr = &syscall.SysProcAttr{Setsid: true}
        cmd.Stdout = &outb
        cmd.Stderr = &errb
        b.childCmd = cmd
        err := cmd.Start()
        ...
        return out, nil
    }

该方法首先根据模板生成目标的cis检测脚本
- 模板位置：/usr/local/bin/container.tmpl，生成文件位置：/tmp/container.sh
- 根据模板生成脚本时传入当前节点所有容器的信息，脚本中会遍历$containers参数，执行检测任务

replaceDockerDaemonCmdline

    func (b *Bench) replaceDockerDaemonCmdline(srcPath, dstPath string, containers []string) error {
        dat, err := ioutil.ReadFile(srcPath)
        if err != nil {
            return err
        }
        f, err := os.Create(dstPath)
        if err != nil {
            return err
        }
        defer f.Close()
    
        //containers only apply to container.sh, no effect to host.sh, because no <<<Containers>>> in it
        var containerLines string
        if len(containers) > 0 {
            containerLines = "containers=\"\n" + strings.Join(containers, "\n") + "\"\n"
        } else {
            containerLines = "containers=\"\"\n"
        }
        r := DockerReplaceOpts{
            Replace_docker_daemon_opts: strings.Join(b.daemonOpts, " "),
            Replace_container_list:     containerLines,
        }
        t := template.New("bench")
        t.Delims("<<<", ">>>")
        t.Parse(string(dat))
    
        if err = t.Execute(f, r); err != nil {
            log.WithFields(log.Fields{"error": err}).Error("Executing template error")
            return err
        }
        return nil
    }


生成玩cis检测脚本后，会调用nstool命令在host上执行检测命令：

    nstools run -f host.sh -m  /proc/1/ns/mnt -n /proc/1/ns/net

> 这里因为脚本会遍历节点上所有容器，通过docker命令执行检测操作，我们并不需要在容器内部执行，所以也不一定使用nstool工具，可以直接执行shell脚本

到了这里已经完成了某个节点上容器的合规性检测任务，我们看看执行返回的信息：
    
    ...
    [WARN] 4.1 - Ensure that a user for the container has been created (Automated)
    [WARN]      * Running as root: k8s_busybox_busybox-hujin_default_7191a3da-4c51-467a-a004-db178d79e92a_1158
    
    [WARN] 5.1 - Ensure that, if applicable, an AppArmor Profile is enabled (Automated)
    [WARN]      * No AppArmorProfile Found: k8s_busybox_busybox-hujin_default_7191a3da-4c51-467a-a004-db178d79e92a_1158
    [PASS] 5.2 - Ensure that, if applicable, SELinux security options are set (Automated)
    [PASS] 5.3 - Ensure that Linux kernel capabilities are restricted within containers (Automated)
    [PASS] 5.4 - Ensure that privileged containers are not used (Automated)
    [PASS] 5.5 - Ensure sensitive host system directories are not mounted on containers (Automated)
    [PASS] 5.6 - Ensure sshd is not run within containers (Automated)
    [PASS] 5.7 - Ensure privileged ports are not mapped within containers (Automated)
    [PASS] 5.8 - Ensure that only needed ports are open on the container (Manual)
    [PASS] 5.9 - Ensure that the host's network namespace is not shared (Automated)
    ...

> 容器的合规性检测功能，neuvector是集成了docker-bench-security项目，将该项目的检测脚本整理成了一个container.tmpl模板

后面会将执行结果进行格式化并保存到数据库，代码流程就讲了

这里还需要稍微提一下法规和cis的关系，我们在获取合规性检测结果的时候，handlerDockerBench - getCISReportFromCluster - _getCISReportFromCluster - 
bench2REST - GetComplianceMeta方法会将合规项添加法规对应的tag标识，方便进行过滤

    func GetComplianceMeta() ([]api.RESTBenchMeta, map[string]api.RESTBenchMeta) {
        if complianceMetas == nil || complianceMetaMap == nil {
            ...
            for _, item := range docker_image_cis_items {
                all = append(all, api.RESTBenchMeta{RESTBenchCheck: item})
            }
    
            for i, _ := range all {
                item := &all[i]
                item.Tags = make([]string, 0)
                if compliancePCI.Contains(item.TestNum) {
                    item.Tags = append(item.Tags, api.ComplianceTemplatePCI)
                }
                if complianceGDPR.Contains(item.TestNum) {
                    item.Tags = append(item.Tags, api.ComplianceTemplateGDPR)
                }
                if complianceHIPAA.Contains(item.TestNum) {
                    item.Tags = append(item.Tags, api.ComplianceTemplateHIPAA)
                }
                if complianceNIST.Contains(item.TestNum) {
                    item.Tags = append(item.Tags, api.ComplianceTemplateNIST)
                }
                ...
    }

支持的法规包括：
- PCI
- GDPR
- HIPAA
- NIST


## 总结
从上面的代码分析，我们看到neuvector支持对host/container/kubernetes平台的合规性检测，同时支持一些常见的法规，可以输出非常清晰的格式化结果并提供下载；
但我们也可以发现一些不足的地方，包括不支持其他runtime、不支持国内的法规等问题。
neuvector实际通过集成一个叫kubernetes-cis-benchmark的项目来实现基线扫描功能，此项目目前基本不维护了，导致支持的k8s版本一直停留在v1.18,且扫描出来的问题有些是不对的。
建议还是使用kube-bench来扫描，或者neuvector可以将kube-bench集成进来