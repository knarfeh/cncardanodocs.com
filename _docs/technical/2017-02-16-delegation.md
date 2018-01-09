---
layout: default
title: 卡尔达诺股权委派
permalink: /technical/delegation/
group: technical
visible: true
---
<!-- Reviewed at c23493d7a33a82d559d5bd9d289486795cf6592f -->

# 卡尔达诺结算层股权委派

这一章描述权益委托过程的实现细节。

如前所述。为了产生新区块，被选举为领导者的股东必须在线。这种情况可能没有什么吸引力，因为大多数的当选股东都必须为了刷新随机数而参加投币协议（领导者选举过程的关键属性）。如果有很多当选的领导者，会让股东和网络都有很大的压力，因为需要广播和存储大量的提交和共享。

委派的功能允许被称为发行人（_issuers_） `I1...In` 的股权所有人将他们的『参与义务』转移给某些代表团（_delegates_） `D1...Dm`，这些代表团会在[投币协议](https://github.com/input-output-hk/cardano-sl/blob/4bd49d6b852e778c52c60a384a47681acec02d22/src/Pos/Ssc/GodTossing.hs)中代表股权所有人 `S1...Sn`。在这种情况下，真正参与到投币协议中节点的数量就少很多，可以看看[论文](/glossary/#paper)的第38页。

不仅如此，代表团不仅可以生产新区块，参与到 [MPC/SSC](/technical/leader-selection/#follow-the-satoshi) 中，还可以在[系统更新](/cardano/update-mechanism/)时进行投票。


## 策略

领导者可以将自己生产新区块的权利转移给代表团。为了转移这个权利，领导者使用一个代理委托的策略：领导者产生一个[代理签名钥匙](https://github.com/input-output-hk/cardano-sl/blob/4378a616654ff47faf828ef51ab2f455fa53d3a3/core/Pos/Crypto/SignTag.hs#L33)，或者说 PSK，然后代表团会使用它[签名](https://github.com/input-output-hk/cardano-sl/blob/ed6db6c8a44489e2919cd0e01582f638f4ad9b72/src/Pos/Delegation/Listeners.hs#L65)信息来认证一个区块。有两种类型的 PSK：重量级和轻量级（见下文）

具体来说，股权所有人通过自己的公钥构建一个特殊证书来指定代表团的身份。以便之后代表团可以在有限的信息空间内用已签名的证书在自己的公钥下为这些信息提供签名。


这是[代理签名](https://github.com/input-output-hk/cardano-sl/blob/d01d392d49db8a25e17749173ec9bce057911191/core/Pos/Crypto/Signing.hs#L256)的格式。它包括了：

* 代理私钥
* 签名

代理私钥包括：

1. omega 值
2. 发行人的公钥
3. 代表团的公钥
4. 代理证书

Omega (or ω) 是[论文](/glossary/#paper)中一个特殊的值。在我们的实现中，它是[一对 epoch 的标识符](https://github.com/input-output-hk/cardano-sl/blob/f374a970dadef0fe62cf69e8b9a6b8cc606b5c7d/core/Pos/Core/Types.hs#L235)。这些标识符定义了委托有效期：如果 epoch 索引在这个范围内那么生产的区块就是有效的。

[代理证书](https://github.com/input-output-hk/cardano-sl/blob/d01d392d49db8a25e17749173ec9bce057911191/core/Pos/Crypto/Signing.hs#L209)就是 omega 和代表团公钥的[签名](https://github.com/input-output-hk/cardano-crypto/blob/84f8c358463bbf6bb09168aac5ad990faa9d310a/src/Cardano/Crypto/Wallet.hs#L74)

## 重量级委派

重量级委托使用权益阈值 `T`，这意味着股权所有人拥有的权益不少于 `T` 时才能参与重量级委托。这个阈值在[配置文件](https://github.com/input-output-hk/cardano-sl/blob/42f413b65eeacb59d0b439d04073edcc5adc2656/lib/configuration.yaml#L224)中定义。就像主网的这个阈值是总权益的 0.03%，这个值可以通过系统更新来改变。

来自重量级委托的代理签名证书存储在区块链中。请注意发行者在每个 epoch 只能发布一个证书。

请注意重量级委托有一个传递关系，所以，如果 `A` 委派给 B，然后 B 又委派给 `C`，那么 `C` 代表的权益等于 `A + B`，而不仅仅是 `B`。


### 到期

在每一个 epoch 开始时，股权所有人不再传递阈值 `T`, 那么重量级委派证书就会过期。这样做是为了预防委派池膨胀攻击：用户提交了一个证书然后将自己所有的钱（高于阈值）都转到另一个账户，并且重复此操作。


## Lightweight Delegation

**WARNING: Currently, lightweight delegation is disabled and will be reworked in [Shelley release](https://cardanoroadmap.com/),
so information below can be outdated.**

In contrast to heavyweight delegation, lightweight delegation doesn't require
that delegate posses `T`-or-more stake. So lightweight delegation is available
for any node. But proxy signing certificates for lightweight delegation are not
stored in the blockchain, so lightweight delegation certificate must be broadcasted
to reach delegate.

Later lightweight PSK can be
[verified](https://github.com/input-output-hk/cardano-sl/blob/42f413b65eeacb59d0b439d04073edcc5adc2656/lib/src/Pos/Delegation/Logic/Mempool.hs#L309)
given issuer's public key, signature and message itself.

Please note that the rule "only one certificate per epoch" doesn't apply to lightweight delegation.
Since lightweight delegation certificates are not stored in the blockchain it's possible to issue
a lot of lightweight certificates per epoch and blockchain won't be bloated.

### Confirmation of proxy signature delivery

The delegate should take the proxy signing key he has and make a signature of PSK using
PSK and delegate's key. If the signature is correct, then it was done by the delegate
(guaranteed by the PSK scheme).

## Why Two Delegations?

You can think of heavyweight and lightweight delegations as of strong and weak delegations correspondingly.

Heavyweight certificates are stored in the blockchain, so delegated stake may participate in MPC
by being added to the stake of delegate. So delegate by many heavyweight delegations may accumulate
enough stake to pass eligibility threshold. Moreover, heavyweight delegates can participate in voting
for Cardano SL updates.

On the contrary, stake for lightweight delegation won't be counted in delegate's MPC-related stake. So
lightweight delegation can be used for block generation only.

## Revocation Certificate

Revocation certificate is a special certificate that issuer creates to revoke delegation.
Both heavyweight and lightweight delegation can be revoked, but not in the same way.

The revocation certificate is the same as standard PSK where issuer and delegate are the same
(in other words, issuer delegates to himself).

To revoke lightweight delegation issuer sends revocation certificate to the network and
_asks_ to revoke delegation, but it cannot _enforce_ this revocation, since lightweight PSKs
are not the part of the blockchain. So theoretically lightweight delegate can ignore revocation
certificate, and in this case it will remain a delegate until its delegation certificate expires.
But such a situation won't compromise the blockchain.

Revocation of heavyweight delegation is handled other way. Since proxy signing certificates
from heavyweight delegation are stored within the blockchain, revocation certificate will be
committed in the blockchain as well. In this case the node removes heavyweight delegation
certificate which was issued before revocation certificate. But there are three important notes
about it:

1.  If the committed heavyweight delegation certificate is in the node's memory pool, and revocation
    certificate was committed as well, the delegation certificate will be removed from the memory pool.
    Obviously, in this case delegation certificate will never be added to the blockchain.
2.  If a user commits heavyweight delegation certificate and _after that_ he loses its money, he still
    can revoke that delegation, even if by that time he does not have enough money (i.e. amount of money
    he has is less than threshold `T` mentioned above).
3.  Although an issuer can post only one certificate in the current epoch, he _can_ revoke his heavyweight
    delegation in the same epoch.
