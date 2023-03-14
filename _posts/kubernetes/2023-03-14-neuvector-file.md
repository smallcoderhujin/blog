---
title: "Neuvector源码分析之 文件管理"
subtitle: ""
layout: post
author: "hujin"
header-style: text
tags:
  - neuvector
  - 文件管理
  - fanotify
  - inotify

---

## 背景

进程规则、文件规则、网络规则、dlp、waf都是在监控组下的功能。本次我们通过源码的方式来深入了解文件规则是如何实现的。
文件规则支持用户自定义关注的文件、目录，设置规则的学习或者保护模式，相应的如果容器访问到了指定的文件，且文件规则设置保护模式，则对文件的写会产生告警。


## 架构图

![neuvector file](/blog/img/neuvector_file0.png)

我们从几个维度来看neuvector的文件管理功能：

- 页面/接口用户下发的文件规则如何让每个节点的Agent服务感知的
- 文件规则如何和容器关联起来的，Agent中是如何处理的
- 对文件执行读写操作后Agent如何感知
- fanotify/inotify怎么处理文件的操作的
- 文件管理怎么体现学习模式下学习
- 文件操作告警信息怎么采集的，告警的规则是什么 
  
下面我们带着这几个问题，来看看源码，我们先看第一个问题。

## 源码

### 下发的文件规则如何到达每个节点的Agent服务
![neuvector file](/blog/img/neuvector_file1.png)

我们先看看界面新增文件规则的流程：

- 获取group对象，获取group中profile和rule两组数据
- profile是文件规则，rule是文件规则中应用规则
- 将新增数据和已有数据合并，然后保存到数据库


    func handlerFileMonitorConfig(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
        ...
        // Check if we can config the profile. Only need authorize group
        grp, err := cacher.GetGroupBrief(group, false, acc)
        if err != nil {
            restRespNotFoundLogAccessDenied(w, login, err)
            return
        }
    
        if grp.Kind != share.GroupKindContainer {
            // "nodes" : share.GroupKindNode
            log.WithFields(log.Fields{"group": group, "kind": grp.Kind}).Error("Get profile failed!")
            restRespError(w, http.StatusBadRequest, api.RESTErrObjectNotFound)
            return
        }
    
        var profChanged bool
        profConf, profRev := clusHelper.GetFileMonitorProfile(group)
        ruleConf, ruleRev := clusHelper.GetFileAccessRule(group)
    
        ...
        // validate add
        if config.AddFilters != nil {
            for _, filter := range config.AddFilters {
                path := filter.Filter
                filter.Filter = filepath.Clean(filter.Filter)
                if filter.Filter == "." || filter.Filter == "/" {
                    restRespErrorMessage(w, http.StatusBadRequest, api.RESTErrInvalidRequest,
                        fmt.Sprintf("Unsupported filter: %s[%s]", path, filter.Filter))
                    return
                }
    
                // append the "/" back
                if path[len(path)-1:] == "/" {
                    filter.Filter += "/"
                }
    
                base, regex, ok := parseFileFilter(filter.Filter)
                if !ok {
                    restRespErrorMessage(w, http.StatusBadRequest, api.RESTErrInvalidRequest,
                        fmt.Sprintf("Unsupported filter: %s", filter.Filter))
                    return
                }
    
                for i, cfilter := range profConf.Filters {
                    if cfilter.Filter == filter.Filter {
                        // conflict, delete predefined
                        if !cfilter.CustomerAdd {
                            profConf.Filters = append(profConf.Filters[:i], profConf.Filters[i+1:]...)
                            // replace the rule below
                            idx := utils.FilterIndexKey(cfilter.Path, cfilter.Regex)
                            delete(ruleConf.Filters, idx)
                            break
                        } else {
                            restRespErrorMessage(w, http.StatusBadRequest, api.RESTErrInvalidRequest,
                                fmt.Sprintf("duplicate filter: %s", filter.Filter))
                            return
                        }
                    }
                }
                flt := share.CLUSFileMonitorFilter{
                    Filter:      filter.Filter,
                    Path:        base,
                    Regex:       regex,
                    Recursive:   filter.Recursive,
                    CustomerAdd: true,
                }
                if fileAccessOptionSet.Contains(filter.Behavior) {
                    flt.Behavior = filter.Behavior
                } else {
                    restRespErrorMessage(w, http.StatusBadRequest, api.RESTErrInvalidRequest, "Invalid File access option")
                    return
                }
    
                profConf.Filters = append(profConf.Filters, flt)
                // add rule
                idx := utils.FilterIndexKey(flt.Path, flt.Regex)
                capps := make([]string, len(filter.Apps))
                for j, app := range filter.Apps {
                    capps[j] = app
                }
                frule := &share.CLUSFileAccessFilterRule{
                    Apps:        capps,
                    CreatedAt:   tm,
                    UpdatedAt:   tm,
                    Behavior:    flt.Behavior,
                    CustomerAdd: true,
                }
                ruleConf.Filters[idx] = frule
                profChanged = true
            }
        }
    
        ...
    
        if profChanged {
            // Write to cluster
            if err := clusHelper.PutFileMonitorProfile(group, profConf, profRev); err != nil {
                log.WithFields(log.Fields{"error": err}).Error("Write cluster fail")
                restRespError(w, http.StatusInternalServerError, api.RESTErrFailWriteCluster)
                return
            }
        }
        // Write access rule
        if err := clusHelper.PutFileAccessRule(group, ruleConf, ruleRev); err != nil {
            log.WithFields(log.Fields{"error": err}).Error("Write cluster fail")
            restRespError(w, http.StatusInternalServerError, api.RESTErrFailWriteCluster)
            return
        }
    
        ...

看看数据怎么保存的，这里有个小坑：
- 默认数据保存在/object/config/file_monitor/<group-name>下面
- 在保存数据前还有一个DuplicateNetworkKey操作，会将数据保存一份到/node/<node-id>/common/profile/file/<group-name>下面
- 后面那个保存位置的数据会被每个enforcer watch

PutFileMonitorProfile:

    func (m clusterHelper) PutFileMonitorProfile(name string, conf *share.CLUSFileMonitorProfile, rev uint64) error {
        key := share.CLUSFileMonitorKey(name)
        value, _ := json.Marshal(conf)
        m.DuplicateNetworkKey(key, value)
        return cluster.PutRev(key, value, rev)
    }

上面看了controller中数据保存流程，下面来看看enforcer中数据watch流程

和其他功能类似，肯定有个地方在watch这个数据变化，然后执行一些操作,这里先看file_monitor部分的逻辑（access_rule是类似的）
    
    func profileDerivedProc(nType cluster.ClusterNotifyType, key string, value []byte) {
        which := share.CLUSNetworkKey2Subject(key)
        value, _ = utils.UnzipDataIfValid(value)
        // log.WithFields(log.Fields{"key": key}).Debug("GRP:")
        switch which {
        case share.ProfileGroup:                     // group
            systemConfigGroup(nType, key, value)
        case share.ProfileProcess:                   // process
            profileConfigGroup(nType, key, value)
        case share.ProfileFileMonitor:               // file
            systemConfigFileMonitor(nType, key, value)  
        case share.ProfileFileAccess:                // fileAccess
            systemConfigFileAccessRule(nType, key, value)
        case share.ProfileScript:                    // script, 这个在数据库里没找到
            systemConfigScript(nType, key, value)
        default:
            log.WithFields(log.Fields{"derived": which}).Debug("Miss handler")
        }
    }

    func systemConfigFileMonitor(nType cluster.ClusterNotifyType, key string, value []byte) {
      switch nType {
      case cluster.ClusterNotifyAdd, cluster.ClusterNotifyModify:
          ...
          updateGroupProfileCache(nType, name, profile)   // 这里重点看下这个函数
      case cluster.ClusterNotifyDelete: // required no group member that means no belonged containers, either
      }
  }

### 文件规则和监控组、容器关联

用户规则创建后保存到数据库，enforcer watch到数据变化会更新本地内存数据

- 对比内存数据，不一致则更新
- targets是当前group中容器id列表
- 可以看到grpNotifyFile是当前节点所有容器id的集合

updateGroupProfileCache:

    func updateGroupProfileCache(nType cluster.ClusterNotifyType, name string, obj interface{}) bool {
        ...
        targets := utils.NewSet()
        switch obj.(type) {
        ...
        case share.CLUSFileMonitorProfile:
            file := obj.(share.CLUSFileMonitorProfile)
            if file.Mode != grpCache.file.Mode || len(grpCache.file.Filters) == 0 || reflect.DeepEqual(file.Filters, grpCache.file.Filters) == false {
                for i, _ := range file.Filters {
                    file.Filters[i].DerivedGroup = name // late filled-up to save kv storages
                }
                grpCache.file = &file
                targets = grpCache.members.Clone()
                if targets.Cardinality() > 0 {
                    fileUpdated = true
                }
            }
        ...
    
        if fileUpdated {
            grpNotifyFile = grpNotifyFile.Union(targets)
        }
    
        targets.Clear()
        targets = nil
        return true
    }

看到这里有点好奇，group和容器是怎么关联上的，也就是上面代码中grpCache.members在哪里维护的：

- 在enforcer中监听runtime事件时，会监听容器的创建事件，对应的回调函数有个groupWorkloadJoin
- 在groupWorkloadJoin中根据workload的learnedGroupName获取对应的系统组，然后加入
- 用户自定义组根据Criteria和domain来识别是否包含此workload

registerEventHandlers&groupWorkloadJoin:

    func registerEventHandlers() {
        ...
        evhdls.Register(EV_WORKLOAD_START, []eventHandlerFunc{
            groupWorkloadJoin,
            scanWorkloadAdd,
        })
        ...


    func groupWorkloadJoin(id string, param interface{}) {
        ...
        if cache, ok := groupCacheMap[wlc.learnedGroupName]; !ok || isDummyGroupCache(cache) {
            ...
        } else {
            if !cache.members.Contains(wl.ID) {
                wlc.groups.Add(wlc.learnedGroupName)
                cache.members.Add(wl.ID)
                memberUpdated = true
                log.WithFields(log.Fields{"group": wlc.learnedGroupName}).Debug("Join group")
            }
        }
    
        // Join user defined group
        for _, cache := range groupCacheMap {
            if cache.group.CfgType == share.Learned {
                continue
            }
    
            if share.IsGroupMember(cache.group, wlc.workload) {
                if !cache.members.Contains(wl.ID) {
                    wlc.groups.Add(cache.group.Name)
                    cache.members.Add(wl.ID)
                    memberUpdated = true
                    log.WithFields(log.Fields{"group": cache.group.Name}).Debug("Join group")
                }
        ...

到这里我们可以看到内存中的数据一直在实时更新，监控组和容器的关系也建立好了，文件规则也更新到了指定组对象了，如何使用这些数据呢，我们继续往下看

### 定时更新fanotify关注的容器文件

![neuvector file](/blog/img/neuvector_file2.png)

enforcer中创建了一个定时任务，每隔5s执行一次，遍历所有容器，将内存中的文件规则应用到的fanotify中

    func fileMemberChanges(members utils.Set) {
        log.WithFields(log.Fields{"count": members.Cardinality()}).Debug("GRP:")
        for cid := range members.Iter() {
            ...
            c, ok := gInfo.activeContainers[id]
            gInfoRUnlock()
            if ok {
                applyFileGroupProfile(c)
            } ...
    }

在applyFileGroupProfile中涉及三个主要流程：

- calculateFileGroupProfile： 根据用户创建的文件规则筛选出所有相关file，根据监控组模式设置file的mask属性
- 更新容器matchRules：将计算到的file/access规则添加到workload对象中（纯数据转换处理逻辑）
- StartWatch：将file添加到fanotify中


calculateFileGroupProfile就是获取内存中group的file和access规则，
getFileMonitorProfile会加载数据库中的数据，如果内存不存在指定的group信息时

    func calculateFileGroupProfile(id, svc string) (*share.CLUSFileMonitorProfile, *share.CLUSFileAccessRule, bool) {
        log.WithFields(log.Fields{"id": id, "svc": svc}).Debug("GRP: ")
    
        file := &share.CLUSFileMonitorProfile{
            Filters:    make([]share.CLUSFileMonitorFilter, 0),
            FiltersCRD: make([]share.CLUSFileMonitorFilter, 0),
        }
        ...
        for _, grpCache := range grpProfileCacheMap {
            if grpCache.members.Contains(id) {
                file.Filters = append(file.Filters, grpCache.file.Filters...)
                file.FiltersCRD = append(file.FiltersCRD, grpCache.file.FiltersCRD...)
                mergeFileAccessProfile(access, grpCache.access)
            }
        }
        grpCacheLock.Unlock()
    
        // log.WithFields(log.Fields{"filter": file.Filters}).Debug("GRP:")
        ok, svc_file := getFileMonitorProfile(svc)
        if !ok {
            log.WithFields(log.Fields{"id": id, "svc": svc}).Debug("GRP: no file profile")
            return nil, nil, false
        }
    
        // basic information
        file.Group = svc_file.Group
        file.Mode = svc_file.Mode
    
        // merge regular files
        file.Filters = append(file.Filters, svc_file.Filters...)
        file.Filters = mergeFileMonitorProfile(file.Filters)
        ...
        return file, access, true
    }

我们重点看下StartWatch方法


    func (w *FileWatch) StartWatch(id string, rootPid int, conf *FsmonConfig, capBlock, bNeuvectorSvc bool) {
        ...
        dirs, files := w.getCoreFile(id, rootPid, conf.Profile)
    
        w.fanotifier.SetMode(rootPid, access, perm, capBlock, bNeuvectorSvc)
    
        w.addCoreFile(id, dirs, files)
    
        w.fanotifier.StartMonitor(rootPid)
        ...

getCoreFile: 会根据用户下发的文件规则（path可能带*）会获取所有相关的目录和文件路径


    func (w *FileWatch) getCoreFile(cid string, pid int, profile *share.CLUSFileMonitorProfile) (map[string]*osutil.FileInfoExt, []*osutil.FileInfoExt) {
        dirList := make(map[string]*osutil.FileInfoExt)
        singleFiles := make([]*osutil.FileInfoExt, 0)
    
        // get files and dirs from all filters
        for _, filter := range profile.Filters {
            flt := &filterRegex{path: filterIndexKey(filter)}
            flt.regex, _ = regexp.Compile(fmt.Sprintf("^%s$", flt.path))
            bBlockAccess := filter.Behavior == share.FileAccessBehaviorBlock
            bUserAdded := filter.CustomerAdd
            if strings.Contains(filter.Path, "*") {
                subDirs := w.getSubDirList(pid, filter.Path, cid)
                for _, sub := range subDirs {
                    singles := w.getDirAndFileList(pid, sub, filter.Regex, cid, flt, filter.Recursive, bBlockAccess, bUserAdded, dirList)
                    singleFiles = append(singleFiles, singles...)
                }
            } else {
                singles := w.getDirAndFileList(pid, filter.Path, filter.Regex, cid, flt, filter.Recursive, bBlockAccess, bUserAdded, dirList)
                singleFiles = append(singleFiles, singles...)
            }
        }
    
        ...
        return dirList, singleFiles
    }


SetMode：内存中创建rootFd对象，需要注意一点如果监控组是保护模式会设置permControl为true，这个在后面处理文件事件会用到

addCoreFile

- 向fanotify注册需要关注的文件列表，以及设置文件mask，以addFile为例
- 这里如果监控path是包管理路径会额外再调用inotify来监控

addCoreFile&addFile:

    func (w *FileWatch) addCoreFile(cid string, dirList map[string]*osutil.FileInfoExt, singleFiles []*osutil.FileInfoExt) {
        // add files
        for _, finfo := range singleFiles {
            // need to move the cross link files to dirs
            di, ok := dirList[filepath.Dir(finfo.Path)]
            if ok && !isRunTimeAddedFile(finfo.Path) {
                finfo.Filter = di.Filter
                di.Children = append(di.Children, finfo)
            } else {
                finfo.ContainerId = cid
                w.addFile(finfo)
            }
        }
    
        // add directories
        ...
    } 


    func (w *FileWatch) addFile(finfo *osutil.FileInfoExt) {
        w.fanotifier.AddMonitorFile(finfo.Path, finfo.Filter, finfo.Protect, finfo.UserAdded, w.cbNotify, finfo)
        if _, path := global.SYS.ParseContainerFilePath(finfo.Path); packageFile.Contains(path) {
            w.inotifier.AddMonitorFile(finfo.Path, w.cbNotify, finfo)
        }
    }

fanotify的addFile

- path形如：/host/proc/17490/root/usr/bin，解析到容器的pid和操作的文件路径
- fn.roots[rootPid]得到容器的root fd
- 这里注意mask的取值逻辑：userAdded/protect基本都符合可以忽略，permControl表示监控组是保护模式，configPerm正常是true，可以理解成监控组保护模式下默认给文件设置mask是FAN_OPEN_PERM，其他情况都是FAN_OPEN，这个需要结合后面fanotify事件的处理流程一起看
- 这样就将所有需要监听的文件权限、路径、容器id等所有相关的元数据都处理好了，后面就是实际应用到fanotify中了

addFile

    func (fn *FaNotify) addFile(path string, filter interface{}, protect, isDir, userAdded bool, files map[string]interface{}, cb NotifyCallback, params interface{}) bool {
        ...
        rootPid, rPath, err := ParseMonitorPath(path)
        ...
        r, ok := fn.roots[rootPid]
        ...
    
        var mask uint64 = faMarkMask
        if userAdded || protect { // user-defined or protected: including access control
            if r.permControl { // protect mode
                if fn.configPerm { // system-wise : access control is available
                    mask |= FAN_OPEN_PERM
                } else {
                    mask |= FAN_OPEN
                }
            } else {
                mask |= FAN_OPEN
            }
        }
    
        var file *IFile
        if isDir {
            ...
    
        } else {
            if _, ok = r.paths[rPath]; ok {
                return false
            }
            file = &IFile{
                path:    path,
                mask:    mask,
                params:  params,
                cb:      cb,
                filter:  filter.(*filterRegex),
                protect: protect,         // access control
                learnt:  r.accessMonitor, // discover mode
                userAdd: userAdded,
            }
    
            r.paths[rPath] = file
        }
    ...

fa.StartMonitor真正将文件规则应用到fanotify

- 这里的Mark是调用fanotify，给出需要监控的文件路径、文件事件、权限
- addHostNetworkFilesCopiedFiles是将hostnetwork的容器中一些通用的文件（/etc/hosts /etc/resolv.conf）进行监控

StartMonitor:
    
    func (fn *FaNotify) StartMonitor(rootPid int) bool {
        ...
        r, ok := fn.roots[rootPid]
        ...
    
        ppath := fmt.Sprintf(procRootMountPoint, rootPid)
        for dir, mask := range r.dirMonitorMap {
            path := ppath + dir
            if err := fn.fa.Mark(faMarkAddFlags, mask, unix.AT_FDCWD, path); err != nil {
                log.WithFields(log.Fields{"path": path, "error": err}).Error("FMON:")
            } else {
                mLog.WithFields(log.Fields{"path": path, "mask": fmt.Sprintf("0x%08x", mask)}).Debug("FMON:")
            }
        }
    
        //
        fn.addHostNetworkFilesCopiedFiles(r)
        return ok
    }


到这里已经将用户下发的文件规则，关联到group，关联到具体的容器（workload）中，同时也将用户规则进行重计算，得到一个完整的文件集合，
将需要关注（监控）的文件以及事件等细节都告诉了fanotify和inotify，下面就看看实际watch到文件事件后如何处理了


### 文件操作的感知

![neuvector file](/blog/img/neuvector_file3.png)

上面直接调用fn.fa.Mark去告诉fanotify需要关注的文件，但是这个对象哪里来的？

- NewFaNotify: 初始化fanotify
- NewInotify: 初始化inotify
- MonitorFileEvents：监听来自fanotify和inotify的文件事件并处理
- fw.loop：这个后面我们讲 
  
NewFileWatcher:
    
    func NewFileWatcher(config *FileMonitorConfig) (*FileWatch, error) {
        ...
    
        n, err := NewFaNotify(config.EndChan, config.PidLookup, global.SYS)
        if err != nil {
            log.WithFields(log.Fields{"error": err}).Error("Open fanotify fail")
            return nil, err
        }
        ni, err := NewInotify()
        if err != nil {
            log.WithFields(log.Fields{"error": err}).Error("Open inotify fail")
            return nil, err
        }
    
        go n.MonitorFileEvents()
        go ni.MonitorFileEvents()
    
        fw := &FileWatch{
            aufs:       config.IsAufs,
            fanotifier: n,
            inotifier:  ni,
            fileEvents: make(map[string]*fileMod),
            groups:     make(map[int]*groupInfo),
            sendrpt:    config.SendReport,
            sendRule:   config.SendAccessRule,
            estRuleSrc: config.EstRule,
            walkerTask: config.WalkerTask,
        }
        go fw.loop()
        ...

文件事件处理：

- 在获取到文件事件后，获取事件中进程pid、容器的root fd、文件mask
- 这里如果文件fmask是FAN_OPEN_PERM，则perm是1。前面提过如果监控组是保护模式，则会设置对应文件的mask是FAN_OPEN_PERM
- resp：用来给fanotify回复的结果，计算方式见下文的流程图

handleEvents:

    func (fn *FaNotify) handleEvents() error {
        for {
            ev, err := fn.fa.GetEvent()
            ...
            pid := int(ev.Pid)
            fd := int(ev.File.Fd())
            fmask := uint64(ev.Mask)
            perm := (fmask & (FAN_OPEN_PERM | FAN_ACCESS_PERM)) > 0
            ...
            resp, mask, ifile, pInfo := fn.calculateResponse(pid, fd, fmask, perm)
            if perm {
                fn.fa.Response(ev, resp)
            }
            ev.File.Close()
            ...


![neuvector file](/blog/img/neuvector_file4.png)

- resp的值默认是true
- 如果文件规则是保护模式，且操作进程不在允许的应用中，则resp是false
- 将fmask转换成mask（一个操作会有多个事件，只有第一个事件中fmask是设置过的）

calculateResponse:

    func (fn *FaNotify) calculateResponse(pid, fd int, fmask uint64, perm bool) (bool, uint32, *IFile, *ProcInfo) {
        ...
    
        ifile, _, mask := fn.lookupFile(r, linkPath, pInfo)
        if ifile == nil {
            return true, mask, nil, nil
        }
    
        // log.WithFields(log.Fields{"protect": ifile.protect, "perm": perm, "path": linkPath, "ifile": ifile, "evMask": fmt.Sprintf("0x%08x", fmask)}).Debug("FMON:")
    
        // permition decision
        resp := true
        if ifile.protect { // always verify app for block-access
            resp = fn.lookupRule(r, ifile, pInfo, linkPath)
            // log.WithFields(log.Fields{"resp": resp}).Debug("FMON:")
        }
    
        if (fmask & FAN_MODIFY) > 0 {
            mask |= syscall.IN_MODIFY
            log.WithFields(log.Fields{"path": linkPath}).Info("FMON: modified")
        } else if (fmask & FAN_CLOSE_WRITE) > 0 {
            mask |= syscall.IN_CLOSE_WRITE
            log.WithFields(log.Fields{"path": linkPath}).Info("FMON: cls_wr")
        } else {
            mask |= syscall.IN_ACCESS
                log.WithFields(log.Fields{"path": linkPath}).Info("FMON: read")
            if fn.isFileException(false, linkPath, pInfo, mask) {
                resp = true
                mask &^= syscall.IN_ACCESS
            }
        }
    
        if perm && !resp {
            pInfo.Deny = true
            log.WithFields(log.Fields{"path": linkPath, "app": pInfo.Path}).Debug("FMON: denied")
        }
        return resp, mask, ifile, pInfo
    }


综合看一下

![neuvector file](/blog/img/neuvector_file5.png)

- 只有监控组是保护模式的时候perm才是true
- perm是true的时候才会给fanotify发送response，其他时候都不会发送
- 只有文件规则是保护模式的时候resp才是false，其他时候都是true
- 换句话说：只有监控组是保护模式且文件规则是保护模式的时候fanotify才会阻断文件操作，其他都是允许


### 规则学习 & 告警信息上报

![neuvector file](/blog/img/neuvector_file6.png)

前面看了文件事件处理流程，但是我们忽略了一个细节

- 这里有个change参数，用来判断文件是否被修改
- 当fmask是FAN_CLOSE_WRITE表示文件被修改
- 当文件规则是学习模式，或者保护模式下修改了文件 事件都需要report
- 文件被修改或者需要上报都需要调用回调函数

handleEvents:

    func (fn *FaNotify) handleEvents() error {
        for {
            ev, err := fn.fa.GetEvent()
            ...
            change := (fmask & FAN_CLOSE_WRITE) > 0
            // log.WithFields(log.Fields{"ifile": ifile, "pInfo": pInfo, "Resp": resp, "Change": change, "Perm": perm}).Debug("FMON:")
    
            var bReporting bool
            if ifile.learnt { // discover mode
                bReporting = ifile.userAdd // learn app for customer-added entry
            } else { // monitor or protect mode
                allowRead := resp && !change
                bReporting = (allowRead == false) // allowed app by block_access
            }
    
            if bReporting || change { // report changed file
                ifile.cb(ifile.path, mask, ifile.params, pInfo)
            }
        }
        return nil
    }

回调函数： 只是将事件更新或者保存到内存中，那事件在哪里处理的呢？

    
    func (w *FileWatch) cbNotify(filePath string, mask uint32, params interface{}, pInfo *ProcInfo) {
        //ignore the container remove event. they are too many
        if (mask&syscall.IN_IGNORED) != 0 || (mask&syscall.IN_UNMOUNT) != 0 {
            w.inotifier.RemoveMonitorFile(filePath)
            return
        }
    
        w.mux.Lock()
        defer w.mux.Unlock()
        if fm, ok := w.fileEvents[filePath]; ok {
            fm.mask |= mask
            fm.delay = 0
            fm.pInfo = append(fm.pInfo, pInfo)
        } else {
            pi := make([]*ProcInfo, 1)
            pi[0] = pInfo
            w.fileEvents[filePath] = &fileMod{
                mask:  mask,
                delay: 0,
                finfo: params.(*osutil.FileInfoExt),
                pInfo: pi,
            }
        }
    }


这里在enforcer启动时创建了两个定时任务，分别用来处理事件和学习规则用

- HandleWatchedFiles会根据路径类型（文件还是目录）调用对应的处理方法
- reportLearningRules:后面细讲

HandleWatchedFiles:

    func (w *FileWatch) HandleWatchedFiles() {
        events := make(map[string]*fileMod)
        w.mux.Lock()
        for filePath, fmod := range w.fileEvents {
            events[filePath] = fmod
            delete(w.fileEvents, filePath)
        }
        w.mux.Unlock()
    
        for fullPath, fmod := range events {
            pid, path := global.SYS.ParseContainerFilePath(fullPath)
            //to avoid false alarm of /etc/hosts and /etc/resolv.conf, check whether the container is still exist
            //these two files has attribute changed when the container leave
            //this maybe miss some events file changed right before container leave. But for these kind of event,
            //it is not useful if the container already leave
            //	log.WithFields(log.Fields{"pid": pid, "path": path, "pInfo": fmod.pInfo[0], "fInfo": fmod.finfo}).Debug("FMON:")
            //	if fmod.pInfo != nil {
            //		log.WithFields(log.Fields{"pInfo": fmod.pInfo[0]}).Debug("FMON:")
            //	}
            rootPath := global.SYS.ContainerProcFilePath(pid, "")
            if _, err := os.Stat(rootPath); err == nil && path != "" {
                var event uint32
                info, _ := os.Lstat(fullPath)
                if fmod.finfo.FileMode.IsDir() {
                    event = w.handleDirEvents(fmod, info, fullPath, path, pid)
                } else {
                    event = w.handleFileEvents(fmod, info, fullPath, pid)
                }
                if event != 0 {
                    w.learnFromEvents(pid, fmod, path, event)
                }
            }
        }
    }


这里我们看看文件事件处理逻辑

- 如果用户删除文件规则后，会更新enforcer内存数据，也就是将fileinfo删除，这里的info就是nil了，也就需要让fanotify知道不再需要关注这些文件了，执行RemoveMonitorFile方法
- 这里的event是文件操作类型参数，不为空时调用learnFromEvents

handleFileEvents:

    func (w *FileWatch) handleFileEvents(fmod *fileMod, info os.FileInfo, fullPath string, pid int) uint32 {
        var event uint32
        if info != nil {
            if info.Mode() != fmod.finfo.FileMode {
                //attribute is changed
                event = fileEventAttr
                fmod.finfo.FileMode = info.Mode()
            }
            // check the hash existing and match
            // skip directory new file event, report later
            hash, err := osutil.GetFileHash(fullPath)
            if err != nil && !osutil.HashZero(fmod.finfo.Hash) ||
                err == nil && hash != fmod.finfo.Hash ||
                fmod.finfo.Size != info.Size() {
                event |= fileEventModified
                fmod.finfo.Hash = hash
            } else if (fmod.mask & syscall.IN_ACCESS) > 0 {
                event |= fileEventAccessed
            }
            if (fmod.finfo.FileMode & os.ModeSymlink) != 0 {
                //handle symlink
                rpath, err := osutil.GetContainerRealFilePath(pid, fullPath)
                if err == nil && fmod.finfo.Link != rpath {
                    event |= fileEventSymModified
                }
            }
            if (fmod.mask & inodeChangeMask) > 0 {
                w.removeFile(fullPath)
                w.addFile(fmod.finfo)
            }
        } else {
            //file is removed
            event = fileEventRemoved
            w.fanotifier.RemoveMonitorFile(fullPath)
        }
        return event
    }

规则学习：
- 如果监控组是学习模式，且进程访问的文件path匹配到文件规则，会将进程path保存到监控组的learnRules中（这个数据后面还有一个定时任务来处理）
- 通过最后的判断条件可以看出：非文件访问或者非学习模式下才发告警

learnFromEvents:

    func (w *FileWatch) learnFromEvents(rootPid int, fmod *fileMod, path string, event uint32) {
        ...
        grp, ok := w.groups[rootPid]
        mode := grp.mode
        if mode == share.PolicyModeLearn {
            flt := fmod.finfo.Filter.(*filterRegex)
            if applyRules, ok := grp.applyRules[flt.path]; ok {
                learnRules, ok := grp.learnRules[flt.path]
                if !ok {
                    learnRules = utils.NewSet()
                }
                for _, pf := range fmod.pInfo {
                    // only use the process name/path as profile
                    if pf != nil && pf.Path != "" {
                        if !applyRules.Contains(pf.Path) && !learnRules.Contains(pf.Path) {
                            learnRules.Add(pf.Path)
                            log.WithFields(log.Fields{"rule": pf.Path, "filter": flt}).Debug("FMON:")
                        }
                    }
                }
                // for inotify, cannot learn
                if learnRules.Cardinality() > 0 {
                    grp.learnRules[flt.path] = learnRules
                }
            } else {
                log.WithFields(log.Fields{"path": path}).Debug("FMON: no access rules")
            }
        }
        w.mux.Unlock()
    
        if event != fileEventAccessed ||
            (mode == share.PolicyModeEnforce || mode == share.PolicyModeEvaluate) {
            w.sendMsg(fmod.finfo.ContainerId, path, event, fmod.pInfo, mode)
        }
    }


### 规则学习
这里有个注意点，学习模式下文件管理和进程管理工作模式有点区别：
- 进程管理会自动学习新的进程规则，并添加到数据库
- 文件管理的规则必须手动创建，只会自动学习应用并添加到对应的规则中（规则允许的应用属性）


在enforcer中还有一个定时任务，用来处理内存中学习到的文件应用数据

    func (w *FileWatch) reportLearningRules() {
        learnRules := make([]*share.CLUSFileAccessRuleReq, 0)
        w.mux.Lock()
        for _, grp := range w.groups {
            if len(grp.learnRules) > 0 {
                for flt, rule := range grp.learnRules {
                    for itr := range rule.Iter() {
                        prf := itr.(string)
                        rl := &share.CLUSFileAccessRuleReq{
                            GroupName: grp.profile.Group,
                            Filter:    flt,
                            Path:      prf,
                        }
                        learnRules = append(learnRules, rl)
                    }
                }
                grp.learnRules = make(map[string]utils.Set)
            }
        }
        w.mux.Unlock()
        if len(learnRules) > 0 {
            w.sendRule(learnRules)
        }
    }

可以看到通过grpc调用ReportFileAccessRule将学习到的应用发送给controller


    func sendLearnedFileAccessRule(rules []*share.CLUSFileAccessRuleReq) error {
        log.WithFields(log.Fields{"rules": len(rules)}).Debug("")
        client, err := getControllerServiceClient()
        if err != nil {
            log.WithFields(log.Fields{"error": err}).Error("Failed to find ctrl client")
            return fmt.Errorf("Fail to find controller client")
        }
    
        ctx, cancel := context.WithTimeout(context.Background(), time.Second*3)
        defer cancel()
    
        ruleArray := &share.CLUSFileAccessRuleArray{
            Rules: rules,
        }
    
        _, err = client.ReportFileAccessRule(ctx, ruleArray)
        if err != nil {
            log.WithFields(log.Fields{"error": err}).Debug("Fail to report file rule to controller")
            return fmt.Errorf("Fail to report file rule to controller")
        }
        return nil
    }

在controller中有个定时任务来处理学习到的文件规则，有兴趣的可以再看下代码，最终就是调用PutFileAccessRule保存到数据库


    func FileReportBkgSvc() {
        for {
            if len(chanFileRules) > 0 {
                if kv.IsImporting() {
                    for i := 0; i < len(chanFileRules); i++ {
                        <-chanFileRules
                    }
                } else {
                    if lock, _ := clusHelper.AcquireLock(share.CLUSLockPolicyKey, policyClusterLockWait); lock != nil {
                        for i := 0; i < len(chanFileRules) && i < 16; i++ {
                            rules := <-chanFileRules
                            updateFileMonitorProfile(rules)
                        }
                        clusHelper.ReleaseLock(lock)
                    }
                }
            } else {
                time.Sleep(time.Millisecond * 100) // yield
            }
        }
    }


## 总结

我们从几位维度去看待文件管理这个功能，了解了文件规则如何下发到enforcer，enforcer如何监控当前节点所有容器的文件操作，文件事件如何处理，
文件学习如何实现。最终呈现在我们面前的文件管理是这样的：


| 对象 | 模式  | | | | | |
| --- | --- | --- | --- | --- | --- | --- |
| 监控组|学习 |学习  |告警  |告警  |保护 |保护  |
|文件规则|告警|保护|告警|保护|告警|保护|
|读告警|否|否|否|否|否|否|
|写告警|是|是|是|是|是|否|
|阻断&告警|否|否|否|否|否|是|

进程管理和文件管理的代码流程基本一致，可以参考来看

以上是代码逻辑以及一些个人理解，有问题的地方可以及时微信联系我更正