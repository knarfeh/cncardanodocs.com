---
layout: default
title: 乌洛波罗斯 权益证明算法
permalink: /cardano/proof-of-stake/
group: cardano
visible: true
---
<!-- Reviewed at c4c45ce9a7a8f4aa6d88a32829755196a017f6a1 -->

# 乌洛波罗斯权益证明算法

乌洛波罗斯权益证明算法是协议中最重要的部分。它定义了节点达到[账本](/glossary/#ledger)一致性的方式。

乌洛波罗斯算法是唯一一个基于科学证明的安全的区块链权益证明协议。


## 为什么要有权益证明？
不选择被比特币采用的 PoW（工作量证明）而选择 PoS (权益证明) 最重要的原因是考虑了能源消耗。运行比特币协议非常消耗资源，据估计，一个比特币的转账所需要的能源是3.8个美国家庭一天消耗的能源。随着越来越多的比特币矿工将资金投入矿业，运行比特币协议的能源要求只会越来越高，他们挖矿的难度也会越来越大。也是为什么研究人员尽力研究达成共识的替代算法，比如使用所谓的 BFT（Byzantine Fault Tolerant）一致性算法和 PoS 算法。

## 什么是权益证明算法

### 证明

『权益证明』的『证明』部分是要证明交易块是合法的。


### 权益

『权益』值得是节点上的地址所拥有的相对价值。『相对价值』指的是『卡尔达诺结算层系统中某个节点钱包上的价值除以总价值』。请阅读 [卡尔达诺结算层的平衡和权益](/cardano/balance-and-stake/) 章节获取更多信息。


## Slot 领导者

有正资产的节点称作 stakeholders，只有 stakeholders 能参与运行协议。stakeholders 必须被选举为 slot 领导者才让区块链生成块。Slot leader 可能监听到其他节点的交易信息，然后通过密钥生成一个交易区块发给全网。

你可以认为 slot 领导者是比特币中的矿工，但上述的一致性协议会确定谁，什么时候能挖矿，能挖到多少矿。

## Epochs 和 Slots

乌洛波罗斯 协议将物理的时间划分为 **epochs**, 每一个 epoch 又划分为 **slots**:

```
+----------+----------+-------+----------+--------------------> t
|  slot 0  |  slot 1  |  ...  |  slot N  |

 \                                      / \
  -------------- epoch M ---------------   -- epoch M+1 -- ...
```


请注意 slot 是相对较短的一段时间（比如 20 秒）。

每个 slot 有且只有一个 领导者（slot leader，SL）：

```
+----------+----------+-------+----------+----> t
|  slot 0  |  slot 1  |  ...  |  slot N  |

    SL 0       SL 1               SL N
```

Slot leader has a (sole) right to produce one and only one block during his slot:

slot 领导者有权在他的 slot 内生成一个区块。

```
  +------+   +------+           +------+
  | Bl 0 |<--| Bl 1 |<-- ... <--| Bl N |
  +------+   +------+           +------+
+----------+----------+-------+----------+----> t
|  slot 0  |  slot 1  |  ...  |  slot N  |

    SL 0       SL 1               SL N
```


这意味着 slot 领导者的数量一定等于一个 epoch 内 slots 的数量（不妨设为 `N`)，因此不可能在一个 epoch 里面生成超过 `N` 个区块。

如果 slot leader 错过了它的 slot（比如，在那个阶段它离线了），在下一次被选举为领导者之前，它没有权利在生成区块。因此，可以有一个或多个 slots 是空的（即，不生成区块），请注意，在一个 epoch 期间，它必须生成大部分块（至少50%+1）。


## Slot 领导者选举

Slot 领导者从所有的 stakeholders 中选举。请注意并不是所有的 stakeholders 能参与这次选举，只有有足够多的权益（比如，总量的 %2)才有资格。我们称这些 stakeholders 为『候选人』

在 epoch 的选举中会选举一个 slot 领导者参与下一次 epoch。因此，在 epoch `N` 结束的时候，我们就能知道 epoch `N+1` 的 slot leaders 是谁，并且这是不可更改的。

你可以把这样的选举当做 『公平抽签』：stakeholders 中的任何一个都能成为 slot leader。PoS 中一个很重要的的思想是，stakeholders 拥有的 stake 越多，它被选举为 slot 领导者的可能性也就越大。

请注意同一个 epoch，一个 stakeholders 可以被多次选做 slot 领导者。


### 多方计算

选举过程的根本问题是无偏性。我们需要一些随机性作为选举的基础，这样的话，选举的结果是随机的，公平的，问题是这个随机性从哪来？

我们用多方计算（multiparty computation (MPC) ）方法：每个选民独立进行一次『投硬币』的行为，然后与其他选民分享结果。这个想法就是：结果由每个选民随机产生，但最终它们在相同的最终价值上达成一致。

#### Commitment Phase

First of all, elector generates a secret (special random value). Then elector forms a
“commitment” which is a message that contains encrypted shares (see an explanation below) and
proof of secret.

Then elector signs this commitment with its secret key, specifies epoch's number and attaches
its public key. In this case everybody can check who created this commitment and which epoch
this commitment is for.

After that elector sends its commitment to other electors, so eventually each elector collects
commitments from all other electors. Please note that commitments are put into block, i.e.
they become a part of the blockchain.

#### Reveal Phase

Now elector sends an “opening”, special value for opening a commitment. You can think of a
commitment as a locked box (with a secret in it), and opening is a key we need to open this
box and get the secret from it.

All openings become a part of the blockchain as well as commitments.

#### Recovery Phase

This is a final phase.

Eventually elector has commitments and openings. But theoretically some elector can be an
adversary. And he could publish its commitment but **not** publish its opening.

In this case the honest electors can post all shares (mentioned above) to reconstruct the
secret. The idea is simple: we have to be sure that election will finish successfully even
if some of electors are adversaries.

Then elector verifies that commitments and openings are match, and if so, he extracts the
secrets from commitment and forms a seed from these secrets. So all electors get the same
seed, and it will be used for Follow the Satoshi algorithm.

### Follow the Satoshi

After electors get the seed (randomness we need), they have to select particular slot leaders for
the next epoch. This is where Follow the Satoshi (FTS) algorithm came in. It can be shown like
this:

```
         +-----+
SEED --->| FTS |---> ELECTED_SLOT_LEADERS
         +-----+
```

Let's elaborate a little bit on how a slot leader gets selected. The smallest, atomic piece
of value is a coin, we call it “[Lovelace](/glossary/#lovelace)”. Fundamentally, we can say
that the ledger produces distribution of coins, and since slot leaders can be selected from
stakeholders only, we are talking about distribution of stake. FTS is an algorithm that
verifiably picks a coin, and when coin owned by stakeholder `S` gets selected, `S` become a
slot leader. It is obviously that the more coins `S` has, the higher probability that one of
his coins will be picked.

The reason why it is called “Follow the Satoshi” is that in Bitcoin, atomic piece of currency
is called “Satoshi”, honoring Satoshi Nakamoto, the creator of Bitcoin. 

## Honest Majority

The fundamental assumption of a protocol is a **honest majority**. It means that
participants owning at least 50% + 1 of the total stake are honest ones. In this
case we can **prove** that adversaries cannot break _persistence_ and _liveness_
of the blockchain. Please read the [paper](/glossary/#paper) (pages 2 and 3) for
more details.
