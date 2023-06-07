---
layout:     post
title:      Cosmos
subtitle:   IBC
date:       2023-05-18
author:     xiaoli
header-img: img/post-bg-debug.png
catalog: true
tags:
    - BlockChain
    - Cosmos
---

# IBC

[toc]

## 一. 什么是IBC

IBC是链间通信协议的缩写（Inter-Blockchain Communication Protocol）。通过数据包交换在多个不同的区块链网络之间转移数据和状态信息。最初的用途更多是通过IBC协议实现跨链通证转移。

![img](/cosmos-img/ibc0.png)

IBC的目标是在两个独立的七层网络之间传递应用信息，所以需要链外的relay（[中继器](https://github.com/cosmos/relayer)）把数据包在链A和链B的网络之间做中继。链B收到链A的数据后必须能独立验证它所包含的证明信息，该证明代表了链A上的某个状态（及其对应操作）的真实性。为了让IBC协议能够工作，必须依赖基础的信任机制，要相信链A和链B里各自的共识算法，也要相信轻客户端验证，通过对区块头信息的验证，证明在区块链上曾经发生过的事情。



## 二. 如何实现IBC

### 1. 连接的生命周期

#### 1.1 建立连接

在两条链之间首先要建立“连接”，也就是彼此的初始信任关系；在连接建立的那一瞬间两条链要交换基础的信任数据（信任根）-- 对PoS网络来说就是两条链的验证人公钥集，信任根必须是可以由第三方独立验真的。

#### 1.2 保持连接

在整个连接期间要持续不断获得对方的新区块头，基于信任根和连续的区块头，可以从连接建立时对方的区块高度，连续验证后续任意高度区块头的正确性；这些区块头是验证IBC数据包的信任基础。

#### 1.3 断开连接

当出现分叉或安全事件的时候，要及时关闭连接；这可以通过链上治理或自动作弊检测来触发。

### 2. 数据包，回执和超时处理

#### 2.1 数据包

是由元数据（数据头）和不透明数据载荷（数据体）组成的报文。数据头包含类型、顺序号、源链ID、目标链ID、超时参数；数据体则包含需要区块链应用层来理解和处理的数据，也就是源链状态变化的证明。

#### 2.2 回执

是“反向”数据包，B链收到并处理完来自A链的数据包后，会给A链发一个对应的回执。

#### 2.3 超时处理

源链在发送数据包时，可以在数据头里指定一个由目标链上的区块高度或时间戳表示的超时参数；目标链对收到的超时数据包将不予处理，而源链如果在发送的数据包超时后还未收到回执，就会对数据包对应的链上状态做回滚操作。

### 3. 严格排序的消息传送

要想让整套系统工作，数据包的传递必须保持严格的全局排序：

- 共识算法确保链上交易的处理遵循单一精准排序。
- IBC协议确保关于链上交易处理状态的消息在跨链传递的过程中遵循单一精准排序。
- IBC用通道机制实现排序控制：每条链为每个连接都维持发送和接收两个通道，每个通道维持一个计数器，发送通道的计数器为流出消息生成顺序号，接收通道的计数器则用于校验流入消息的顺序号。
- 严格的排序保证是对全局状态一致性进行推导的前提条件。



### 4. 共识要求

IBC协议安全需要共识算法的最终性来防止双花，不同共识算法的最终性表现不一样：

- Tendermint和PBFT类共识算法满足即时最终性（最理想）
- 以太坊的Casper FFG共识算法提供快速最终性
- 比特币类共识算法（PoW, Tezos）提供概率最终性，需要应用层选择安全阈值



## 三. IBC 包含的基本元素

类似于TCP/IP协议定义了不同的计算机在传输信息是采用了IP地址（计算机 ID）、端口号（应用程序ID）、协议号（传输层标准）的结构来传输信息。

IBC协议里，类似于定位计算机的IP地址是channel ID，定位应用程序的端口是port ID，再加上客户端同步信息构成了标准化沟通信息的方式。简洁的协议减少了跨链通信带给链本身的负担，没有过多约束参与跨链通信的应用本身，更加灵活。

#### 1.1 [Client](https://github.com/cosmos/ibc/blob/master/spec/core/ics-002-client-semantics/README.md)

跨链双方需要在链上初始化一个对方链的轻客户端，这个Client实质上是另一个区块链的轻客户端，而且必须满足Cosmos规定的一套Client接口。

需要保证在本链上能够验证来自来源链的跨链交易是能够验证的，否则无法保证在本链上执行该交易的有效性和合法性。

- 轻客户端规定了一套通用的接口，不同类型的区块链通过实现该 Client来达到接入的效果。
- 现阶段支持 Tendermint Client和Solo Client，也就是同构链之间原生支持跨链。
- 不是使用Cosmos构建的区块链想要接入Cosmos Hub进行跨链的话，必须通过一个额外的“转接桥”

#### 1.2 [Connection](https://github.com/cosmos/ibc/blob/master/spec/core/ics-003-connection-semantics/README.md)

连接将两个对象封装`ConnectionEnd`在两个独立的区块链上。每个`ConnectionEnd`都与另一个区块链（交易对手区块链）的客户端相关联。连接握手负责验证每条链上的轻客户端对于各自的交易对手来说是正确的。连接一旦建立，负责促进 IBC 状态的所有跨链验证。一个连接可以与任意数量的通道相关联。

#### 1.3 [Channel](https://github.com/cosmos/ibc/blob/master/spec/core/ics-004-channel-and-packet-semantics/README.md)

“通道”抽象为跨链通信协议提供消息传递语义，分为三类：排序、一次性传递和模块许可。通道充当数据包在一条链上的模块和另一条链上的模块之间传递的管道，确保数据包只执行一次，按照发送的顺序传递（如果需要），并且只传递到相应的模块拥有目标链上通道的另一端。每个通道都与一个特定的连接相关联，并且一个连接可以有任意数量的关联通道，允许使用通用标识符并使用连接和轻客户端在所有通道上分摊标头验证的成本。

可以在两个 IBC 端口之间建立 IBC 通道。端口由单个模块独占。IBC 数据包通过通道发送。正如IP数据包包含目的IP地址、IP端口、源IP地址和源IP端口一样，IBC数据包包含目的端口ID、通道ID、源端口ID和通道ID。IBC 数据包使 IBC 能够正确地将数据包路由到目标模块，同时还允许接收数据包的模块知道发送方模块。

- 通道可以`ORDERED`使得来自发送模块的数据包必须由接收模块按照发送的顺序进行处理。
- 推荐，一个通道可以`UNORDERED`使来自发送模块的数据包按照它们到达的顺序被处理，这可能不是数据包被发送的顺序。

模块可以选择它们希望与之通信的通道。IBC 期望模块实现在通道握手期间调用的回调。这些回调可能会执行自定义通道初始化逻辑。如果返回错误，则通道握手失败。通过在回调中返回错误，模块可以以编程方式拒绝和接受通道。

采用了有序的模式。

Connection和Client一起负责跨链交易的“合法性”——包括跨链交易确实在目的链上发生，以及跨链交易只提交了一次。

Connection和Channel在跨链扮演的角色和能不同， Channel用来保证跨链交易的有序性，每笔交易按照 Sequence Number来进行发送。

Connection和Channel的建立过程都如上面所示，只是数据包的名称和内容会有不同，

- 建立Connection的时候发送的便是 ConnOpenInit请求
- 建立的Channel的时候便是ChanOpenInit 请求，之后的请求依次类推。

#### 1.4 [Packet](https://github.com/cosmos/ibc/blob/master/spec/core/ics-004-channel-and-packet-semantics/README.md)

module之间通过相互发送packets 利用IBC channels来通讯。
所有的IBC packets中包含了：

- 目标`portID`
- 目标`channelID`
- 源`portID`
- 源`channelID`

以上信息可让module知道某packet对应的sender module。

此外，IBC packets中还包含了：

- sequence : A sequence to optionally enforce ordering

- `TimeoutTimestamp`and `TimeoutHeight`

Modules相互发送IBC 数据包的 `Data []byte` 字段中的自定义应用程序数据。对于IBC handlers来说，Packet data是完全透明的。sender module 必须将其特定于应用程序的数据包信息编码到packets中的`Data` field。receiver module必须将该`Data []byte`数据字段解码回原始应用数据。

#### 1.5 Acknowledgements

当处理a packet时，modules还会写application-specific acknowledgements。

acknowledgements有2种方式：

1)  Synchronously on `OnRecvPacket`：若module一收到来自IBC module的packet，就对该packet进行处理时。

2) Asynchronously：若module收到packet后，会稍晚再对该packet进行处理。

与packet中的Data类似，acknowledgement data对于IBC来说也是透明的。IBC会将acknowledgement data当成简单的byte string []byte。receiver modules 必须 encode their acknowledgement 使得 sender module can decode it correctly。具体对acknowledgement 的encode方式可在channel handshake时沟通确定。



## 四. Cosmos IBC relayer

> relayer github地址：https://github.com/cosmos/relayer
>
> osmosis上关于中继的设置 https://docs.osmosis.zone/developing/relaying/relay.html#build-setup-hermes
>
> 快速入门使用方法：https://pkg.go.dev/github.com/cosmos/relayer#readme-quickstart-guide

`relayer`软件包包含一个基本的中继器实现，供希望在启用IBC的链组之间中继数据包/数据的用户使用；

简单的看下relayer的操作指南，这里介绍了如何启用cosmos和osmosis的中继, 如下：

> ```
> $ rly chains add cosmoshub osmosis
> ```

查看源码，想要中继的两条链必须在[chain-registry](https://github.com/cosmos/chain-registry) 仓库中：

![image-20220318121930359](/cosmos-img/image-20220318121930359.png)

![image-20220318122012667](/cosmos-img/image-20220318122012667.png)



简单的看下配置内容 [chain.json](https://github.com/cosmos/chain-registry/blob/master/osmosis/chain.json)    中继器通过为每个链配置的 RPC 端点连接到各自网络上的节点。确保 config.yaml 中两个链的 rpc-addr 字段都指向有效的 RPC 端点。

导入或创建新密钥供中继器在签署和中继交易时使用

> $ rly keys add cosmoshub-4 [key-name]  
>
> $ rly keys add osmosis-1 [key-name]  

....

最后，我们在路径上启动中继器。创建了两个链客户端。中继器将定期更新客户端并侦听要中继的 IBC 消息。

中继器支持以下功能：

- 创建 IBC 连接
- 创建 IBC 传输通道。
- 发起跨链转移
- 中继跨链转账交易、其确认和超时
- 从状态中继
- 从流事件中继
- 为 IBC 突破性升级发送 UpgradePlan 提案
- 在交易对手链为 IBC 重大更改执行升级后升级客户端
- 从 GitHub 存储库获取规范链和路径元数据以快速引导中继器实例

中继器当前无法：

- 使用用户选择的参数（例如UpgradePath）创建客户端
- 提交IBC客户解冻建议
- 监控并为客户提交不当行为
- 使用除Tendermint之外的IBC light客户端，例如Solo Machine(手机，笔记本电脑等设备)
- 连接到未实现/未启用IBC的链
- 使用不同的IBC实现连接到链（链未使用SDK的`x/ibc`模块）



## 五. IBC 是如何工作的

Hub 与 Zone 直接通信，而 Zone 与 Zone 之间通过 IBC 间接通信。当 Zone 对 Hub 建立起一个 IBC 连接，它可以自动访问其他连接到该 Hub 上的 Zone ，这意味着 Zone 无需与其他 Zone 连接，而仅仅连接到 Hub 上即可。

通过保持各种跨 Zone 代币的固定总额，Hub 可以预防双重支付问题（Double Spend）。这些代币可以通过一种被称为「币包」 的特殊 IBC 数据包而实现 Zone 之间的跨链转移。

当一个 Zone 通过 Hub 收到来自其他 Zone 的代币时，它只需要信任 Hub 以及代币来源的 Zone，而不需要信任网络中所有其它的 Zone 。

*让我们看个例子：*假设当前由两条区块链：Zone 1 以及 Zone 2 。现在如果我们想要从 Zone 1 上发送代币到 Zone 2 ，会发生什么呢？

![image-20220325141427286](/cosmos-img/image-20220325141427286.png)

要让数据包从 Zone 1 发送到 Zone 2 ，Zone 1 首先要向 Hub 发送一个指向 Zone 2 的数据包。

![image-20220325141455023](/cosmos-img/image-20220325141455023.png)

在此之后，Zone 2 必须验证关于 Zone 1 的证明是否真实。为此，Zone 2 要利用 Zone 1 存储在 Hub 中的区块头。我们前面提到过 Hub 帮助 Zone 同步记录其它每一个 Zone 的状态，而 Hub 是通过记录其它 Zone 的区块头实现这一功能的。

![image-20220325141511758](/cosmos-img/image-20220325141511758.png)



现在你可能会疑问：为什么 Cosmos 不直接利用 IBC 建立 Zone 与 Zone 之间的连接？为什么需要进行 Hub 和 Zone 的设计？事实上，随着接入到网络中 Zone 的数量上升，以直连方式实现通信会导致链路数量呈平方级上升。以 100 个 Zone 接入到网络中为例，如果各个 Zone 直接都要建立起 IBC 连接，则网络中需要有 4950 条通信链路！如此快速的增长显然会令网络不堪重负。

![image-20220325141610227](/cosmos-img/image-20220325141610227.png)

- 创世「Hub」: Cosmos Hub

如前所述，Hub 是连接不同 Zone 的组件，而 Cosmos Hub 正是 Cosmos 网络中第一个 Hub ，它通过 IBC 连接其它的 Zone。Cosmos 网络上构建的第一个区块链（或者说 Zone ）会应用该主 Hub 来与网络中其他的 Zone 进行交互。这就意味着 Cosmos Hub 必须具备足够高的安全性（即许多验证者）来保证使用它的 Zone 能安全地进行互操作。



## 五. IBC跨链转账流程

IBC是属于Cosmos-SDK中一个特殊的模块。之所以特殊，主要体现在IBC提供了区块链之间的跨链能力

如果A链上的Alice上需要发送10个ATOM代币到B链上的Bob上，会经过下面的四个步骤

- **Tracking**

A链上的IBC模块会不断的同步B链上的区块头信息，B链上的IBC同理。通过这种方式，双方能够实现跟踪对方区块链上的验证者集合的变化，本质上来说，就是A链、B链相互维护了一个对方的轻节点

- **Bonding**

当使用IBC初始化一笔跨链转账之后，A链上的10个ATOM事实上处于锁定的状态

![](20210218165540916.png)

- **Proof中继**

一份证明A链上已经锁定10ATOM的“证据”会被路由到B链上的IBC模块

![](20210218165554573.png)

- **Verify**

B链结合A链的轻节点信息，对这份“证据”验证通过之后，B链上会“铸造”10份ATOM Voucher（抵用券），这些Voucher可以进行后续的流通使用。当然这些Voucher也可以通过同样的跨链方式返回到A链，A链上的ATOM代币相应执行解锁的操作。

实际上，跨链交易只是两条链上的资产的所有权的交换，BTC 仍然在比特币区块链上，ETH在以太坊区块链上。BTC离开比特币区块链就失去了价值，而交易实际上把BTC作为资产的价值进行转移，在Cosmos的模式下，资产本身就能在链上转移。



## 六. IBC 跨链交易详解

> ibc-go官方实现：https://github.com/cosmos/ibc-go

IBC协议是Cosmos中最核心的接口协议，能够实现区块链间跨链消息的可信、可靠转发，并有效进行流量控制、多路复用等功能。

在Cosmos中，每个功能都是高度模块化的，每个功能通过加载不同的模块来实现，IBC也是如此。在IBC设计时，借鉴了传输层的TCP协议，也是希望成为区块链领域的“TCP协议”。不仅如此，在IBC的各个方面也能看到TCP的身影，首先我们来看IBC中的一些基本概念。Cosmos IBC采用了有连接的、可靠的跨链消息传输

接下来我们来看一下一次完整IBC协议的握手和通信流程

![](cosmos IBC.assets/20210218165617140.png)



### 1. 通信过程

> 以下代码截图自 cosmos/ibc-go

#### 1.1 握手连接

在轻客户端的基础上建立握手连接，握手连接基本上分别为三个部分。Connection和Channel的建立都如下：

● 启动跨链的用户向链A发起OpenInit请求，等待Relayer 接收到该请求。

● Relayer进行路由跨链消息包的工作，如果收到 OpenInit的请求，Relayer 会构造一个的OpenTry 的请求发送到链B上。

● 链B收到OpenTry请求之后，如果同意的话，会对该消息进行确认（生成OpenACK数据包，并按照之前的方式由 Relayer 转发给链A。

● 链A通过OpenACK数据包判断此次握手是否成功，如果成功，对此次握手发送最后的 OpenConfirm 数据包返回链B。如果握手失败，此次连接也就是建立失败了。

![image-20220317175843626](/cosmos-img/image-20220317175843626.png)



上面的步骤不仅是指Connection的建立过程，Channel的建立也是遵循同样的流程，只是数据包的名称和内容会有不同，像建立Connection的时候发送的便是 ConnOpenInit请求，建立的Channel的时候便是ChanOpenInit 请求，之后的请求依次类推。

需要说明的是，Connection和Channel在跨链扮演的角色和功能并不相同，按照Cosmos的设计，Connection和Client一起负责跨链交易的“合法性”——包括跨链交易确实在目的链上发生，以及跨链交易只提交了一次。而Channel用来保证跨链交易的有序性，每笔交易按照 Sequence Number来进行发送。

虽然在Cosmos设计中有提到可以实现无序的Channel，但是默认实现上是采用了有序的模式。如果按照TCP协议簇来类比的话，有序Channel和TCP类似，无序Channel类似于UDP，无序Channel按照UDP来讲的话，在某些不太关注跨链消息包顺序的场景下也是适用的。同时Cosmos设计中也考虑到Channel的消息发送能力，允许一条Connection上建立多个Channel，在不同的跨链应用场景中，可以使用不同的Channel发送消息，从而隔离不同业务。



#### 2.2 发送跨链数据包

该`sendPacket`函数被模块调用，以便将调用模块拥有的通道端的IBC数据包发送到交易对手链上的相应模块。

调用模块必须与调用一起以原子方式执行应用程序逻辑`sendPacket`。

IBC 处理程序按顺序执行以下步骤：

- 检查通道和连接是否打开以发送数据包
- 检查调用模块是否拥有发送端口
- 检查数据包元数据是否匹配通道和连接信息
- 检查指定的超时高度是否尚未在目标链上传递
- 增加与通道关联的发送序列计数器
- 存储对数据包数据和数据包超时的固定大小承诺

完成握手之后，应用层便可以在Channel上发送自己的数据了。Cosmos规定了发送跨链交易的一些必要字段，如下图：

![image-20220317204814457](/cosmos-img/image-20220317204814457.png)

其中Sequence和SourcePort字段都是承担其字面意思的功能，也是必须指定的字段，而TimeoutHeight和TimeoutTimestamp是Cosmos提供的一种超时机制。如果某个区块高度或者某个时间这笔跨链交易还没有完成的话，用户能够指定将这笔交易回退（比如是跨链转账的话，可以防止资金长时间冻结）。而Data字段是留给用户进行自定义，以应对可能的各种复杂的跨链场景。

IBC-GO实现：

发送交易 SendPack：

![image-20220401160734244](/cosmos-img/image-20220401160734244.png)

- 若发送链为源链，铸币链，此时会创建一个托管地址，即锁仓。![image-20220401174554827](/cosmos-img/image-20220401174554827.png)



中继器接收到交易包后构建需要发送到接收链的中继消息MsgRecvPacket{}，比之上面的Packet{}获取了相关的包证明PacketCommitment。

QueryTendermintProof使用给定的key执行ABCI查询，并返回查询的值、原编码的merkle证明以及包含状态根的Tendermint块的高度。执行查询所需的初始高度应该在客户端上下文中设置。为了获得正确的merkle证明，查询将在低于这个高度1的位置(在IAVL版本中)执行。不支持高度小于或等于2的证明查询。客户端上下文高度为0的查询将在可用的最新状态执行查询。

![image-20220401160621355](/cosmos-img/image-20220401160621355.png)

#### 2.3 接收数据包

该`recvPacket`函数由模块调用，以便接收在交易对手链上相应通道端发送的 IBC 数据包。

以原子方式与调用结合`recvPacket`，调用模块必须执行应用程序逻辑或将数据包排队以供将来执行。

IBC 处理程序按顺序执行以下步骤：

- 检查通道和连接是否打开以接收数据包
- 检查调用模块是否拥有接收端口
- 检查数据包元数据是否匹配通道和连接信息
- 检查数据包序列是否是通道端期望接收的下一个序列（对于有序通道）
- 检查超时高度是否尚未通过
- 检查输出链状态中数据包数据承诺的包含证明
- 设置存储路径以指示数据包已被接收（仅限无序通道）
- 增加与通道端关联的数据包接收序列（仅限有序通道）

![image-20220401170133638](/cosmos-img/image-20220401170133638.png)

构建凭证**voucher**

![image-20220401175835137](/cosmos-img/image-20220401175835137.png)

#### 2.4 发送acknowledgements

该`writeAcknowledgement`函数由模块调用，以写入处理发送链可以验证的 IBC 数据包产生的数据，一种“执行回执”或“RPC 调用响应”。调用模块必须与调用一起以原子方式执行应用程序逻辑`writeAcknowledgement`。这是一个异步确认，在收到数据包时不需要确定其内容，仅在处理完成时确定。在同步情况下，`writeAcknowledgement`可以在同一个事务中（原子地）调用`recvPacket`.

![image-20220401170114457](/cosmos-img/image-20220401170114457.png)

![image-20220401170552972](/cosmos-img/image-20220401170552972.png)

中继器获取Acknowledgement Packet后，组装了MsgAcknowledgement{}后中继。

![image-20220401171245974](/cosmos-img/image-20220401171245974.png)

#### 2.5 处理acknowledgements

该`acknowledgePacket`函数由模块调用以处理对先前由调用模块在通道上发送到交易对手链上的交易对手模块的数据包的确认。 `acknowledgePacket`还清理了数据包提交，由于数据包已被接收并采取行动，因此不再需要这样做。

接收到中继器的`MsgAcknowledgement{}` 进行处理

![image-20220401172038430](/cosmos-img/image-20220401172038430.png)



## 七. 异构跨链

而对于非Cosmos SDK开发的区块链（如已经存在的这些区块链），如果要与Cosmos体系中的链进行交互（即能与Hub连接），需要使用Peg Zone进行桥接，**所谓的Peg Zone就是使用Cosmos SDK开发的，既能接入Hub的，又能和原链进行交互的一条链**。如Eth，如果要接入Cosmos Hub，则需要专门使用Cosmos SDK开发一条起Peg Zone作用的新链。

![img](/cosmos-img/b8389b504fc2d562fa26801fbea7d9ea77c66c55.jpeg)

上图我们可以看出 PegZone 可以分为 5 个部分：

1. Smart Contract ：资产托管的角色，保管以太坊中的代币和 Cosmos 中的代币。主要提供了 lock、unlock、mint、burn 四个方法。

2. Witness ：是一个以太坊全节点，监听以太坊合约的 event ，并等待 100 个区块产生后，封装 WitnessTx 提交到 PegZone 中来证明在以太坊内状态更改。

3. PegZone ：PegZone 是基于 Tendermint 的区块链，负责维护用户的账户信息，允许用户之间资产的转移，并提供交易查询。

4. Signer ：使用 secp256k1 对交易进行签名，以便签名能够高效的被智能合约验证，对应于智能合约的校验者公钥集合。

5. Relayer ： 中继器负责交易转发。将所有 Signer 签名后的 SignTx 转发到 smart contract 中。

