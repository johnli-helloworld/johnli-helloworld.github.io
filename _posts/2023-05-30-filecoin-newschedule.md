---
layout:     post
title:      Filecoin新版调度
subtitle:   new schedule
date:       2023-05-18
author:     xiaoli
header-img: img/post-bg-debug.jpg
catalog: true
tags:
    - BlockChain
    - Filecoin
---

# 新版调度

----

​​
![图片描述](/filecoin-img/tapd_45270576_base64_1635991846_16.jpg)

一些重点的函数介绍（概括）
- requestWindows
worker请求任务的window，默认是两个。通知到sched并以此分配任务。
- checkSession
主要是用于检测worker是否关闭或重启了，重启之后worker的session会改变，若session变了则退出协程
- workerCompactWindows
主要是将worker上已存在的activeWindows中的任务合并，使他可以释放window去接取新任务。
- trySched
将任务队列里的任务分配到合适的window中


任务执行阶段
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636101427_32.jpg)
- method： startProcessingTask（）
- path：sched_worker.go
上图中do precommit1：![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636080458_70.jpg)
我们以precommit1举例，`req.work()` 真正执行是manage中的`WorkerAction`：
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636080611_100.jpg)
- method: startWork()
- path: manage_calltracker.go
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636093423_96.jpg)
持久化**WorkState**信息到miner数据库并记录**callToWork**
callToWork：map结构 key：CallID， value：WorkID
首先我们看一下这个 **WorkID** 是什么：
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636091669_20.jpg)
以任务类型，扇区ID和一些参数生成的唯一ID。
`startWork`返回的是一个 func()，入参`storiface.CallID` 和 `error` 即`w.SealPreCommit1(ctx, sector, ticket, pieces)` 的执行结果，那么我们看下这个 **CallID** 如何生成的.
追踪`w.SealPreCommit(xxx)`执行:
- method： SealPreCommit1()
- path： worker_local.go
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636081477_66.jpg)
重点关注`l.asyncCall()`
- method: asyncCall()
- path: worker_local.go
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636081826_7.jpg)
代码中的 **ci** 即是`startworker()`的**CallID**由`sectorId`和`UUID`生成的唯一标识；
`onStart(ci, rt)`在数据库中记录sector的状态（注意是在worker的数据库记录而不是miner），state是**CallStarted**；
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636082358_48.jpg)
接着起了一个协程，去执行具体的**precommit1**的过程了，既然是协程，于是**ci**此时可以立即返回，开始执行`startWork（）`函数了：
协程中的`work(ctx, ci)` 执行的是真正的**precommit1**的过程了，这里不过多赘述；
接下来`work()`执行完毕，我们取得了**precommit1**的执行结果**res**
执行成功`onDone()`改变sector的state为**CallDone**并保存结果
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636084092_56.jpg)
`doReturn()` 利用一系列的反射执行ReturnSealPreCommit1
- method： ReturnSealPreCommit1()
- path: manage.go

![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636084567_68.jpg)
![图片描述](/filecoin-img/newsched/tapd_45270576_base64_1636084606_43.jpg)
这里有几个需要明确的结构：
1. results：map结构 key：WorkID，value：res
2. waitRes：map结构 key：WorkID，value：chan
那么这个函数主要是将我们获取的结果**res**写道results中，并且通过waitRes对应的chan来通知`waitWork()`我们已经将结果返回了。

那么我们已经执行完了`startWork()`，接着执行`waitRes()`, waitRes实际执行的是`waitWork`方法
- method： waitWork
- path： manage_calltracker.go
这个就是等着将results的结果返回













