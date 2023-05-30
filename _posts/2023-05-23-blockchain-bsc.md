---
layout:     post
title:      bsc-relayer
subtitle:   bsc-relayer 剖析
date:       2023-05-18
author:     xiaoli
header-img: img/post-bg-debug.png
catalog: true
tags:
    - BlockChain
    - bsc
---

# bsc-relayer

[toc]

> github相关项目地址：
>
> *bsc-relayer*：[bsc-relayer](https://github.com/bnb-chain/bsc-relayer)
>
> *内置系统合约*：[bsc-genesis-contract](https://github.com/binance-chain/bsc-genesis-contract) ; 官方简介：https://docs.bnbchain.world/docs/learn/system-contract/

## 1. 简介

![image-20220414172858079](/bsc-relayer-img/image-20220414172858079.jpg)



[**bsc-relayer**](https://github.com/bnb-chain/bsc-relayer) 是一个独立进程，可单独运行，运行时首先会向 BSC 查询是否注册过，如果没有，则会自动在 BSC 上注册，需要 deposit 100 BNB，注册成功后，该 bsc-relayer 的信息被记录在 BSC 上名叫 `RelayerHub.sol` 的 solidity 系统合约中，之后就可以执行中继的服务了。

bsc-relayer 主要有两个功能：

- 拉取 BC 的块头，并同步给 BSC
- 拉取 BC 的跨链数据包，并同步跨链数据包给 BSC

相关代码：

注册过程: 注册	`bsc-relayer` 	到	`bsc chain`	合约，首先判断该地址是否已经被注册过，没有则在合约注册此 `bsc-relayer` 的 **address**；

![image-20220413115303206](/bsc-relayer-img/image-20220413115303206.jpg)

下面是[bsc-genesis-contract](https://github.com/bnb-chain/bsc-genesis-contract)  中的	`RelayerHub.sol`	合约；可以看到注册的hub都存放在了relayersExistMap中

![image-20220413115602656](/bsc-relayer-img/image-20220413115602656.jpg)



BSC 上存在 solidity 两个系统合约（除了这两个还有其它的系统合约），分别为 `TendermintLightClient.sol` 和 `CrossChain.sol`，bsc-relayer “同步块头” 的操作会调用 BSC 的 `TendermintLightClient.sol`，“同步跨链数据包” 的操作会调用 BSC 的 `CrossChain.sol`。

除此之外，bsc-relayer 还有部分其它的功能，例如本地查询状态，tx 追踪等。



## 2. bsc-relayer 与 BC、BSC 的连接通道

bsc-relayer 与 BC、BSC 均通过 RPC 进行通信，RPC 的信息记录在 config/config.json 文件下。

1. **bsc-relayer <-----> BC**
   bsc-relayer 通过发送不同的请求信息获取 BC 上的数据，例如 `abci_info`、`block` 等。
2. **bsc-relayer <-----> BSC**
   bsc-relayer 直接使用了 Etheruem 提供的 RPC 模块，可直接调用发送数据或者请求，其中最常用的是 `eth_sendRawTransaction` 进行交易的发送（用来调用前文提到的两个合约 `TendermintLightClient.sol` 和 `CrossChain.sol`）。

![img](/bsc-relayer-img/webp.webp)



## 3. 拉取、同步跨链数据包

### 3.1 拉取跨链事件 info

bsc-relayer 会一直轮询每个高度的 BC 块的所有 **跨链事件**，事件的格式如下：

```
{
  "type": "IBCPackage",
  "attributes":
  [
    {
      "key": "IBCPackageInfo",
      "value": "96::8::19"
    }
  ]
}
```

其中：

- **type** 为 "IBCPackage"，表示跨链包；
- **key** 为 "IBCPackageInfo"，表示跨链包的信息；
- **value** 通过“::”分隔为3个字段，分别为 `CrossChainID of destination chain`、`channel id`、`sequence`；

1. **CrossChainID**
   对于 bsc-relayer 来说，srcCrossChainID 为 BC 的 ChainID，destCrossChainID 为 BSC 的 ChainID。
2. **channel id**
   channel id 为跨链调用 BSC 系统合约（solidity）的 id，bsc-relayer 中包含有以下 4 个 id：

![image-20220413152547239](/bsc-relayer-img/image-20220413152547239.jpg)

3. **sequence**
   每个 channel id 都对应有一个 sequence，用来计数。

![image-20220414162456354](/bsc-relayer-img/image-20220414162456354.jpg)



### 3.2 拉取跨链事件 payload

bsc-relayer 根据 `channel id` 和 `sequence` 组合生成唯一标识，通过 RPC 向 BC 请求对应的 `payload`，这些 `payload` 以 bytes 的形式传到 bsc-relayer。

![image-20220421110839744](/bsc-relayer-img/image-20220421110839744.jpg)

### 3.3 同步

bsc-relayer 将上述的 payload 生成调用 `CrossChain.sol` 的 tx ，通过 RPC 发送到 BSC。（调用[bsc-genesis-contract](https://github.com/bnb-chain/bsc-genesis-contract)  中的	`CrossChain.sol`合约的`handlePackage`函数）

![image-20220414154232710](/bsc-relayer-img/image-20220414154232710.jpg)

执行此函数时，还会向中继此跨链数据包的 `bsc-relayer` 发送奖励。



## 4. 拉取、同步 BC 块头

bsc-relayer 拉取的是 BC 的块头，BC 的块头本质上是 Tendermint 块头，这里就不得不先介绍一下 Tendermint 的相关特性了：一个块的 **状态** 和 **签名** 等数据需要至少等到下一个块才能得到，所以如果需要验证高度为 H 的块，需要等到高度为 H+1 的块的 `LastCommit` 信息。

bsc-relayer 拉取 BC 块头的行为完全是 **自身驱动** 的，当出现以下两种情况时进行触发：

- **BC validator 集合更新**

​	BC 本地会定期更新 bc-validator 集合，BSC 需要获取每一轮的 bc-validator 集合信息，用于验证跨链数据，bc-validator 集合信息包括所有 bc-validator 的账号、公钥、投票（VotingPower）等。
 bsc-relayer 会一直轮询每个高度的 BC 块的 bc-validator 集合是否发生变化（包括当前正在工作的 bc-validator 集合和下一轮即将更新的 bc-validator 集合），如果发生变化，则会打包 **当前工作的 bc-validator 集合信息**、**下一轮更新的 bc-validator 集合信息**、**当前查询的 BC 高度的块头**，将其生成调用 `TendermintLightClient.sol` 的 tx，并发送到 BSC。

- **查到跨链数据包**

bsc-relayer 会一直轮询每个高度的 BC 块是否有跨链数据包，如果在高度 H 查到了跨链数据包，则会先打包高度为 H+1 的块头 和 validator 集合信息，将其生成调用 `TendermintLightClient.sol` 的 tx，并发送到 BSC。



## 5. gas问题

上述可知 bsc-relayer 会不停向 BSC 发送交易，相当于中继自己出 gas 进行工作。为了弥补这一损失，BSC 会在 `RelayerIncentivize.sol` 系统合约中向 bsc-relayer 发放系统 reward。

![image-20220414154522989](/bsc-relayer-img/image-20220414154522989.jpg)

奖励在合约中使用map `relayerRewardVault` 存储相应 `bsc-relayer`的`reward`；

在`bsc-relayer` 中会起协程每60s 调用 **bsc** 合约获取`relayerRewardVault`中的奖励。

![image-20220414155649919](/bsc-relayer-img/image-20220414155649919.jpg)

