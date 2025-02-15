---
layout:     post
title:      Solana共识
subtitle:   bsc-relayer 剖析
date:       2023-05-18
author:     xiaoli
header-img: img/post-bg-debug.png
catalog: true
tags:
    - BlockChain
    - Solana
---

[toc]

## solana 共识机制

![](/solana-img/2022-06-13-17-22-42.jpg)

solana 中有4类角色：

- 普通用户
- SOL质押者/矿工
- 验证节点
- 领导者节点(由验证节点产生)

### 什么是POH?

`POH`全称为`Proof of history`,它用 SHA 256 作为 Solana 的验证延迟函数（Verifiable Delay Function），不断的对自身进行`hash`,以此来证明链上时间的流逝。

其特点是具有不可逆性，只能单向计算。
![](/solana-img/2022-06-13-17-31-27.jpg)

可以向输入中添加其它数据，下图可以证明，插入的数据必发生在第三个哈希前。
![](/solana-img/2022-06-13-17-37-39.jpg)

下图可以证明插入数据的时间必发生在 A 和 C 之间，且由于对数据中包含了 A 的哈希，再加上签名。可
以保证这条交易信息一定发生在 A 之后。
![](/solana-img/2022-06-13-17-49-10.jpg)

验证的时候可以并行验证，所以速度会很快。每个记录的切片都可以在单独的内核上从头到尾验证，所
需时间为生成时间的 1/（内核数）。因此，具有 4000 个内核的现代 GPU 可以在 0.25 毫秒内验证一
秒。
![](/solana-img/2022-06-13-17-49-49.jpg)



### 为什么分布式网络需要可靠的时间戳

- 确保事务顺序的准确性：

时间对于分布式账本技术的意义不言而喻，任何账本都需要达到「有序」，人们不能花没有收到的钱，也不能花已经花了的钱。区块链技术本身必须在无需第三方的情况下，明确地对账本进行排序。虽然区块链中还有许多其他技术细节，但时间是至关重要的，没有时间与顺序，就没有区块链。

#### 区块链中的时间

区块链的设计者们深知这一点，并给时间戳留下了足够的冗余。在[比特币的block timestamp](https://en.bitcoin.it/wiki/Block_timestamp)中，如果时间戳大于前11个区块的中值时间并小于网络调整时间+2小时，就可以被接受为有效时间。这导致比特币区块的时间戳并不完全按顺序排列。[以太坊](https://github.com/ethereum/wiki/wiki/White-Paper#mining)则明确规定区块的时间戳需要大于前一个区块且小于2小时的偏移。

区块链上真正重要的不是时间戳，而是交易的顺序。在一个点对点的电子现金系统中，正确性基于以下两点：

- 你不能花你还没收到的钱
- 你不能花你已经花出去的钱

一般来说，经典分布式系统处理时钟有两种方式。消息由发件人加盖时间戳并签名。节点丢弃太旧或太新的消息。此计算基于时间戳和本地时钟之间的差异。第二种方法是每个状态转换在过期之前都有一个本地超时。例如，在 Tendermint 上，预提交状态有一秒的超时时间。下一个区块生产者可以尝试提议下一个区块，但网络中的所有节点会在预提交状态转换开始后等待至少 1 秒，然后再考虑新的提议。

在这两种方法中，提议者的时钟都不能被信任，并且每个节点都会采取预防措施并强制延迟共识状态机的进度，以确保提议者不会作弊。尽管这些延迟对于网络安全至关重要，但它们会导致更慢的阻塞时间。

比特币的难度调整迫使网络平均大约每 10 分钟产生一次区块。以太坊难度设置为平均每 15 秒产生一次区块。两个网络之间的差异可以通过冲突或叔叔的数量来衡量。出块时间越短，两个节点同时出块的可能性就越高，而 15 秒可能是 Nakamoto 风格链出块速度的下限。

### 同步

​	快速、可靠的同步是Solana能够实现如此高吞吐量的最大原因。传统的区块链在被称为区块的大块交易上进行同步。区块打包交易时消耗的时间，我们称之为`区块时间`，在区块打包完成前，只有在区块打包完成时，才能确定交易执行顺序，各个节点才能在同一个区块上生成同样的状态。

- **工作证明：**像一般的工作证明机制，生成区块的时间较长，像是比特币（约10分钟）、以太坊（约12秒），如果缩短出块时间，就会增大不同矿工在**同一时间出块**的概率。

- **权益证明**：在权益证明中，需要可靠的时间戳，才能确定区块的顺序。否则出块节点提前出块，验证节点无法确认是自己的时间戳慢还是说`块`提前发出了。或者说，出块节点延迟出块，过段时间再把区块给补上，验证节点无法确认是自身网络延迟或者节点延迟出块。

  > 一般来说，经典分布式系统处理时钟有两种方式。消息由发件人加盖时间戳并签名。节点丢弃太旧或太新的消息。此计算基于时间戳和本地时钟之间的差异。第二种方法是每个状态转换在过期之前都有一个本地超时。
  >
  > 例如，在 Tendermint 上，预提交状态有一秒的超时时间。下一个区块生产者可以尝试提议下一个区块，但网络中的所有节点会在预提交状态转换开始后至少等待 1 秒，然后再考虑新的提议。

  - 举个例子：在`COSMOS HUB`中，每个区块的产生时间约为6秒，在这段时间内，验证者必须等待提议者打包完成交易消息，然后验证者再对区块中的消息进行执行验证，生成下一个区块的前置状态。如果当前所轮到的提议者离线或超时，则由下一轮的提议者出块。在这个过程中，`COSMOS`在区块上标记了一个时间戳，验证者通过对比区块上的时间戳来确定下一个区块的提议者是否已经超时。由于时钟漂移和网络延迟的变化，时间戳只在一两个小时内准确。验证者之间的系统时间并不完全相同，为了解决这个问题，这些系统延长了块的时间，以提供合理的确定性。如果想要通过缩短出块时间来提高`TPS`，就会因为时间同步问题，导致产生很多分叉(超时时间变短了)。

  ![pos的时间问题.drawio](/solana-img/pos的时间问题.drawio-16545806858542.jpg)

 



Solana 采用了一种不同寻常的方法，利用历史证明或称之为`POH`，解决分布式系统的时间戳问题。`POH`能够证明已经流逝了`这么多的时间`。所以，所有添加到历史证明中的数据，肯定是在证明生成前发生的。于是`solana`节点能够知道区块的产生顺序，区块可以以任何顺序到达验证节点。`Solana`还将区块分解成更小的交易批次，称之为`条目`，**条目是在区块形成之前，实时流向验证节点的**。

![sync_leader_validate.drawio](/solana-img/sync_leader_validate.drawio-16541503387771.jpg)

来看以下两个例子

- 在`Filecoin`中，如果节点收到一个来自`未来`的区块，要么是挖出该区块的矿工提前出块，要么是该节点的系统时间比其它集群慢。此时`lotus`会将该区块丢弃，如果错误原因是后者，则节点将会与网络失去同步。产生这个问题的主要原因是`lotus`的时间与其它节点不同步，区块自身无法证明其时间有效性。
- 在`Filecoin`中，是先将消息打包进入区块内，再执行区块内的消息，链状态才发生改变。区块之间的产生间隔是`30秒`，前6秒用于等待上一个区块的网络传播，后面24秒用于打包交易并发布区块。在这`30秒`中，非出块节点把大部分时间都花在了等矿工出块上，验证新区块及更新节点状态花费的时间较少。`Solana`将区块内交易分解成多个小批次，边打包交易边传播，当区块打包完成时，验证节点刚好执行完区块中的交易。

![filecoin 出块过程.drawio](/solana-img\filecoin_出块过程.drawio.jpg)

#### 同步过程

条目被实时传输给验证者，就像领导者节点可以将一组有效的交易批量化为一个条目一样快。验证者在对其有效性进行投票之前很久就会处理这些条目。通过优化处理交易，**在收到最后一个条目和节点可以投票的时间之间实际上没有延迟**。在没有达成共识的情况下，一个节点只需回滚其状态。这种优化处理技术是在1981年引入的，称为[乐观并发控制](#乐观并发控制)。它可以应用于区块链架构，其中一个集群对代表完整账本的哈希值进行投票，直到某个区块高度。在Solana中，它是使用最后一个条目的PoH哈希值来实现的。

> Solana网络中的验证器不是将哈希链接到每个块上，而是在块内连续地散列哈希本身。

> ### 乐观并发控制
>
> **乐观并发控制**（又名“**乐观锁**”，Optimistic Concurrency Control，缩写“OCC”）是一种[并发控制](https://zh.m.wikipedia.org/wiki/并发控制)的方法。它假设多用户并发的[事务](https://zh.m.wikipedia.org/wiki/数据库事务)在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。在[提交](https://zh.m.wikipedia.org/w/index.php?title=提交_(SQL)&action=edit&redlink=1)数据更新之前，每个事务会先检查在该事务读取数据后，有没有其他事务又修改了该数据。如果其他事务有更新的话，正在提交的事务会进行[回滚](https://zh.m.wikipedia.org/wiki/回滚_(SQL))。
>
> 乐观并发控制多数用于数据竞争(data race)不大、冲突较少的环境中，这种环境中，偶尔回滚事务的成本会低于读取数据时锁定数据的成本，因此可以获得比其他[并发控制](https://zh.m.wikipedia.org/wiki/并发控制)方法更高的[吞吐量](https://zh.m.wikipedia.org/wiki/吞吐量)。

![区块流式验证过程抽象.drawio](/solana-img/区块流式验证过程抽象.drawio.jpg)

就好比我们用下载器下载一部电影，有的下载器只能等整部电影下载完成，才能观看；有的下载器支持边下边播，无需漫长的等待。

值得一提的是：历史证明不是一种共识机制，但它被用来提高Solana的权益证明共识的性能。

#### 利用POH同步的优势

- 相对于其它同步方法、利用POH同步带来了哪些优势？为什么？//TODO
- 传统的区块链在生成下一个区块时，全局状态会被锁定

-----

### 领导者轮换

> 在任何时候，一个集群都希望只有一个验证者产生区块。通过一次只有一个领导者，所有验证者都能够重放相同的账本副本。然而，一次只有一个领导者的缺点是，一个恶意的领导者能够审查投票和交易。由于审查无法与网络丢弃的数据包区分开来，集群不能简单地选出一个节点来无限期地担任领导者的角色。相反，集群通过轮换哪个节点来最大限度地减少恶意领导者的影响。

领导者时间表是在本地定期重新计算的

- 在一段时间内分配领导者时间表，这段时间称为`Epoch` 纪元。
- `Epoch`中有很多时段（slot），每个时段由一位领导者负责出块。
- 在纪元结束时重新计算下一纪元的领导者时间表

也就是说，验证者的出块顺序，在纪元开始前就已经安排好了。

![共识机制-Epoch组成.drawio](/solana-img/共识机制-Epoch组成.drawio.jpg)

#### 领导者时间表生成

1. 定期使用PoH tick高度（一个单调增加的计数器）来作为稳定的伪随机算法的种子。
2. 在这个高度上，对所有在集群配置的tick数内投过票的有质押的账户进行抽样。这个样本被称为活跃集。
3. 按质押权重对活跃集进行排序。
4. 使用随机种子来选择按质押加权的节点，以创建一个质押加权的排序。



### 分叉

验证节点按照时间表轮流担任出块节点，生成编码了状态的`POH`。集群可以通过合成领导者会产生的内容来容忍领导者的连接失败（能够接着生成不添加交易内容的POH）。

![共识机制-分叉.drawio](/solana-img/共识机制-分叉.drawio.jpg)

#### 流程

1. 交易由当前的领导者接收。
2. 领导者过滤出来有效交易。
3. 领导者执行有效的交易，更新其状态。
4. 领导将交易打包成基于其当前PoH插槽的条目
5. 领导者将条目传送给验证者节点（以签名的碎片形式）。
   1. PoH流包括ticks；空条目用来表示领导者的有效性和集群上的时间流逝。
   2. 一个领导者的数据流从完成PoH所需的tick条目开始，到上一个领导者结束的时段。
6. 验证者将条目重新传送给他们集合中的对等体和更多的下游节点。
7. 验证者验证交易并在其状态上执行。
8. 验证者计算出状态的哈希值。
9. 在特定的时间，即特定的PoH tick计数，验证者向领导者传送投票。
   1. 投票是该PoH tick计数时计算出的状态的哈希值的签名。
   2. 投票也是通过`gossip`传播的
10. 领导者执行投票，与其他交易一样，并将其广播到集群中。
11. 验证者观察他们的投票和集群中的所有投票。

![共识机制-出块流程.drawio](/solana-img/共识机制-出块流程.drawio.jpg)

#### 分叉产生

在对应于投票所投的POH的时刻位置，可能会出现分叉，因为下一个领导者可能在没有观察到最后一个投票时刻（POH）里的投票时，基于一个虚拟的`POH`生成下一个时段(slot)的账本。

在一个投票时段内，只会存在两种`POH`版本，一种是又当前领导者生成加入投票数据后的`POH`,一种是基于上一个`slot`，所生成的虚拟`POH`(不含其它内容，所有节点都能自己独立生成)。

验证节点可以忽略其它点的分叉（分叉来自错误的领导者等），或者削减掉错误的领导者。

验证者基于贪婪的选择进行投票，以最大化[`塔式拜占庭`](#Tower BFT)中的奖励。

##### 在验证者视角中的分叉

![共识机制-分叉-验证者视角.drawio](/solana-img/共识机制-分叉-验证者视角.drawio.jpg)

##### 在领导者视角中的分叉

当领导者开始一个时段(slot)时，它必须先生成将最近观察到的时段及投票的`POH`，联系起来的`POH`。

####  分叉管理

账本被允许在时段(slot)的边界分叉，由此产生的数据结构形成一棵树。在状态被最终确认前，验证器必须维持各分支的状态。验证者的责任就是权衡这些分叉，最终选出一个分叉。

验证者通过向该分叉的时段(slot)领导者提交投票来选择一个分叉。该投票使验证者在一定的时间锁定在此分叉。在锁定期结束之前，验证人不允许在不同的分叉上投票。在同一分叉上的每一次后续投票都会使锁定期的长度增加一倍。在同一分叉上的投票数达到32后，锁定期长度达到最长，验证者无法对其它分叉进行投票。如果投票数未达到32，验证者可以等待锁定期过去后，再对其它分叉进行投票。

当验证者对另外的分叉进行投票时，需要将状态回滚到其它分叉。验证者能回滚的深度与最大锁定期挂钩，无法回滚超过最大锁定期的深度，所以可以将超过最大深度的分叉裁剪掉。



#### 裁剪分叉

假设最大回滚深度为2。现有以下分叉：

![image-20220607155437906](/solana-img/image-20220607155437906.jpg)

对`5`进行投票，由于最大回滚深度是2，只能回滚到分叉点`2`。于是`2`以外的分叉被裁剪掉。

![image-20220607155614608](/solana-img/image-20220607155614608.jpg)





### 投票机制

- `验证者·通过签名一条交易消息，来对所选择的分叉进行投票。

- ·质押持有者·是一个对质押有控制权的身份，它们将质押委托给验证者。验证者所投的票代表所有被委托的质押的投票权重，并为所有质押用户产生奖励。

  #### 验证者投票

  一个验证者节点，在启动时，创建一个新的投票账户，并在集群中注册。集群中的其他节点将新的验证者列入活动集。随后，验证人在每个投票事件中提交一个用验证人的投票私钥签名的 "新投票 "交易。

  #### 投票和质押程序

  奖励过程被分成两个链上程序。`Vote`程序使质押可被削减。Stake程序作为奖励池的保管人，并提供被动接受委托功能。当显示出委托的验证者参与了验证分类账时，Stake程序负责向验证者和委托人支付奖励。

验证者进行投票是有激励的，同时验证者投票时会消耗一定的交易手续费。



