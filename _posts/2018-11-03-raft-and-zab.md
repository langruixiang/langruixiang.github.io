---
layout: post
title: Raft协议和Zab协议及其对比
categories: [Paper]
description: Raft协议和Zab协议的原理及其对比
keywords: Raft，Zab，分布式一致性
---

本文主要讲述 Raft 的原理和 ZK 的原理，并将两者主要机制进行了对比。

# Raft Overview

Raft是一个分布式共识算法（Distributed Consensus Algorithm），国内一般把分布式共识算法称为分布式一致性算法，个人觉得这个翻译非常不好，一致性这个词在计算机领域出现的太多了，以至于你没办法真正理解这个一致性到底指的是什么，比如这里的一致性和数据库ACID的一致性就有很大区别。因此还是叫分布式共识算法比较清晰。

Raft是Diego Ongaro在斯坦福大学的博士论文，当时Paxos是分布式算法领域的一枝独秀，但是由于Paxos太过于晦涩难懂，以至于Google内部的Paxos的实现实际上是和Paxos论文是不一致的。Diego的目标是提出一个易于理解的分布式共识算法，弥补分布式共识算法领域理论和实践的鸿沟

![](/images/post/raft_phd.png)

Raft协议由三部分组成，Leader Election，Log Replication，Safety。同时Diego在其博士论文中提到了大量Raft相关的优化，这些优化对于Raft在实际工程领域的应用具有非常重要的作用。

从时间维度看，Raft由多个term组成，每进行一次选举，term自增。Raft每个节点的状态一共有三种：leader，folower，candidate。

* 节点启动默认是folower状态，超时没有收到leader的心跳则变为candidate状态；
* candidate状态收到超过半数节点的投票则变为leader，收到leader的心跳变为folower；
* leader收到更高term leader的心跳变为folower。

基本的Raft协议一共用到了两种 RPC， RequestVote RPC 和 AppendEntry RPC。RequestVote RPC主要用于 Leader Election 的投票。AppendEntry RPC 用于 Log Replication 和 Leader 发送心跳

![](/images/post/raft_state.png)

## Leader Election

如前文所述，节点启功时状态默认为 folower，如果 election timeout 没有收到 leader 心跳，怎变为 candidate，在 candidate 状态收到超过半数节点投票，则成功当选 leader。

![](/images/post/raft_leader_election.png)

上述描述的是最理想的状态，特殊情况下，可能两个节点差不多同时超时。这种情况下，先超时的节点获得超过半数节点投票，当选为 leader，后超时的节点在 candidate 状态下收到 leader 的心跳，转变为 folower。再为极端一点，两个节点同时超时，这时可能产生分票行为，导致两个节点都没有收到超过半数节点的投票。这种情况下认为此次选举失败，节点再等一个 election timeout，自增 term，开启一轮新的选举。

![](/images/post/raft_leader_election_same_time.png)

Raft 为了避免这种情况的发生，采用了随机 election timeout 的机制，这样避免两个节点同时超时。 随机的 election timeout 需要满足以下的理论要求。broadcastTime 是心跳时间间隔，即超时时间必须大于心跳间隔时间，不然 election timeout 就不能真正反映节点与 leader 失去了联系。MTBF 是机器的平均故障时间，一般是几个月到几年。超时时间必须小于机器的平均故障时间，不能说机器都 down 掉了，心跳还没有超时，这也失去了超时机制的意义。总得来说，这是一个非常松的要求，只是理论上必须有这样的限制条件。

![](/images/post/raft_election_timeout.png)

## Log Replication

说到 Log Replication，先来搞清楚什么是 Raft 的 Log。复制状态机的架构如下所示。分布式共识算法负责维护一系列的 log，每个 log 包含一条命令，共识算法负责保证所有节点上的 log 序列完全一致。这样就保证 log apply到状态机的时候，状态机的状态是一致的。

![](/images/post/raft_state_machine.png)

Raft 算法的日志结构如下。每条 log 包含一个 index，一个 term 和一条 command。由于每个 term 只有一个 leader，因此 Raft 的 log 有一个特性是如果某个 index 处 log 的 term 一致，则该 index 处对应的 command 也一定是一致的。这个特性在日志复制的 Log Match 过程中会被应用到。

![](/images/post/raft_log.png)

Raft 的 Log Replication 主要依赖于 AppendEntry RPC。该 RPC 具体定义如下。该 RPC 能够保证 log 一致性的关键在于 Log Match 过程。每次 AppendEntry RPC 都包含两个重要的属性，prevLogIndex 和 prevLogTerm。记录了当前所复制日志的前一条日志的 index 和 term。接受者收到后，会匹配一下 index 处的 log 的 term 是否一致。

* term 一致则对应的 command 一定是一致的。并且可以推出在此次新增的数据之前的数据是一致的，那么就接收此次 AppendEntry RPC 所携带的 log。
* term 不一致则说明之前已经存在数据不一致，则拒绝此次 RPC。发送者在收到拒绝后会向前搜索，直至找到第一个匹配成功的 log，请将此之后的 log 全部复制。

![](/images/post/raft_log_replication.png)

举例说明这个过程，如图所示。

* leader 要把 index 为10的日志复制给 a，则会匹配 index 为9处的 term，即 prevLogIndex 为9，prevLogTerm 为6，此时可以匹配成功，则复制AppendEntry RPC携带的 log
* leader 要把 index 为8的日志复制给e，则会匹配 index 为7处的 term，即 prevLogIndex 为7，prevLogTerm 为5。由于 eindex 为7处的 term 为4，匹配失败，则 leader 会向前搜索并进行匹配，直至 index 为5处的 log 匹配成功，则发送6之后所有的 log 给 e

![](/images/post/raft_log_replication_example.png)


## Safety

Raft 数据的安全性我们可以从两个角度理解：

* 每个状态机以严格相同的顺序执行相同的 command
* 所有已经 commit 的 Entry 必须出现在未来的所有任期内

对于第一条规则，已经通过 AppendEntry RPC 的 log match 得以保证。但是到目前为止，第二条规则还没办法满足。如图所示，假设leader 此时 down 了，b 节点率先 election timeout，此时，如果 b 节点得到了超过半数节点的投票当选 leader，name 显然红色虚线框内已经被 commit 的 entry 就丢失了。为了避免这种情况的出现，必须加一条限制：

> **限制一：**节点 m 向节点 n 发送了 RequestVote RPC，如果节点 n 发现节点 m 的数据没有自己新，则节点 n拒绝节点 m 的投票请求。这里的“新”包含两个方面，term 更大的数据更新，term 相同，index更大的数据更新。

加上这条限制之后，我们看到图中节点 b 最多只能拿到自己和节点 f 的投票，未超过半数，不能当选 leader。


![](/images/post/raft_safety_election_restriction.png)

更为普适的去理解为什么加了这个限制之后，就能保证一条已经被 commit 的 entry 一定会出现在未来的 term 中：

 1. 对于一条已经被提交的 log **I**，**I**一定被复制到了超过半数的节点上，记这个节点集合为**Q1**
 2. 对于之后的一个 leader **L**而言，**L**一定获得了超过半数节点的投票，记这个节点集合为**Q2**
 3. 根据鸽笼原理，**Q1**和**Q2**两个集合至少存在一个交集，记这个交集节点为**S**
 4. **S**既包含**I**同时又向**L**投了票，因此在**限制一**的限制下，**L**一定包含**I**

有了**限制一**就能满足第二条规则吗？看一个例子：

* a 时刻 S1 是leader复制了一条log到 S2 上
* b 时刻 S1 down 掉 S5 当选 leader
* c 时刻 S5 down 掉 S1 成功当选 leader，S1将之前 term2 的日志复制到了 S3 上，term2的日志已经超过半数。此时我们能将 term2 的日志标记为 commited 吗？

假设我们将 term2的日志标记为 commited，如果此时 S1 down 机 S5可能当选 leader，但是 S5上没有 term2的这条日志，就导致这条被标记为 commited 的日志没有出现在未来的 term 中。

为什么会发生这种情况？最根本的原因在于 S1 在 term4当选 leader 时，其他节点根本不知道 term4 的存在，导致 term 出现了倒退的现象，把 term4复制的日志给覆盖了。为了避免这种情况的产生，便有了另外一条限制：

> **限制二：**不直接提交之前 term 的log，必须通过提交本 term 的 log，间接的提交之前 term 的 log

加上这条限制，之前的例子中，S5就不可能当选 leader，因为超过半数的节点已经知道 term4的存在从而不会给 S5 投票。很多系统的实现中，都是在当选新 leader 后，立马提交一个 NOP Entry 来满足这条限制的。


![](/images/post/raft_commit_restriction.png)


## Raft相关的优化

Diego 的论文中，提到的优化很多，本次我们主要涉及以下4个。

### Log Compaction

随着客户端的请求，Raft 放入 log 会越来越多，占用的空间也越来越过。为了解决这个问题，必须将 log 进行合并。 日志合并并没有大一统的方法，跟状态机的实现有很大关系，有些状态机是在内存中，有些状态机是在硬盘中，本文主要讲解状态机在内存中的日志合并。

在内存中的状态机一般是通过 snapshot 机制实现日志合并的。如图所示，将 index 为5时刻状态机的 snapshort 保存在硬盘中，那么 5之前的所有 log 就可以删除了，达到日志合并的目的。对于大小在GB 或10GB 的状态机而言，这是一种行之有效的方法。当然你需要根据状态机的实际数据结构实现状态机的序列化和反序列化，并根据实际系统决定在什么样的时机进行 snapshot。可以使用 copy-on-write 的机制解决 snapshot 与客户端写请求的冲突。

![](/images/post/raft_log_compaction.png)

### Pipeline & batch & async flush

如图所示，正常情况下，Raft 的一个请求需要经历以下过程：

* 接收请求 
* 添加到 log 队列中 
* 将此条 log 刷盘 
* 复制给其他节点 
* 等待过半数节点提交（如果5个节点，由于自己已经刷盘，则需要等待其他4个中的2个）
* 将 log 标记为提交
* apply 到状态机
* 返回给客户端

异步 flush 则是说，我可以在 leader 刷盘之前，就将日志复制给其他节点，对应的过程变为

* 接收请求 
* 添加到 log 队列中 *将此条 log 异步刷盘*
* 复制给其他节点 
* 等待过半数节点提交（如果5个节点，由于自己尚未刷盘，则需要等待总共5个节点中的3个）
* 将 log 标记为提交
* apply 到状态机
* 返回给客户端

这样 leader 甚至可以再本身还没有将 log 刷盘的情况下提交一条 commit 一条 log。

![](/images/post/raft_async.png)


Raft 的 batch 机制非常容易实现，AppendEntry RPC 支持一次性复制多条日志，从而实现 batch。Pipeline 机制则是当前一条 log 的处理流程还没有结束的时候，就接收新的日志，从而提高日志数据处理效率。需要说明的是，batch 机制是收到一个请求后不立马处理，而是攒够一批数据后一次性处理。pipeline 则相反，收到一个请求后尽快处理并开始处理下一个请求。batch 机制有助于提高 Raft 的吞吐率，pipleline 机制有助于较少请求的响应时间。因此需要在 batch 和 pipeline 之间做一定的权衡。

### Lease Read

Raft 算法是不允许stale read 的存在的。

* Raft log read

正常情况下 Raft 的读操作和其他的 command 一样，leader 将 read 操作复制到过半的节点，等到 apply 到状态机的时候去读取状态机对应的值。因为Raft 的 log 是按序 apply 的，因此 read 操作 apply 的时候，其之前所有的操作一定 apply 过了，所以不会出现 stale read。

* readIndex read

但是 read 操作并不会更改状态机的值，可不可以省去把 read 操作复制给过半节点的过程呢？Diego 提出了一种 ```readIndex read``` 的算法。当收到用户请求的时候，将当前的 index 值记录到 readIndex 变量中，等待历史 log 的提交直至 readIndex 的提交，读取状态机对应的值。这种方法省去了 read log 的复制，只是等待 readIndex 之前所有的 log 提交即可。但是，在网络分区的情况下，如果客户端访问了一个已经过期的 leader，则可能导致客户端无限期的阻塞下去。因此，把 index 的值记录到 readIndex 变量前，leader 需要确认一下他自己当前是不是已经过期的 leader，他会向集群中所有节点发送请求，当收到过半节点的返回时，表明他目前不是过期的 leader，否则他目前是过期的 leader，拒绝客户端的读请求。

* lease read

leader 对于每次去请求都去确认一下自己当前是不是真正的 leader 也带来了很多开销，能不能避免每次都去确认呢？Diego 提出了```lease read```的算法，即当 leader 确认自己的身份时，会带一个租期的变量，folower 响应 leader 的请求则需要保证在租期内，不给其他节点投票，从而保证租期内，不会有新 leader 的产生。因此，一个租期内的读请求是不需要重复确认自己是不是真正的 leader 的。当然，此处需要考虑集群的时钟漂移问题。可以让用户设置一个集群的最大时钟漂移值s，在（lease-s）这段时间内，不需要重复确认自己是不是真正的 leader。

除此之外，folower 可以分担 leader 读请求的部分工作，为了保证没有 stale read，readIndex 的赋值必须在 leader 处完成，当时等待 log 提交至 readIndex并去状态机读取对应值这个过程是可以在 folower 完成的。

社区的一些反馈表明，readIndex read + batch 已经可以实现很好性能的读操作。

### Prevote

在Raft集群发生网络分区时，网络分区的节点由于收不到 leader 的心跳，term 会在不断的自增。当网络分区恢复时，由于 leader 收到该节点 RequestVote RPC 的请求，并发现其 term 比自身大，使得 leader 失去 leader 的身份，重新开始一轮选举，导致集群的抖动。正常而言，一个网络分区的节点不应该在网络恢复加入集群的时候，给集群带来这样的抖动。因此，Diego 提出了 Prevote 机制。当一个节点 election timeout 时，他不会立即自增 term，而是与集群中节点进行一次通信，如果能收到超过半数节点的响应，才会自增 term。Prevote 机制的存在，使得网络分区的几点 term 不会发生自增，在其重新加入集群时也不会导致集群的抖动。


# Zab Overview

原始的 Zab 协议分为四个阶段

* Phase 0：选择一个 leader，这个 leader 只需要 alive 并得到超过半数节点同意即可，对 leader 的数据是不是最新没有要求。
* Phase 1：leader 收集 各folower的数据信息，并将最新的数据复制到自身
* Phase 2：leader 将自身最新的数据复制给所有 folower
* Phase 3：处理客户端的读写请求

在 Zab 的具体实现 Zk 中，对 Zab 协议进行了部分改动：

* FLE：Fast Leader Election，Zk 的 FLE 将 Zab 的 Phase 0 和 Phase 1 合二为一，FLE 通过在选举的过程中，不断比较各节点的数据，实现最终选出的 leader 一定具有最新的数据
* Recovery Phase： 原始的 Zab 协议中，leader 是将所有的数据复制给 folower 实现数据同步的，在 Zk 的实现中，根据 folower 的 zxid 进行了分情况的处理，大部分情况下发送差异数据即可，减少通信的数据量
* Broadcast Phase：Zk 的Broadcast Phase与 Zab 的 Phase 3一致

**本文只根据伪代码讲解 Zk 的主要流程，详细可参考本人添加中文注释的 zk 源码可见[中文注释的 zk 源码](https://github.com/langruixiang/zookeeper)**

![](/images/post/zab_phase.png)

## FLE



![](/images/post/raft_zab_fle.png)

FLE的伪代码如图所示，关键代码介绍:

* **主要变量:** timeOut, ReceivedVotes, OutOfElection, P.state, P.vote<P.lastZxid, P.id>, P.queue  P.round
 * timeOut: 网络超时的判定标准
 * ReceivedVotes: 当前投票轮收到选票集合
 * OutOfElection: 用于集群已经形成稳定leader, folower, 自身处于looking状态时, 收到选票的集合
 * P.state: 当前节点状态, 选主过程中位election(对应zk实现的looking), 选举结束为LEADING或FOLOWING
 * P.vote<P.lastZxid, P.id> 当前节点的投票, 包含投票对应节点的lastZxid以及id(对应zk的myid)
 * P.queue: 收到其他节点选票队列,
 * P.round: 当前投票轮次
* **[line 1-4]:** 初始化
* **[line 5]:** 选票投给自己, 发送给所有节点
* **[line 8-10]:** 如果timeout超时, 还没有收到其他节点选票, 则重新发送自己给其他节点
* **[line 11]:** 收到的选票为election状态, 表明当前选举尚未结束, 参考**[line28]**
* **[line 12-17]:** 收到选票的选举轮次大于自身选举轮次, 更新自己选举轮次, 清空自己的选票箱,  pk选票 发送pk后的选票
* **[line 18-20]:** 收到选票和自己属于一个轮次, 并且收到选票节点数据比自己新, 则更新自己选票, 发送给其他节点
* **[line 21]:** 忽略选票轮次小于自己节点的选票
* **[line 23-24]:** 已经收到当前集群所有节点选票, 评估所有选票, 更新P.state结束选举 
* **[line 25-27]:** 尚未收到所有节点选票, 但是某个节点选票已超过板书, 等待一段时间选票, 评估当前选票, 更新P.state结束选举
* **[line 28]:** 收到选票为leading或folowing状态,此时分为两种情况:
  * 1. 当前处于选举阶段的尾声, 其他节点已经率先结束选举, 即**[line 29]**情况
  * 2. 当前集群已有稳定的leader, folower, 自己宕机重启或是新节点, 即**[line 39]**情况
* **[line 31-32]:** 对方状态是LEADING, 评估所有选票,  更新P.state为FOLOWING, 结束选举
* **[line 33-34]:** 对方状态是FOLOWING, 并且对方选票是自己, 自己再选票箱中超过了半数,  更新P.state为LEADING, 结束选举
* **[line 35-36]:** 对方状态是FOLOWING, 对方投票的节点在自己的选票箱中超过半数, 对方状态是LEADING, 更新P.state为FOLOWING, 结束选举. 实际上**[line 31-36]**没有分条件判断的必要, 直接评估选票, 更新P.state即可, zk的实现上也是这么做的, 这里应该是更多考虑算法的完备性
* **[line 40-42]:** 这里是为了算法的严谨性, 理论上程序不会走到这里(zk的实现也没有这个条件的判断)
* **[line 43-45]:** 更新自己的round, 评估选票, 结束选举

## Recovery Phase

![](/images/post/raft_zab_recovery.png)

关键代码介绍:

* **[line 2]:** 更新lastZxid, 纪元好加1, counter置0
* **[line 3]:** 建立连接后, folower先主动发送FOLOWERINFO信息给leader, 该信息包含了folower的lastZxid号, 并发送NEWLEADER信息给folower, 告诉folower最新的lastZxid
* **[line 5-10]:** folower的lastZxid小于leader的Zxid情况, 如果folower落后太多, 超过某个阈值, 则采取SNAP机制同步, 如果folower的不太多, 大于这个阈值, 则采取DIFF机制同步.
* **[line 11-12]:** folower的lastZxid大于leader的Zxid情况, 发送TRUNC消息给folower, 告诉folower自己最新的lastCommittedZxid
* **[line 15-16]:** ACKNEWLEADER消息标志着一个folower数据同步的完成
* **[line 19-20]:** folower连接到leader的第一个消息是FOLOWERINFO, 包含自己的lastZxid号
* **[line 25-27]:** 收到NEWLEADER消息, 进行相关校验, 但是先不回复ACKNEWLEADER消息
* **[line 29-30]:** TRUNC类型同步, 删除对应事务
* **[line 31-32]:** DIFF类型同步, 接收所有proposol并commit
* **[line 33-34]:** SNAP类型同步, 接收snapshort文件
* **[line 36]:** 发送NEWLEADER消息, 表示当前folower同步完成

## Broadcast Phase

![](/images/post/raft_zab_broadcast.png)

* **[line 2-4]:** 收到客户端的写请求, 向Q中所有的folower发送Proposol
* **[line 5-7]:** 某条Proposol的ACK超过半数, 向所有folower发送COMMIT消息
* **[line 9-11]:** 收到FOLOWERINFO消息, 表明有个节点加入集群, 可能是宕机恢复或是新节点, 这个地方实际上执行Recovery Phase数据同步流程
* **[line 13-15]:** 收到ACKNEWLEADER消息, 表明有个新节点数据同步完毕, 加入Q中
* **[line 19-21]:** folower收到Proposol, 返回ACK消息
* **[line 24-25]:** 保证事务的按需提交, 即顺序一致性
* **[line 27]:** 提交事务
 

# Raft Vs Zab

## Leader Election

从以下角度对比 Leader Election

* 检测 leader down 机：Raft 协议 leader 宕机仅仅由 folower 进行检测，当 folower 收不到 leader 心跳时，则认为 leader 宕机，变为 candidate。Zk 的 leader down 机分别由 leader 和 folower 检测，leader 维护了一个 Quorum 集合，当该 Quorum 集合不再超过半数，leader 自动变为 LOOKING 状态。folower 与 leader 之间维护了一个超链接，连接断开则 folower 变为 LOOKING 状态。
* 过期 leader 的屏蔽：Raft 通过 term 识别过期的 leader。Zk 通过 Epoch识别过期的 leader。这点两者是相似的
* leader 选举的投票过程：Raft 每个选举周期每个节点只能投一次票，选举失败进入下次周期才能重新投票。Zk 每次选举节点需要不断的变换选票以便选出数据最新的节点为 leader
* 保证 commited 的数据出现在未来 leader 中：Raft选取 leader 时拒绝数据比自己旧的节点的投票。Zk 通过在选取 leader 时不断更新选票使得拥有最新数据的节点当选 leader

![](/images/post/raft_vs_zab_leafer_election.png)

## Log replication and commitment

* 选取新 leader 后数据同步：Raft 没有固定在某个特定的阶段做这件事情，通过每个节点的 AppendEntry RPC 分别做数据同步。Zk 则在新leader 选举之后，有一个 Recovery Phase 做这个件事情。
* 选取新 leader 后同步的数据量：Raft 只需要传输和新 leader 差异的部分。Zab 的原始协议需要传输 leader 的全部数据，Zk 优化后，视情况而定，最坏情况下需要传输 leader 全部数据。
* 新 leader 对之前 leader 未 commit 数据的处理：Raft 不会直接 commit 之前 leader 的数据，通过 commit 本 term 的数据间接的 commit 之前 leader 的数据。Zk 在 Recovery Phase直接 commit 之前 leader 的数据。
* 新加入集群节点的数据同步：Raft 对于新加入集群的节点数据同步不会影响客户端的写请求。Zk 对于新加入集群的节点，需要单独走一下 Recovery Phase，目前是通过读写锁同步的，因此会阻塞客户端的写请求。（Zk 可以在这里使用 copy-on-write 机制避免阻塞问题？？）

![](/images/post/raft_vs_zk_replication.png)

## Log compaction

Raft 和 Zk 状态机的实现机制不同使得两者在 snapshot 的时候有很大差别。Raft是典型的传统复制窗台机，对于更新请求，严格复制对应的 command。Zk 则不是严格的复制状态机，对于更新请求，复制的不是响应的 command 而是更新后的值。如图所示。

* Raft 的 snapshot 机制对应了某一个时刻完成的状态机的数据
* Zk 是一种 fuzzy snapshot 机制，Zk snapshot 的时候，会记录下来当前的 zxid，恢复的时候，从该 snapshot 开始，replay 所有的 log 实现最终数据的正确性。Zk 甚至不存在一个时刻和 snapshot 的数据完全一致。

Zk 之所以可以这样做，很大程度上依赖于其状态机的实现，比如 y++ 这条 log 在 Raft的复制状态机机制下如果被重复执行就会导致错误的结果，但是 y <- 2 被重复执行不会影响数据的正确性。

![](/images/post/raft_vs_zk_snapshot.png)

## Client read

* Raft 严格禁止 stale read 的存在，无论是 Raft log read 或 readIndex read 或 lease read 都不会狐仙 stale read。
* Zk 默认情况下会出现 stale read，如果想避免 stale read 必须使用 sync() + read()

# 本文参考

1. Ongaro D. Consensus: Bridging theory and practice[D]. Stanford University, 2014.
2. Medeiros A. ZooKeeper’s atomic broadcast protocol: Theory and practice[J]. Aalto University School of Science, 2012, 20.


