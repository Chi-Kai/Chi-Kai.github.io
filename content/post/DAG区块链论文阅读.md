---
author : "KC"
type : "post"
tags :
    - 区块链
    - 论文阅读
title: "DAG区块链论文阅读"
description: 时不时补充
date: 2022-11-10
draft: false
toc: true
---


# 《Direct Acyclic Graph-Based Ledger for Internet of Things: Performance and Security Analysis》
## 问题背景

由于区块链的安全性，去中心化，可信性，在IoT系统上有可观的应用前景（如智能车，能源交易）。IoT系统具有规模大，资源受限的特性。所以其上的共识算法必须满足资源需求小,低消耗，和高的交易吞吐量。

现在主要的两种共识算法:PoW需要高的资源消耗，PoS的币龄证明可能造成垄断和中心化。

典型的区块链是一种单链结构，为了避免非法的fork，应用的共识算法必须降低新的block生成速率。这导致了吞吐量瓶颈和区块认证延迟的问题，在IoT系统上又有交易花费高和资源消耗大的问题。

DAG共识算法可以允许任何节点可以立即向ledger插入一个新的block，前提是它能先处理更早的交易。这种方式会造成很多fork，DAG有很多算法来避免在传统区块链上面临的double-spending问题(Markov Chain Monte Carlo algorithm and virtual voting algorithm)。DAG共识算法的交易吞吐是不受限制的，而且资源消耗很低，这符合IoT的应用场景。

## DAG概述
### 名词定义

![](https://s2.loli.net/2022/11/19/auRt2FzeCjwgsGM.png)

这里使用典型的Tangle算法来进行解释。

- Block: 所有块是记录信息的存储单元(包括交易，数字签名，哈希值)，在Tangle里一个块记录一个交易
    
- Tip: 还没有被验证的块(交易)
    
- Direct approval：直接验证，两个块直接由一条边来链接，称为直接验证。
    
- indirect approval：间接验证，两个块有通过一个块和
    
- Own weight: 与它的提出者的工作量有关
- Cumulative weight: 代表一个交易的认证级别。是一个交易自身own weight以及它直接证明和间接证明的交易的交易own weight总和。

### 共识过程

1. 节点创造一个块来储存交易
2. 节点通过MCMC tips 选择算法来选择两个没有冲突的tips，然后添加它们的hash到块中
3. 节点解决一个低难度的pow问题，来避免垃圾信息
4. 使用私钥给交易签名并广播，当其他节点收到时会检查是否合法
5. 成功添加的交易成为tip,等待被验证。直到它的cumulative weight 达到定义的标准。

### 分叉问题

![](https://s2.loli.net/2022/11/19/VnrEiPCoslHG9KY.png)

在分布式账本中，构建分叉以重做工作是篡改存储数据的唯一方法。基于此，double-spending的主要思想是将两笔相互冲突的交易并行放置在两条链上。在第一笔交易花费在服务上之后，攻击者扩展包含冲突交易的链并让它超过第一条链。当此操作成功时，第一笔交易将被孤立，攻击者可以多次使用token。

- 单链模型: 以最长的一个链为标准，正常的矿工会在最长的链上工作
- DAG模型: 以累计权重最大的子图为标准，正常的节点会通过MCMC tips 选择算法扩展权重最大的链。

## 



# 《TIPS: Transaction Inclusion Protocol with Signaling in DAG-based Blockchain》
## 问题背景

由于DAG区块链的高并发场景和网络延迟，矿工通常不能及时获取整个网络的更新信息，导致重复在一个并行区块包括相同的交易，在区块链中生成冗余的记录。这个交易包含冲突会浪费区块容量和降低系统性能。尽管DAG区块链已经限制交易的高并发，但是交易冲突的风险实际还会诱发**矿工收益**和**系统吞吐**的困境。

### 问题分析

![符号.png](http://tva1.sinaimg.cn/large/008upJWily1h804wzs4nsj30rk0jtgqa.jpg)

三种交易包含策略:

1. 随机包含($P^{rand}$): $p_{1}=p_{2}=\cdots=p_{m}=\frac{n}{m}$
2. 有优先级的随机包含($P^{priority}$): $p_{1} \geq p_{2} \geq \cdots \geq p_{m} \text { and } \frac{p_{1}}{f_{1}}=\frac{p_{2}}{f_{2}}=\cdots=\frac{p_{m}}{f_{m}}$
3. Top n ($P^{top}$): $p_{1}=p_{2}=\cdots=p_{n}=1 \text { and } p_{n+1}=p_{n+2}=\cdots=p_{m}=0$

这里仅考虑矿工收益中的交易费用奖励。

#### 收入困境

## 算法设计

《SilentDelivery: Practical Timed-delivery of Private Information using Smart Contracts》

