---
layout: post
title: Elasticsearch 架构原理和网易云 NES 最佳实践
categories: [Middleware]
description: 介绍 Elasticsearch 的架构原理和网易云 NES 服务最佳实践
keywords: Elasticsearch
---

介绍 Elasticsearch 的架构原理和网易云 NES 服务最佳实践

# Elasticsearch 架构简介

Elasticsearch 是一个分布式的搜索和分析引擎，可以用于全文检索、结构化检索和分析，并能将这三者结合起来。Elasticsearch 基于 Lucene 开发，现在是使用最广的开源搜索引擎之一，Wikipedia、Stack Overflow、GitHub 等都基于 Elasticsearch (后文简称 es) 来构建他们的搜索引擎。

## 节点类型

在介绍 es 架构之前有先介绍一下 es 节点类型:

* Master-eligible node

* Data node

* Ingest node

* Tribe node

为了介绍 es 架构，本文只关注前两种，后两种介绍可参考[Node Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)  。Master-eligible node 是指有资格竞选 Master 的节点，一般配置为奇数个节点，真正的 Master 从 Master-eligible node 中选出，负责 es 集群的节点发现和元数据管理。Data node 是存储数据的节点，es 的数据将会以 Shard 为单元保存在 Data node 中。每个 Shard 就是一个完整的 Lucene Index。对于一个 es 的 Index 而言。es 默认将其 Hash 分为 5 份(```number_of_shards```配置项)，称为 5 个 Primary Shard， 同时每个 Primary Shard 默认有1个数据副本(```number_of_replicas```配置项)，称为 Replica Shard。

## 架构概览

es 架构如图所示。启动之后，Master-eligible 节点通过选举选出 Master 节点。之后，Master 节点会周期性的其他节点发送 Ping 命令，Ping 命令类似于心跳，通过 Ping 命令 Master 知道各节点的存活状况、 Shard 在各个 Data 节点中的分布情况等，从而不断维护和更新整个集群的元数据信息。其他节点也会周期性的向 Master 发送 Ping 命令，感知 Master 的存活状况，当一个节点认为当前 Master 挂了，则会走重新选举，加入集群的流程。

我们将 es 中每一条记录称为一个 Document， 向 es 中添加 Document 并不会经过 Master，因此单个 Master 节点不会成为 es 集群的瓶颈。es 集群中每个节点都维护了一个 Routing Table，Routing Table 记录了 Index 对应 Primary Shard 和 Replica Shard 在各个节点的分布情况，只有 Master 节点可以更新 Routing Table，更新后把最新的 Routing Table Publish 给其他节点。

当客户端发送 Document 的 CRUD 请求时，请求可以连接 es 集群的任意一个节点，当前节点会根据 Routing Table 将请求路由到对应的 Primary Shard 上，Primary Shard 会把请求复制到 Replica Shard 上，当规定数量(one, quorum, all)的 Shard 执行成功后，请求由 Primary Shard 返回给客户端。

![](/images/post/es_architecture.png)

## Master 节点选举

es Master 选举并没有依赖如 zk 之类外部组件，而是自己实现的一套服务发现机制，默认采用 Zen Discovery。为什么没有依赖外部组件本人理解如下：1. 避免架构过重：依赖 zk 会大大加重 es 的架构，增加部署和运维的复杂度。2. Zen Discovery 模块建立在内部 RPC 模块 Transport 之上，使得 Zen Discovery 的实现相对简单。 

如图所示，es 选举大致可以分为两个阶段：集群信息收集，基于收集的信息投票。

![](/images/post/es_architecture_zen.png)

**集群信息收集阶段**

集群信息收集阶段采用了 Gossip 协议。es 的配置文件中有```discovery.zen.ping.unicast.hosts```这个配置项，实际上就是 Gossip 的 seed nodes，es 官方建议配置为所有的 Master-eligible 节点。es 配置文件中有一个```discovery.zen.ping_timeout```配置项，Zen Discovery 会在这个时间段内进行三次 Ping。初始情况下，节点会向所有seed nodes发送 Ping 命令。接收方每收到一个 Ping 命令就把发送方节点记录在一个 temporalResponses 的集合中，并把 temporalResponses 中的节点加上自己返回给发送方，发送方把返回的结果记录在自己的 pingCollections 中。这个过程实际上就是 Gossip 协议 Pull 的通信方式。从第二轮开始，节点将不仅仅向 seed nodes 发送 Ping 命令，而是向 seed nodes + temporalResponses 中所有的 node 发送 Ping 命令，也就是向 seed nodes 和 所有 Ping 过自己的节点发送 Ping 命令。依次类推，共进行三次。每次 Ping 的结果记录都在 pingCollections 中，pingCollections是一个 Map，所以可以自动去重。pingCollections最后转换成 List 也就是 fullPingResponse 返回，该阶段结束。如果对 Gossip 不了解，会觉得这个过程有点绕，下边举个简单的例子：

- initial state：seedNodes 是 node1，初始状态，每个节点只知道自己和 seedNodes 节点
- round 1： node 1只知道自己，不需发送ping； node 2 向 node 1发送 ping，node 1 知道了 node 2的存在；node 3 向 node 1 发送 ping，node 1 知道了 node 3 的存在
- round 2： node 1 向 node 2， node 3 发送 ping，node 2， node 3知道了 node 1的存在(本身通过 seedNodes 已经知道)；另外，node 2 向 node 1发送 ping，收到的返回包含 node 1，node 2，node 3，所以，node 2 知道了所有的节点。类似的，node 3 向 node 1 发送 ping， 收到的返回包含了 node 1， node 2，node 3，node 3 也知道集群的所有节点信息。
- round 3： 任一个节点向其他两个节点发送ping，ping 中包含本节点信息。任一节点收到 ping 请求后，将三个节点的信息返回给请求节点。

本例中，经过两轮 ping，所有节点已经知道了集群中全部节点信息。

![](/images/post/es_architecture_gossip_example.png)

**基于收集的信息投票**

收集完集群节点信息后，当前节点可能是扩容新增加的节点或者网络分区后恢复的节点，所以第一步是寻找集群中是否已经存在 Master，如果已经存在，则直接向 Master 发送 joinRequest。如果不存在则将所有的 Master-eligible node 按照\<stateVersion, id\> 排序。stateVersion 从大到小，以便选出集群元信息较新的节点作为 Master；id 从小到大，避免在 stateVersion 相同时发生分票无法选出 Master。排序后第一个节点即为 Master 节点。

* 如果这个节点是自己，则等待其他节点的 joinRequest，joinRequest 相当于其他节点的投票，等待```discovery.zen.minimum_master_nodes```-1个 joinRequest，则本节点成功当选 Master，回复其他节点的 joinRequest 请求同意加入集群。

* 如果这个节点不是自己，则向该节点发送 joinRequest。

如果 Master 没有收到足够的的 joinRequest，则此次选举失败，重新从集群信息收集开始，重复上述流程。选主流程详细分析可以参考[Elasticsearch Zen Discovery 选主实现原理](https://niceaz.com/2018/11/12/es-zen-discovery-master-election/)

## 节点探活

Master 节点选出后，就开始节点探活。如上文提到的 es 集群中存在两类节点探活。

* Master 节点 Ping 其他所有节点

* 其他节点 Ping Master节点。

默认情况下，会每隔1s(```ping_interval```配置项)进行一次 Ping，每次 Ping 的超时时间是30s(```ping_timeout```配置项)，3次(```ping_retries```配置项)Ping 失败则认为节点宕机。所以默认情况下，一个节点宕机，es 集群 至少需要 90s 才会开始相应的处理流程。因为一个节点宕机后会触发集群 Shard 的 Fail Over，重分配，复制等操作，这个过程可能涉及大量的数据迁移工作(实际上，Fail Over后，真正开始数据迁移前，es 还有一个 5min 的缓冲期，这 5min 是不影响数据的正常读写的，只是少一个副本)，所以 es 默认了一个相对较长的时间。如果宕机的节点不是 Master，则 Master 会更新集群的元信息，Master 节点将最新的集群元信息 Publish 给其他节点，其他节点回回复 Ack，Master 节点收到```discovery.zen.minimum_master_nodes```-1个 Master-eligible node 节点回复，则发送 Apply 消息给其他节点，集群状态更新完毕。如果宕机的节点时 Master，则其他的 Master-eligible node 则开始 上述的 Master 节点选举流程。

## Shard 内部原理

一个 Shard 就是一个 Lucene 的 Index，因此本节其实是在介绍 Lucene 的工作原理。从工作原理上讲，Lucene实际上是一个 LSM Tree(见引用1)。LSM Tree 是上个世纪90年代的提到的一种数据结构，这种数据结构充分利用磁盘顺序写速度大于随机写的特点，将随机写的操作都在内存里进行，当内存达到一定阈值后，使用顺序写一次性刷到磁盘里。LSM Tree 真正流行起来是在 Google 发表 Bigtable 论文(传送门见引用2)之后，Bigtable Tablet 的工作机制本质上就是 LSM Tree。BigTable 之后，Hbase，Cassandra，LevelDB，RocksDB 等采用 LSM Tree 作为存储引擎。实际上，Doug Cutting写于1999年，正式开源于2001年的 Lucene 是一种基于 LSM Tree 的倒排索引(Inverted Index)结构。什么是倒排索引？如果我们假设每篇文档都有一个 DocId，那么文档可以视为 DocId 到单词序列的映射，倒排索引就是单词到包含该单词的 DocId 集合的映射，所以称“倒排”索引。本文不会去介绍分词，tf-idf，BM25，向量空间相似度等构建倒排索引和查询倒排索引所用到的技术，读者只需要对倒排索引有个基本的认识即可。

Lucene 工作原理如图所示，当新添加一片文档时，Lucene 进行分词等预处理，然后将文档索引写入内存中，并将此次操作写入 translog，translog 类似于 mysql 的 binlog，用于宕机后内存数据的恢复。默认情况下，Lucene 每隔1s(```refresh_interval```配置项)将内存中的数据刷入到文件系统缓存中，称为一个 segment。一旦刷入文件系统缓存，segment就可以被用于检索。，因此```refresh_interval```决定了 es 数据的实时性。segment 在磁盘中是immutable 的，因此避免了磁盘的随机写，所有的随机写都在内存中进行。随着时间的推移，segment 越来越多，默认情况下，Lucene 每隔 30min 或 segment 空间大于512M，将缓存中的 segment 持久化落盘，称为一个commit point，删除对应的 translog。

![](/images/post/es_architecture_lsm_tree.png)

大量的segment 和 commit point 在磁盘中存在，会影响数据的读性能。因此 Lucene 会按照一定的策略将磁盘中的 segment 和 commit point 合并，多个小的文件合并成一个大的文件并删除小文件，从而减少磁盘中文件数据，提升数据的读性能。倒排索引的数据结构使得文件的合并是比较容易的。

![](/images/post/es_architecture_lsm_tree_compaction.png)

总的来说，Lucene 总体的工作原理是一个比较典型的 LSM Tree 的工作方式，Lucene 内部用到了很多数据结构用于加速文件的搜索，本文不详细阐述。

## 架构总结

其实对比下来，可以发现 es 在很多方面都和 Bigtable 很相似。

* es 的 Master 和 Bigtable 的 Master类似， 都用来存储集群的元信息，如 es shard 的分布和 Bigtable SSTable 的分布。

* Elasticsearch 的 Data Node 和 Bigtable 的 Tablet Server 类似，都用于托管响应的数据分片，如 es 的 Shard 和 Bigtable 的 SSTable。

* 就数据分片而言，es 的 Shard 和 Bigtable 的 SSTable 都是基于 LSM Tree 的存储机制。

* 集群信息维护类似，es 的 Master 和 Data Node 通过类似于心跳的 Ping 通信，不断维护集群的数据分片信息等集群元信息。Bigtable 的 Master 和 Tablet Server 通过心跳维护 Tablet 信息。

很多分布式数据库都采用了类似于这样的机制：

* 分布式存储总需要一个维护集群信息的地方，也就是 Master，这样集群的元信息才能一直，数据访问才不会错乱。

* 大量数据的存储总需要合适的分片策略，常见的分片方式如 Hash 和 Range，分片内部数据存储就需要合适的存储引擎如 InnoDB，RocksDB 等。

* 数据分片的位置信息和 Master 通过心跳去维护，这样数据分片的信息就是无状态的，如果 Master 宕机了只需要等待几个心跳，数据分片的信息就完整了。

es 和 Bigtable两点最大的不同在于：

1. 副本机制不同。Bigtable 底层基于 GFS，所以在 Bigtable 这一层，完全没有考虑副本的问题，全部丢给 GFS 去解决。es 则采用本地存储，es 的 Master 就需要解决诸如 Shard 的 Fail Over，Shard 的复制和迁移等。说到这点不同其实有点类似 AWS Aurora 和 Mirrored mysql 的区别，Aurora 把数据存储在了 S3上，实际上也是把副本机制从 Mirrored mysql 层下放到了共享存储层。做成这种架构的关键是有一个牛 x 的共享存储层，如 Google 的 GFS，AWS 的 S3。如果把 es 的存储放在成熟稳定的共享存储上，那么对 es 本身而言，只需要维护 Master 就可以，不涉及数据，几乎没有什么运维工作。这样做的一个弊端就是数据访问性能受到影响，毕竟共享存储的访问速度还是不及本地存储。Google 之后发表多篇数据存储论文(Bigtable,MegaStore,Percolator,Spanner)基本都是基于 GFS 的，相信这样确实减少了 Google 后续组件的开发难度和维护成本。

2. LSM Tree 的具体机制不同。Bigtable 的 LSM Tree 是 BloomFilter + SSTable Index + SSTable。es 的 LSM Tree 是 FST + 倒排索引。FST 是一种类似于 Trie 的数据结构，相比于 Trie 多了后缀压缩的过程，用于快速定位 Term 在倒排索引中的位置。SSTable 和 Inverted Index 的不同如图所示。这也最终决定了 Bigtable 是一个 KV 存储，es 是一个搜索引擎。

![](/images/post/inverted_index_sstable.png)

# 网易云 NES 服务最佳实践

网易云 NES 服务[Elasticsearch服务_网易云](https://www.163yun.com/product/elasticsearch)提供托管的 Elasticsearch 服务，支持快速部署、节点扩容、性能监控、报警等能力，让用户能够快速部署服务，轻松运维。NES 服务还提供了 logstash 和 kibana ，可以实现一键 ELK 集群的部署。目前公有云在华东1的可用区 B 和华东3的可用区 A 提供服务，并部署在[瀚海私有云_企业级私有云平台-网易云](https://www.163yun.com/product-npc)中提供私有云服务。

## 专属 Master 节点

如前文所述 Master 节点负责集群信息的维护，Data 节点负责数据的存储。节点的角色分别由以下两个配置项决定:

* node.master: true 

* node.data: true

两者组合可以产生四种组合：```<true, false>```则该节点只承担 Master 角色功能，称为专属 Master 节点。```<true, true>```则该节点既承担 Master 角色功能，又承担 Data节点功能，其实就是混部。```<false, true>```则该节点只承担 Data 节点功能。```<false, false>```则该节点既不承担 Master 节点功能也不承担 Data 节点功能，但是该节点依然拥有 Routing Table，可以承担请求转发的功能。

Master 节点维护的集群信息对于集群数据安全非常重要，因此生产环境建议采用专属 Master 的方式部署，即 Master-eligible节点的配置为```<true, false>```，Data 节点的配置为```<false, true>```。Master-eligible节点3或5个。Data 节点根据业务情况而定，建议多于Shard副本数，数据的 CRUD 请求也都发往 Data 节点。NES 服务提供了快速的专属 Master 部署和扩容，只需在创建集群的时候勾选专属 Master 选项即可拥有专属 Master 集群。

## 创建多个索引

Shard 是 es 集群数据恢复和调度的基本单位，但是 es 集群的 Shard 有一个缺点：没有分裂能力。不像 Bigtable 的 SSTable，在其体积过大的时候可以自己进行分裂，变成小的 SSTable 从容方便数据的均衡分布和数据调度。es 的 Shard不具备这个能力，如果你只创建一个索引，Shard 的体积只增不减，给数据均衡和调度带来很大不便，并且在集群出现问题时，可能增大运维的难度。一个好的解决方式是按照时间创建多个索引，比如每天/每周一个索引，这样每个 Shard 不会无限增长，并且体积差不多，方便Shard 的均衡和调度。Logstash 就对按时间创建索引提供了很好的支持。这样做的另外一个优点是当你查询和时间相关的数据时，你可以选择对应时间段内索引而不必去搜索全部的索引，从而提高查询效率，尤其是当你把 es 作为时序数据库去使用的时候，按照时间去创建多个索引给你带来巨大方便。

## 关注集群指标

磁盘总在你不知不觉中就用完了，因此生产环境一定要密切关注集群指标。NES 服务的节点监控指标可以在前端随时查看，涉及 cpu，memory，disk，network 等多各维度的监控指标。同时建议在[报警管理](https://www.163yun.com/product/alarm)配置关键指标的监控，尤其是数据盘的利用率，将集群故障防范于未然。如果真发生了数据盘用完的情况，如果你配置了专属 Master，大部分情况通过扩容 Data 节点即可自行恢复。其他情况可以咨询网易云相关人员。

## 一键 ELK 集群搭建

为了方便 es 集群的使用，NES服务的 es 集群默认提供 kibana 服务，集群创建完成即可使用，无需额外部署。同时在创建的 es 集群的时候或者 es 集群创建完成后都可以创建 Logstash 节点，Logstash 节点数可根据需求扩容或缩容，方便向 es 集群中导入数据和数据的预处理。这样用户便拥有了一个 ELK 集群的功能，提高日志查询处理的效率，并可以方便的进行数据可视化。

## 网易云日志服务

NES 服务和[网易云日志服务](https://www.163yun.com/product/log)提供了更加方便快捷的日志服务流程，用户只需在同一个 VPC 内分别创建 Kafka 集群和 es 集群，并在日志服务中创建日志项目选择对应的 kafka 集群 及 topic 和 es 集群，最后在日志服务的日志流中选择 Kafka 的 topic 和 es 集群的 index，日志服务将会自动将 Kafka 集群和 es 集群连接起来，完全省去了Logstash Pipeline 的配置。用户只需要将日志发送至 Kafka对应的 topic，就可以在 es 集群的 kibana 中或日志服务的前端中进行日志的查询和分析。

## 全文搜索引擎

ELK 的使用场景太过于经典以至于很多人只识 ELK 而不识 es，太多人认为 es 只是 ELK 中的一个组件，而忽略了 es 本身定位是一个全文搜索引擎。在分布式全文搜索领域，es 称得上是首选。Wikipedia，Stack Overflow，GitHub 等都基于 es 搭建了其搜索引擎。 es 针对不同的数据类型创建了不同的索引，并提供近实时搜索（NRT）。一种比较好的实践是将需要全文搜索的字段插入 es ，将原始信息插入其他的存储组件如 HBase，mysql 等，两者通过文档 id 关联，通过“主存储系统 + es 架构”既满足大规模数据的存储又提供搜索的能力。目前 NES 服务还没有提供这方面的支持，如果有需要可以给网易云提相关需求。

## 时序数据库

es 还提供了丰富的数据聚合能力，与专业的时序数据库如 InfluxDB，Druid 相比，es 作为时序数据库性能确实稍显逊色，但是 es 架构相对较轻，运维相对简单，在满足性能的前提下，把 es 作为时序数据库不失为一个好的选择。如开源的分布式追踪系统[Zipkin](https://zipkin.io/)就支持将 es 作为其存储组件，es 的母公司 elastic 也基于 es的数据聚合和分析能力逐渐提供 APM 的功能。网易云内部一个经典的实践是[网易云APM](https://www.163yun.com/product/apm)针对不同的应用场景提供了基于 Druid 和 es 的不同部署架构，充分满足大数据量、高性能和快速部署的不同应用场景。

# 引用

1. [Log-structured merge-tree - Wikipedia](https://en.wikipedia.org/wiki/Log-structured_merge-tree)

2. [bigtable](https://static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf)
