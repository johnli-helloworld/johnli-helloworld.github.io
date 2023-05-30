---
layout:     post
title:      Filecoin旧版调度
subtitle:   old schedule
date:       2023-05-18
author:     xiaoli
header-img: img/post-bg-debug.png
catalog: true
tags:
    - BlockChain
    - Filecoin
---

# 扇区存储


## 概览
![图片描述](/filecoin-img/oldsched/tapd_45270576_base64_1631155653_8.jpg)
##扇区状态管理逻辑
Sector的状态管理基于[状态机](https://baike.baidu.com/item/%E7%8A%B6%E6%80%81%E6%9C%BA/6548513?fr=aladdin)。通用状态机的实现是通过go-statemachine实现。状态的存储通过go-statestore实现。在这些模块的基础上，storage-sealing实现了Sector的状态定义以及状态处理函数。

### 01 模块框架
sector的相关模块如下：
Miner|Lotus|
:-:|:-:
SectorState| extern/storage-sealing
StateGroup||
StateMachine|go-stateMachine|
StoredState|go-statestore|

Sector的状态管理基于状态机。Lotus实现了通用的状态机（statemachine），在go-statemachine包中。在StateMachine之上，进一步抽象出了StateGroup，管理多个状态机。StateMachine只是实现状态的转变，具体状态是通过go-statestore进行存储。在StateMachine之上，定义状态转变的规则以及状态对应的处理函数，这些就是在具体的业务。SectorState就是Lotus管理Sector的具体业务。在这些底层模块的基础上，Lotus的相关代码调用就比较简单。先从状态管理的底层模块，StateMachine讲起：


### 02 StateMachine

结构定义在go-statemachine包中，machine.go:
```
type StateMachine struct {
    planner  Planner
    eventsIn chan Event

    name      interface{}
    st        *statestore.StoredState
	. . .
}
```
其中，planner是抽象出来的状态机的状态转化函数。Planner接收Event，结合当前的状态user，确定下一步的处理。

 !!#ff0000 type Planner func(events []Event, user interface{}) (interface{}, uint64, error)!! 

StateMachine的核心是run函数，分成三部分：接收Event，状态处理，下一步的调用。其中状态处理是关键：
```
err := fsm.mutateUser(func(user interface{}) (err error) {
 nextStep, processed, err = fsm.planner(pendingEvents, user)
 ustate = user
 if xerrors.Is(err, ErrTerminated) {
 terminated = true
 return nil
 }
 return err
})
```
mutateUser就是查看当前存储的状态（StoredState），并执行planner函数，并将planner处理后的状态存储。planner函数返回下一步的处理，已经处理的Event 的个数，有可能的出错。run函数会启动另外一个go routine执行nextStep。

![图片描述](/filecoin-img/oldsched/tapd_45270576_base64_1631000325_40.jpg)


### 03 StoredState

StoredState是抽象的Key-Value的存储。相对比较简单，Get/Put接口，非常容易理解。

### 04 SectorState
storage-sealing实现了和Sector状态相关的业务逻辑。也就是状态的定义，状态的转换函数都是在这个包里实现。整个Sector的信息定义在storage-sealing/types.go中(需要持久化到store的信息)：
```
type SectorInfo struct {
    State        SectorState
    SectorNumber abi.SectorNumber
	. . .
    // PreCommit2
    CommD *cid.Cid
    CommR *cid.Cid
    Proof []byte
	. . .
}

```
SectorInfo包括了Sector状态，Precommit1/2的数据，Committing的数据等等。其中，SectorState描述了Sector的具体状态。Sector的所有的状态定义在sector_state.go文件中：
```
const (
    UndefinedSectorState SectorState = ""

    PreCommit1 SectorState = "PreCommit1" // do PreCommit1
    PreCommit2 SectorState = "PreCommit2" // do PreCommit2
	. . .
)

```
所有Sector的状态转换如下( !!#ff0000 一体机!! ):
```

				      UndefinedSectorState (start)
				       v                     |
				*&lt;- WaitDeals &lt;-&gt; AddPiece   |
				|   |   /--------------------/
				|   v   v
				*&lt;- Packing &lt;- incoming committed capacity
				|   |
				|   v
				*&lt;- PreCommit1 &lt;--&gt; SealPreCommit1Failed
				|   |       ^          ^^
				|   |       *----------++----\
				|   v       v          ||    |
				*&lt;- PreCommit2 --------++--&gt; SealPreCommit2Failed
				|   |                  ||
				|   v          /-------/|
				*   PreCommitting &lt;-----+---&gt; PreCommitFailed
				|   |                   |     ^
				|   v                   |     |
				*&lt;- WaitSeed -----------+-----/
				|   |||  ^              |
				|   |||  \--------*-----/
				|   |||           |
				|   vvv      v----+----&gt; ComputeProofFailed
				*&lt;- Committing    |
				|   |             |
				|   v             |
		        |   CleanCache    |
		        |   |             |
		        |   v             |
				*&lt;- FinalizeSector &lt;--&gt; FinalizeFailed
				|   |
				|   v
				*&lt;- SubmitCommit
				|   |
				|   v
				*&lt;- CommitWait &lt;-----+---&gt; CommitFailed
				|   |
				|   v
				*&lt;- Proving
				|
				v
				FailedUnrecoverable
```
## 扇区调度逻辑
### 01 模块框架：
![图片描述](/filecoin-img/oldsched/tapd_45270576_1631065823_66.jpg)

整幅图分为上下两个部分：上部分是Manager，下部分是Remote Worker。Manager中包括一个Local Worker。stores.Index是所有Sector存储的索引。Scheduler，上部分的中间，管理所有的Worker，并且调度Sector相关的存储

worker management APIs通过/rpc/v0的JsonRPC接口实现remote worker的管理。通过/remote的HTTP API实现存储的Fetch操作，简单的说，传输文件。specs-storage.Prover/Sealer/Storage是Manager暴露出来的接口，实现Sector的证明，封存和存储。

每个连接到Manager的Worker会和Manager同步它的信息()。Scheduler在接受到新的请求时，会针对请求(Task)的类型以及资源的需求，从当前Worker中挑选最合适的Worker进行请求的处理。

### 02 ARS集群角色划分
![图片描述](/filecoin-img/oldsched/tapd_45270576_base64_1631150415_94.jpg)
从存储角度来看关系是这样的
![图片描述](/filecoin-img/oldsched/tapd_45270576_base64_1631150996_43.jpg)