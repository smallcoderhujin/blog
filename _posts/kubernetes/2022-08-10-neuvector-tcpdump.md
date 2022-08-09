---
title: "Neuvector源码分析之 网络抓包"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - istio
  - bookinfo

---


## 功能介绍

![neuvector_pcap](/blog/img/neuvector_tcpdump.png)

抓包功能是针对容器的功能，用户在界面选择某个容器点击抓包功能，可以控制抓包开始和结束，可以选择抓包时间；完成后可以下载对应的pcap格式的文件，
在本地的wireshark中直接打开进行分析。底层实际还是通过进入容器的网络namespace，执行tcpdump命令来实现。

说明下：hostnetwork的容器暂不支持抓包功能

API接口：
    
    neuvector\controller\rest\rest.go:1517

    r.GET("/v1/sniffer", handlerSnifferList)
	r.GET("/v1/sniffer/:id", handlerSnifferShow)
	r.POST("/v1/sniffer", handlerSnifferStart)
	r.PATCH("/v1/sniffer/stop/:id", handlerSnifferStop)
	r.DELETE("/v1/sniffer/:id", handlerSnifferDelete)
	r.GET("/v1/sniffer/:id/pcap", handlerSnifferGetFile)

这里我们重点看下创建，也就是handlerSnifferStart的代码流程：

    func handlerSnifferStart(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
      ...
      # 从request中获取对应的参数，这里是workloadid，也就是对应的pause容器的container id
      query := restParseQuery(r)
    
      # 获取容器id参数，并根据容器id获取对应的agentid
      agentID, wlID, err := getAgentWorkloadFromFilter(query.filters, acc)
      if err != nil {
          restRespNotFoundLogAccessDenied(w, login, err)
          return
      }
    
      // Check if we can config workload
      wl, err := cacher.GetWorkloadBrief(wlID, "", acc)
      if wl == nil {
          restRespNotFoundLogAccessDenied(w, login, err)
          return
      } else if !acc.Authorize(&share.CLUSSnifferDummy{WorkloadDomain: wl.Domain}, nil) {
          restRespAccessDenied(w, login)
          return
      }
      ...
    
      args := proc.Sniffer
      req := &share.CLUSSnifferRequest{WorkloadID: wlID, Cmd: share.SnifferCmd_StartSniffer}
      ...
    
      res, err := rpc.SnifferCmd(agentID, req)
      ...
      restRespSuccess(w, r, &resp, acc, login, &proc, "Start sniffer")
    }

- 代码会从request中获取需要抓包的容器id
- getAgentWorkloadFromFilter中获取容器id并查询对应的agent id
- GetWorkloadBrief 获取指定容器的详细信息，并校验容器是否允许抓包
- SnifferCmd 通过grpc调用对应agent的抓包接口，这里会提前配置一些抓包的参数，包括文件名称、文件大小（默认2M）、抓包时间等等

我们在agent中查看对应的调用接口SnifferCmd，文件位置：neuvector\agent\service.go:830

    func (rs *RPCService) SnifferCmd(ctx context.Context, req *share.CLUSSnifferRequest) (*share.CLUSSnifferResponse, error) {
        if req.Cmd == share.SnifferCmd_StartSniffer {
            id, err := startSniffer(req)
            return &share.CLUSSnifferResponse{ID: id}, err
        } else if req.Cmd == share.SnifferCmd_StopSniffer {
            return &share.CLUSSnifferResponse{}, stopSniffer(req.ID)
        } else if req.Cmd == share.SnifferCmd_RemoveSniffer {
            return &share.CLUSSnifferResponse{}, removeSniffer(req.ID)
        }
        return &share.CLUSSnifferResponse{}, grpc.Errorf(codes.InvalidArgument, "Invalid sniffer command")
    }

继续查看对应的startSniffer方法：
    
    func startSniffer(info *share.CLUSSnifferRequest) (string, error) {
        var pid int
    
        gInfoRLock()
        c, ok := gInfo.activeContainers[info.WorkloadID]
        ...
    
        proc := &procInfo{
            workload:   info.WorkloadID,
            fileNumber: uint(info.FileNumber),
            duration:   uint(info.DurationInSecond),
        }
    
        key := generateSnifferID()
    
        proc.fileName, proc.args = parseArgs(info, key[:share.SnifferIdAgentField])
        _, err := startSnifferProc(key, proc, pid)
        if err != nil {
            return "", grpc.Errorf(codes.Internal, err.Error())
        } else {
            return key, nil
        }
    }

- 这里根据容器id或者内存中容器对象，这个对象实际是通过独立线程监听节点的runtime维护的信息
- generateSnifferID 这个是根据agent id生成一个id作为文件名称的一部分

    func parseArgs(info *share.CLUSSnifferRequest, keyname string) (string, []string) {
        ...
        filename = defaultPcapDir + keyname + "_"
        filenumber = fmt.Sprintf("%d", info.FileNumber)
        filesize = fmt.Sprintf("%d", info.FileSizeInMB)
        ...
    
        tcpdumpCmd := []string{"-i", "any", "-U", "-C"}
        cmdStr = append(tcpdumpCmd, filesize, "-w", filename, "-W", filenumber)
        ...
        return filename, cmdStr
    }

- parseArgs用来生成完整的文件名称，并准备具体的tcpdump命令，完整的命令类似： tcpdump -i any -U -C 2 -w /var/neuvector/pcap/0a5bdf2c_0

下面就是进入容器的network namespace然后执行tcpdump命令

    func startSnifferProc(key string, proc *procInfo, pid int) (string, error) {
        ...
    
        var script string
        if proc.duration > 0 {
            script = fmt.Sprintf("timeout %d ", proc.duration)
        }
        script += "tcpdump " + strings.Join(proc.args, " ")
        log.WithFields(log.Fields{"key": key, "cmd": script}).Debug()
    
        proc.cmd = exec.Command(system.ExecNSTool, system.NSActRun, "-i", "-n", global.SYS.GetNetNamespacePath(pid))
        proc.cmd.SysProcAttr = &syscall.SysProcAttr{Setsid: true}
        proc.cmd.Stderr = &proc.errb
        stdin, err := proc.cmd.StdinPipe()
        if err != nil {
            e := fmt.Errorf("Open nsrun stdin error")
            log.WithFields(log.Fields{"error": err}).Error(e)
            return "", e
        }
    
        err = proc.cmd.Start()
        if err != nil {
            e := fmt.Errorf("Failed to start sniffer")
            log.WithFields(log.Fields{"error": err}).Error(e)
            return "", e
        }
    
        pgid := proc.cmd.Process.Pid
        global.SYS.AddToolProcess(pgid, pid, "sniffer", script)
    
        io.WriteString(stdin, script)
        stdin.Close()
    
        ...
        return status, err
    }

- 这里就是通过nstool工具进入容器network namespace, 将tcpdump命令作为stdin在namespace中执行
- 完整的命令类似：echo "tcpdump -i any -U -C 2 -w /var/neuvector/pcap/xxx  -W 5" | ./nstools run -i -n /proc/2271/ns/net
- nstools这个工具类似nsenter，为了安全工具内部会校验调用方必须是neuvector agent服务，所以一般情况下执行这个命令是会失败的
- 监听tcpdump进程状态并返回状态信息

其他方法比如stop、下载抓包文件的调用路径是类似的