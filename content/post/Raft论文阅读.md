---
author : "KC"
type : "post"
tags :
    - 分布式
    - 论文阅读
    - Raft
description: MIT6.824学习 
title: "Raft论文阅读"
date: 2022-10-30
draft: false
toc: true
---


## 领导人选举

首先按照论文中最关键的figure 2补全节点和RPC结构。

节点有三个状态: Leader,Candidate,Follower 和 两个计时器: 选举计时器，心跳计时器。

**Follower**: 有一个选举计时器，随机选举超时时间，每当选举超时，就转变为Candidate.

**Candidate**: 中间态，当Follower一端时间没有收到心跳，选举计时器到期，就会转变为Candidate,term加1，为自己投票并向其他节点发送投票请求，

**Leader**: 当一个Candidate获得半数以上的投票就会转变为Leader,最重要的节点，负责和客户端交互。需要定时向每个Follower发送心跳来维持权威。


1. 节点一开始状态都是Follower,Term为0。
2. 当一个选举计时器到期时，节点转变为Candidate，term加1，开始发送投票请求。这里有三种情况:
	- 其他节点按照先来先到的原则投票，获得半数以上的投票可以胜出成为Leader.
	- 在选举过程中得到更高term的RPC，Candidate会转变为Follower.
	- 如果两个Candidate获得同样的票数，等选举计时器再次超时，会开始下一轮投票。
3. 选出Leader后，Leader马上广播心跳来维持权威。
  
代码遇到的问题: 

1. 选举超时要真的随机时间，有的随机函数返回的是固定值，这里用时间做种子。
	```golang
	r := rand.New(rand.NewSource(time.Now().UnixNano()))
	t := time.Duration(ElectionTime+r.Intn(ElectionTime)) * time.Millisecond
	```
2. 由于网络原因，可能一些节点的回复不能及时收到。当收到一个超期的回复时，处理办法就是抛弃。

	```go

	// 如果回复晚了，不是同一个term或者leader则抛弃
	// 处理心跳回复
	if rf.currentTerm == args.Term && rf.state == StateLeader 

	// 在本Term内的投票且state仍为Candidate，超时过期的丢弃
	// 处理投票回复
	if rf.state == StateCandidate && rf.currentTerm == args.Term 

	```

3. 当收到一个更高term的回复RPC,Candidate转变为Follower，term变为更高的term,投票变为null。
4. 当收到一个更高的term的心跳时，状态转变为Follower,
term变为更高的term,投票变为null。
5. 节点在两种情况拒绝投票，一是Candidate的term小于自己，二是自己在本term中已经投过票。
	```golang
	if rf.currentTerm > args.Term ||
		(rf.currentTerm == args.Term && rf.votedFor != -1 && rf.votedFor != args.CandidateId) 
	```
6. 锁的使用: 当一个数据有写有读，写和读必须加锁。如果一个数据只读不写，不用加锁。
 
## 日志复制

当选举结束后，leader开始为客户端提供服务。客户端发出的每一条请求会被交给leader处理。

leader将每一条指令打包成一个entry <index,term,cmd>,将这个entry附加到日志中去，然后并行地发起 AppendEntries  RPCs 给其他的服务器，让他们复制这条entry。

当大部分服务器同意接收这个entry,leader将这个entry应用于状态机中，称为已提交，同时领导人的日志中之前的所有日志条目也都会被提交，包括由其他领导人创建的条目。然后将执行结果返回给客户端。

为了维护日志的一致性，要保证日志匹配特性:
- 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们存储了相同的指令。
- 如果在不同的日志中的两个条目拥有相同的索引和任期号，那么他们之前的所有日志条目也全部相同。

第一个特性由 “只有一个leader可以创建entry” 来保证。

第二个特性由附加日志 RPC 的一个简单的一致性检查所保证。它的步骤如下:

**在发送附加日志 RPC 的时候，会带上上一条entry信息。**如果跟随者在它的日志中找不到包含相同index和term的条目，那么他就会拒绝接收新的日志条目。

**找到最大共识点。** 在被跟随者拒绝之后，leader就会减小 nextIndex 值并进行重试(发送再上一个entry),直到找到最大共识点，被follower接收。之后，leader会强制覆盖follower最大共识点后面所有日志。这样就保证follower与leader日志始终一致。

> leader为所有follower节点维持一个nextIndex，记录每个follower下一个日志的index。当一个leader刚刚当选的时候，初始化所有nextIndex为自己最后一条日志的Index加1。（假设所有follower与leader日志保持一致）。

> 日志复制可以做一些优化。比如在正常复制时可以批量复制日志以减少系统调用的开销；在寻找共识点时可以只携带一条日志以减少不必要的流量传输。

## 安全性
### 选举限制

leader 只能发送日志给follower，而不能从follower接收日志，所以选出的leader必须包含集群中所有已经提交的日志。

在选举投票时，携带最新的日志信息，和follower相比较，看谁的日志最新。如果候选人更新，则获得投票。

> 这里更新的定义是: 通过比较两份日志中最后一条日志条目的索引值和任期号定义谁的日志比较新。如果两份日志最后的条目的任期号不同，那么任期号大的日志更加新。如果两份日志最后的条目任期号相同，那么日志比较长的那个就更加新。

### 提交之前任期内的日志条目

![幽灵复现](http://tva1.sinaimg.cn/large/008upJWily1h7oklyiaigj30ue0vaww7.jpg)

在上图中，raft 为了避免出现一致性问题，要求 leader 绝不会提交过去的 term 的 entry （即使该 entry 已经被复制到了多数节点上）。leader 永远只提交当前 term 的 entry， 过去的 entry 只会随着当前的 entry 被一并提交。（上图中的 c，term2 只会跟随 term4 被提交。）

如果一个 candidate 能取得多数同意，说明它的日志已经是多数节点中最完备的， 那么也就可以认为该 candidate 已经包含了整个集群的所有 committed entries。

因此 leader 当选后，应当立刻发起 AppendEntriesRPC 提交一个 no-op entry。注意，这是一个 Must，不是一个 Should，否则会有许多 corner case 存在问题。如:

- 读请求：leader 此时的状态机可能并不是最新的，若服务读请求可能会违反线性一致性，即出现 safety 的问题；若不服务读请求则可能会有 liveness 的问题。

- 配置变更：可能会导致数据丢失

实际上，leader 当选后提交一个 no-op entry 日志的做法就是Raft 算法解决 “幽灵复现” 问题的解法，[相关博客](https://mp.weixin.qq.com/s/jzx05Q781ytMXrZ2wrm2Vg)


