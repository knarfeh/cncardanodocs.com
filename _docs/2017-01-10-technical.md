---
layout: default
title: 技术细节
group: base
permalink: /technical/
children: technical
---

<!-- Reviewed at d0868afac50ba6ffcbd95054e65cbf77fa513082 -->

# 卡尔达诺运算层技术细节

对于想要贡献原始客户端，以及想基于卡尔达诺清算层创建自己的客户端的开发人员来说，这一章节是一个起点。尽管如此，这一节将主要覆盖原始客户端，并有所扩展，在一段时间内可以把它当做最初的参考文档

## 高层次概述

一个卡尔达诺清算层节点是一个区块链节点。运行时，他会找到其他节点(通过 [DHT](http://ast-deim.urv.cat/cpairot/dhts.html))，然后开始执行区块链的相关任务。

卡尔达诺清算层中的时间会以 epochs 划分。epochs 又会以 slots 划分。 Epochs 和 slots 会被编号。 因此，slot `(3,5)` 被读作『第3个 epochs 的第5个 slot』 (第0个 slot 以及第0个 epoch 也是可以的).

卡尔达诺清算层会使用一些常量集, 特殊值定义在
[`constants.yaml` 配置文件中](https://github.com/input-output-hk/cardano-sl/blob/bf5dd592b7bf77a68bf71314718dc7a8d5cc8877/core/constants.yaml)。
主要有两种：生产模式和开发模式。 在本指南中，我们将参考生产常量。

假设卡尔达诺清算层的值是：:

-   slot 持续时间: 120秒,
-   安全参数 *k*: 60.

换句话说，, **一个 slot 可以持续120秒**, 而一个 epochs有 [`10×k`](https://github.com/input-output-hk/cardano-sl/blob/9ee12d3cc9ca0c8ad95f3031518a4a7acdcffc56/core/Pos/Core/Constants/Raw.hs#L161)
个 slot, 所以它可以持续**1200分钟**或**20个小时**.

每个 slot 上有一个节点被称作 slot 领导者。只有这个 slot 有权在这些 slot 中生成一个新区块；这个区块会被加入到区块链中。然而我们并不能确保这个区块一定会被生成(比如 slot 领导者在响应的过程中可能会离线)。

此外，slot 领导者可以将其权利委托给另一个节点 `N`；在这种情况下，节点 `N` 而非 slot 领导者将有权生成一个新的块。请注意，`N` 具有委托权的节点不能被称为 slot 领导者，它只是一个委托。

理论上可以将 slot 领导者的权力委托给多个节点，但是不推荐，之后会解释原因。此外，使用相同的密钥（即一台计算机上）我们可以运行中多个节点，假设有节点 `A`, `B`, `C`，如果节点 `A` 被选为 slot 领导者，不仅 `A` 本身，节点 `B` 和 `C` 都能够生成一个新区块。在这种情况下，每一个节点都将发出一个不同的块，网络将分叉 - 每个其他节点将只接受这些并发区块块中的一个。之后，这个分叉将被淘汰。

在 epoch 中，节点之间相互发送 MPC 消息，以达成共识，谁将被允许在下一个时期生成区块。Data 消息中的有效载荷 （以及事务）会被包含在块中。

一个地址持有的货币（或『股份』）越多，被选择生成一个区块的可能性就越大。请阅读 [Ouroboros 权益证明算法](/cardano/proof-of-stake/)获取更多细节。


简而言之:

1. 发送信息，
2. 接收信息/交易/等等，
3. 形成一个区块 (如果你是 slot 领导者的话)，
4. 重复。

## 商业逻辑

### 接收者

接收者处理传入的消息并对其作出响应。各种补充的听众不会被覆盖，而是集中在一个接收者上。

接收者大多使用[中继框架](/technical/protocols/csl-application-level/#invreqdata-and-messagepart)，其中包括三种类型的消息：

* `Inventory` 消息：节点在获取新数据时向网络发布消息。  
* `Request` 消息：如果某个新数据没有被这个节点获取的话，节点会向其他节点获取在 `Inventory` 消息中的新数据。  
* `Data` 消息：节点对 `Request` 消息回复的数据。`Data`消息包含具体的数据。

例如，当用户创建新的交易时，钱包将具有交易 ID 的 `Inventory` 消息发送到网络。如果收到 `Inventory` 的节点没有该 ID 相关的交易记录，那么它会回复 `Request` 消息，然后钱包会在 `Data` 消息中发送该交易信息。节点收到 `Data` 消息后，将 `Inventory` 消息发送给 DHT 网络中的邻居，并重复之前的操作。

另一个例子 - 区块接收者：[`handleGetHeaders`](https://github.com/input-output-hk/cardano-sl/blob/69e896143cb02612514352e286403852264f0ba3/src/Pos/Block/Network/Listeners.hs#L30)，
[`handleGetBlocks`](https://github.com/input-output-hk/cardano-sl/blob/69e896143cb02612514352e286403852264f0ba3/src/Pos/Block/Network/Listeners.hs#L50)，
[`handleBlockHeaders`](https://github.com/input-output-hk/cardano-sl/blob/69e896143cb02612514352e286403852264f0ba3/src/Pos/Block/Network/Listeners.hs#L77)。

### Worker

一个 Worker 会在一个时间区间内进行重复性的工作. 比如：


- [`onNewSlotWorker`](https://github.com/input-output-hk/cardano-sl/blob/69e896143cb02612514352e286403852264f0ba3/infra/Pos/Communication/Protocol.hs#L218)：在每个插槽的开始时运行。做一些清理，然后运行其他功能。这个 Worker 在这个 epoch 的开始时也会创造了一个 『起始块』。有两种类型的块：『生成块』和『主块』。主块储存在区块链中; 在 epoch 之间，每个节点都会间断性地生成块。主块不会被告知其他节点。但是，如果节点离线一段时间，并且需要同步区块链，节点可以请求其他人的创世区块。
- [`blkOnNewSlot`](https://github.com/input-output-hk/cardano-sl/blob/d01d392d49db8a25e17749173ec9bce057911191/src/Pos/Block/Worker.hs#L69): 创建一个新块（当轮到节点创建一个新块时），并将其发给其他节点。


## 权益证明

卡尔达诺清算层的核心基于 Ouroboros 权益证明算法。正如同名的[白皮书](https://eprint.iacr.org/2016/889)所描述的那样。


## 分叉

通常，一个链（主链）由一个节点维护，但最终可能会出现分叉链。回想一下，只有区块 `k` 和更多 slot 被认为是稳定的。这样一来，如果接收一个区块，它既不是区块链的一部分也不是 blockchain 的延续，我们首先检查其复杂程度（复杂性是链的长度）是否比我们的大，TODO


然后我们开始随后请求来自先前块提供替代链头的节点。如果我们来得深入k插槽前，替代链被拒绝。否则，一旦我们到达我们连锁店中​​存在的区块，替代链就会被添加到存储区。从国家的角度来看，我们存储和维护所有可行的替代链。如果看起来一个替代链比主链更长，那么它们被替换，使替代链成为新的主链。

Generally, one chain (the *main chain*) is maintained by a node, but eventually
alternative chains may arise. Recall that only blocks `k` and more slots deep are
considered stable. This way, if a block which is neither a part nor a
continuation of our blockchain is received, we first check if its complexity is
bigger than ours (the complexity is the length of the chain), and then we start
subsequently requesting previous blocks from the node that provided an
alternative chain header. If we come deeper than `k` slots ago, the alternative
chain gets rejected. Otherwise, once we get to the block existing in our chain,
the alternative chain is added to storage. From the standpoint of state, we
store and maintain all the alternative chains that are viable. If it appears
that an alternative chain is longer than the main chain, they are swapped,
making the alternative chain the new main chain.

## Supplemental parts

### Slotting

The consensus scheme we use relies on correct slotting. More specifically, it
relies on the assumption that nodes in the system have access to the current
time (small deviations are acceptable), which is then used to figure out when
any particular slot begins and ends, and perform particular actions in this
slot.

System start time is a timestamp of the `(0,0)` slot (i.e. the 0-th slot of the 0-th
epoch).

## P2P Network

### Peer discovery

We use Kademlia DHT for peer discovery. It is a general solution for distributed
hash tables, based on [a whitepaper by Petar Maymounkov and David Mazières,
2002](https://pdos.csail.mit.edu/~petar/papers/maymounkov-kademlia-lncs.pdf).

However, we only take advantage of its peer discovery mechanism, and use none of
its hash table capabilities.

In short, each node in the Kademlia network is provided a `160`-bit id generated
randomly. The distance between the nodes is defined by `XOR` metric. The network
is organized in such a way that node knows no more than `K` (`K=7` in the
original client implementation) nodes for each relative distance range:
`2^i < d <= 2^(i+1)`.

Initial peer discovery is done by
[sending](https://github.com/serokell/kademlia/blob/bbdca50c263c6dae251e67eb36a7d4e1ba7c1cb6/src/Network/Kademlia/Implementation.hs#L194)
a Kademlia `FIND_NODE` message with our own node id as a parameter to [a
pre-configured set of
nodes](https://github.com/input-output-hk/cardano-sl/blob/43a2d079a026b90ba860e79b5be52d1337e26c6f/src/Pos/Constants.hs#L89)
and the nodes passed by the user on the command line. Our implementation
[sends](https://github.com/input-output-hk/cardano-sl/blob/43a2d079a026b90ba860e79b5be52d1337e26c6f/infra/Pos/DHT/Real/Real.hs#L228)
this request to all known peers at once and then waits for the first reply.

While the client runs, it collects peers per Kademlia protocol. The list of
known peers is preserved and
[restored](https://github.com/serokell/kademlia/blob/bbdca50c263c6dae251e67eb36a7d4e1ba7c1cb6/src/Network/Kademlia.hs#L197)
between subsequent launches. For each peer, we keep their [host and port
number](https://github.com/serokell/kademlia/blob/bbdca50c263c6dae251e67eb36a7d4e1ba7c1cb6/src/Network/Kademlia/Types.hs#L42),
as well as their [node
id](https://github.com/serokell/kademlia/blob/bbdca50c263c6dae251e67eb36a7d4e1ba7c1cb6/src/Network/Kademlia/Types.hs#L70).

### Messaging

Kademlia already provides the notion of nodes that are known. Such nodes can be
called *neighbors*. To send message to all nodes in a network, you can send it
to neighbors, they will resend it to their neighbors, and so on. But sometimes
we may need to not propagate messages across all network, but send it to
neighbors only. Hence we have three types of sending messages:

-   send to a node,
-   send to neighbors,
-   send to network.

#### Message types

To handle this, three kind of message headers are used, and there are two
message types:

-   Simple: sending to a single peer.
-   Broadcast: attempting to send to the entire network, iteratively sending
    messages to neighbors.

Broadcast messages are resent to neighbors right after retrieval (before
handling). Also, they are being checked against LRU cache, and messages that
have been already received once get ignored.

### Leaders and rich men computation (LRC)

"Slot leaders" and "rich men" are two important notions of Ouroboros Proof of
Stake Algorithm.

-   Slot leaders: Slot leaders for the current epoch (for each slot of the
    current epoch) are computed by [Follow the
    Satoshi](/cardano/proof-of-stake/#follow-the-satoshi) (FTS) algorithm in the
    beginning of current epoch. FTS uses a `shared seed` which is result of
    [Multi Party Computation](/cardano/proof-of-stake/#multi-party-computation)
    (MPC) algorithm for previous epoch: in the result of MPC some nodes reveal
    their seeds, `xor` of these seeds is called `shared seed`.

-   Rich men: Only nodes that have sent their VSS certificates and also have
    enough stake can participate in the MPC algorithm. So in the beginning of
    epoch node must know all potential participants for validation of MPC
    messages during this epoch. Rich men are also computed in the beginning of
    current epoch.

Rich men are important for other components as well; for instance, Update system
uses rich men for checking that node can publish update proposal and vote.

There are two ways of computing who the rich men will be: - considering common
stake - considering delegated stake (Ouroboros provides opportunity to delegate
own stake to other node, see more in [Delegation
section](/cardano/differences/#stake-delegation))

MPC and Update System components need rich men with delegated stake, but
Delegation component with common stake.

## 常量

卡尔达诺清算层使用一些基础常量。他们的值经过了协议原作者和独立安全评论员的讨论，因此强烈推荐可选客户端使用这些常量。 

这些常量在 
[`constants.yaml` 配置文件](https://github.com/input-output-hk/cardano-sl/blob/bf5dd592b7bf77a68bf71314718dc7a8d5cc8877/core/constants.yaml)
中定义，分为生产环境和开发环境。
