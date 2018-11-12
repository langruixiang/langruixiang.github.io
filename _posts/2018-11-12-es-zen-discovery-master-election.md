---
layout: post
title: Elasticsearch Zen Discovery 选主实现原理
categories: [Middleware]
description: 分析 Elasticsearch Zen Discoveery 的实现
keywords: Elasticsearch, Zen Discovery
---

Zen Discovery 是 Elasticsearch 选主和集群状态管理的模块，本文结合 Elasticsearch 6.3 源码分析 Zen Discovey 模块的实现。

# Zen Discovery 概览

Zen Discovery 从功能上可以分为两部分，第一部分是集群刚启动时的选主，或者是新加入集群的节点发现当前集群的Master。第二部分是选主完成后，Master 和 Folower 的相互探活。Zen Discovery 管理着整个集群的 Meta Data。其功能非常类似于 Zookeeper 的功能，但是Elasticsearch 的架构设计使得数据的 CRUD 过程都是不经过 Zen Discovery 模块的，所以 Zen Discovery 是一个相对轻量级的模块，尤其是 Zen Discovery 的通讯复用整个集群的通讯模块（Transport Module），Zen Discovery 的实现还是比较简单的。这可能也是 Elasticsearch 不依赖 Zookeeper 的一个重要原因，避免架构过重。

> The zen discovery is integrated with other modules, for example, all communication between nodes is done using the transport module.

本文主要介绍 Zen Discovery 选主实现原理

***带中文注释的源代码可见本人 github 仓库[中文注释源代码](https://github.com/langruixiang/elasticsearch/tree/6.3)***

# 选主实现原理

选主分为两类情况，一类是集群刚启动，不存在 Master，需要从所有的节点中选出 Master；另一类是一个节点加入一个集群，找到当前集群的 Master 加入。不管是哪一种情况，都经历了两个过程：同步集群信息，基于集群信息选主。

选主阶段时序图如下

![](/images/post/zen_sequence_diagram.png)

## 同步集群信息

这个阶段，Es 进行了类似于 Gossip 通信的过程。该过程的入口是 ZenDiscovery 的 findMaster 函数，调用流程如下

着重分析 UnicastZenPing.ping 函数:

* 该函数首先获取 seedNodes，seedNodes 的来源有两处：1. yml 配置文件中配置的 seedNodes；2. File-Based Discovery Plugin配置的 seedNodes。关于File-Based Discovery参见[File-Based Discovery Plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/current/discovery-file.html)
* 构造 PingRound 和 PingSender，具体的 ping 逻辑在 PingRound 里处理
* 配置文件中有一个```discovery.zen.ping_timeout```参数，Es 在这个时间内共进行了3轮 ping
* 最后执行```finishPingingRound```函数，实际上执行的是 CompleteableFuture 的 complete 函数，结束上层的阻塞等待，并返回 pingCollections 结果。

```
// UnicastZenPing.ping函数
protected void ping(final Consumer<PingCollection> resultsConsumer,
                    final TimeValue scheduleDuration,
                    final TimeValue requestDuration) {
    final List<DiscoveryNode> seedNodes;
    try {
        /*
         * 从配置文件中获取 seed nodes
         */
        seedNodes = resolveHostsLists(
            unicastZenPingExecutorService,
            logger,
            configuredHosts,
            limitPortCounts,
            transportService,
            UNICAST_NODE_PREFIX,
            resolveTimeout);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }

    /*
     * 加入File-Based Discovery Plugin发现的 seed nodes
     */
    seedNodes.addAll(hostsProvider.buildDynamicNodes());
    final DiscoveryNodes nodes = contextProvider.clusterState().nodes();
    // add all possible master nodes that were active in the last known cluster configuration
    for (ObjectCursor<DiscoveryNode> masterNode : nodes.getMasterNodes().values()) {
        seedNodes.add(masterNode.value);
    }

    final ConnectionProfile connectionProfile =
        ConnectionProfile.buildSingleChannelProfile(TransportRequestOptions.Type.REG, requestDuration, requestDuration);
    /*
     * pingingRound 是一个关键的类, 负责向所有的 seedNodes 发送 ping, 并把结果记录在 pingCollections 中
     */
    final PingingRound pingingRound = new PingingRound(pingingRoundIdGenerator.incrementAndGet(), seedNodes, resultsConsumer,
        nodes.getLocalNode(), connectionProfile);
    activePingingRounds.put(pingingRound.id(), pingingRound);
    final AbstractRunnable pingSender = new AbstractRunnable() {
        @Override
        public void onFailure(Exception e) {
            if (e instanceof AlreadyClosedException == false) {
                logger.warn("unexpected error while pinging", e);
            }
        }

        @Override
        protected void doRun() throws Exception {
            sendPings(requestDuration, pingingRound);
        }
    };
    /*
     * 一共发送三次 pingRound, pingCollections 是一个 map, 多次发送自动去重, 增加了网络抖动情况下的鲁棒性
     */
    threadPool.generic().execute(pingSender);
    threadPool.schedule(TimeValue.timeValueMillis(scheduleDuration.millis() / 3), ThreadPool.Names.GENERIC, pingSender);
    threadPool.schedule(TimeValue.timeValueMillis(scheduleDuration.millis() / 3 * 2), ThreadPool.Names.GENERIC, pingSender);

    /*
     * finishPingingRound 执行 pingConsumer 的 accept 函数
     * 外边传进来的是 CompleteableFuture的complete, 所以此处CompleteableFuture.get()停止阻塞返回pingCollection
     * 见ZenDiscovery.java pingAndWait函数
     */
    threadPool.schedule(scheduleDuration, ThreadPool.Names.GENERIC, new AbstractRunnable() {
        @Override
        protected void doRun() throws Exception {
            finishPingingRound(pingingRound);
        }

        @Override
        public void onFailure(Exception e) {
            logger.warn("unexpected error while finishing pinging round", e);
        }
    });
}
```

UnicastZenPing.ping一共执行了三轮 ping，下边着重分析每一轮 ping 做了什么，主要逻辑在UnicastZenPing.sendPings 函数

* 首先构造 pingRequest，pingRequest 包含了以下信息

> private final long id; 单调递增的 id，反映 pingRequest 的时序关系

> private final ClusterName clusterName; 当前集群的 clusterName，用于区分集群

> private final DiscoveryNode node; 节点本身

> private final DiscoveryNode master; 自己所知的当前集群的 master

> private final long clusterStateVersion; 自己 clusterStateVersion

* 构造要发送节点的集合，包含两部分：1. seedNodes 集合；2. nodesFromResponses中的节点，nodesFromResponses记录了ping 过当前节点的节点。这一步是 Gossip 协议的一个关键，要通讯的节点并不是静态的，而包括那些曾经 ping 过自己的节点，即便他们不再 seedNodes 中
* 遍历要发送的节点集合，发送 ping 请求

```
// UnicastZenPing.sendPings 函数
protected void sendPings(final TimeValue timeout, final PingingRound pingingRound) {
    final ClusterState lastState = contextProvider.clusterState();
    /*
     * 构造 pingRequest, 包含本身节点信息
     */
    final UnicastPingRequest pingRequest = new UnicastPingRequest(pingingRound.id(), timeout, createPingResponse(lastState));

    Set<DiscoveryNode> nodesFromResponses = temporalResponses.stream().map(pingResponse -> {
        assert clusterName.equals(pingResponse.clusterName()) :
            "got a ping request from a different cluster. expected " + clusterName + " got " + pingResponse.clusterName();
        return pingResponse.node();
    }).collect(Collectors.toSet());

    /*
     * 构造要发送请求的节点集合, 包含两部分:
     * 1. seed nodes, 在配置文件中配置
     * 2. 所有给自己发送过 ping 请求的节点
     */
    // dedup by address
    final Map<TransportAddress, DiscoveryNode> uniqueNodesByAddress =
        Stream.concat(pingingRound.getSeedNodes().stream(), nodesFromResponses.stream())
            .collect(Collectors.toMap(DiscoveryNode::getAddress, Function.identity(), (n1, n2) -> n1));


    // resolve what we can via the latest cluster state
    final Set<DiscoveryNode> nodesToPing = uniqueNodesByAddress.values().stream()
        .map(node -> {
            DiscoveryNode foundNode = lastState.nodes().findByAddress(node.getAddress());
            if (foundNode == null) {
                return node;
            } else {
                return foundNode;
            }
        }).collect(Collectors.toSet());

    nodesToPing.forEach(node -> sendPingRequestToNode(node, timeout, pingingRound, pingRequest));
}

```
下边分析UnicastZenPing.sendPingRequestToNode函数做了什么

* 调用了 Transport Module 的transportService.sendRequest函数，并注册了一个收到 response 后的回调。Transport Module的实现不再本文的分析范围，关键分析该回调函数。

```
// UnicastZenPing.sendPingRequestToNode函数
private void sendPingRequestToNode(final DiscoveryNode node, TimeValue timeout, final PingingRound pingingRound,
                                   final UnicastPingRequest pingRequest) {
    submitToExecutor(new AbstractRunnable() {
        @Override
        protected void doRun() throws Exception {
            Connection connection = null;
            if (transportService.nodeConnected(node)) {
                try {
                    // concurrency can still cause disconnects
                    connection = transportService.getConnection(node);
                } catch (NodeNotConnectedException e) {
                    logger.trace("[{}] node [{}] just disconnected, will create a temp connection", pingingRound.id(), node);
                }
            }

            if (connection == null) {
                connection = pingingRound.getOrConnect(node);
            }

            logger.trace("[{}] sending to {}", pingingRound.id(), node);
            transportService.sendRequest(connection, ACTION_NAME, pingRequest,
                TransportRequestOptions.builder().withTimeout((long) (timeout.millis() * 1.25)).build(),
                getPingResponseHandler(pingingRound, node));
        }

        @Override
        public void onFailure(Exception e) {
            if (e instanceof ConnectTransportException || e instanceof AlreadyClosedException) {
                // can't connect to the node - this is more common path!
                logger.trace(() -> new ParameterizedMessage("[{}] failed to ping {}", pingingRound.id(), node), e);
            } else if (e instanceof RemoteTransportException) {
                // something went wrong on the other side
                logger.debug(() -> new ParameterizedMessage(
                    "[{}] received a remote error as a response to ping {}", pingingRound.id(), node), e);
            } else {
                logger.warn(() -> new ParameterizedMessage("[{}] failed send ping to {}", pingingRound.id(), node), e);
            }
        }

        @Override
        public void onRejection(Exception e) {
            // The RejectedExecutionException can come from the fact unicastZenPingExecutorService is at its max down in sendPings
            // But don't bail here, we can retry later on after the send ping has been scheduled.
            logger.debug("Ping execution rejected", e);
        }
    });
}
```
pingRequest收到请求后的回调函数，UnicastZenPing.getPingResponseHandler函数

* 该函数逻辑比较简单，把 response 中的 pingResponses 中的节点信息加入到pingCollections 中，也就是最终的返回结果

```
//UnicastZenPing.getPingResponseHandler函数
protected TransportResponseHandler<UnicastPingResponse> getPingResponseHandler(final PingingRound pingingRound,
                                                                               final DiscoveryNode node) {
    return new TransportResponseHandler<UnicastPingResponse>() {

        @Override
        public UnicastPingResponse read(StreamInput in) throws IOException {
            return new UnicastPingResponse(in);
        }

        @Override
        public String executor() {
            return ThreadPool.Names.SAME;
        }

        @Override
        public void handleResponse(UnicastPingResponse response) {
            logger.trace("[{}] received response from {}: {}", pingingRound.id(), node, Arrays.toString(response.pingResponses));
            if (pingingRound.isClosed()) {
                if (logger.isTraceEnabled()) {
                    logger.trace("[{}] skipping received response from {}. already closed", pingingRound.id(), node);
                }
            } else {
                Stream.of(response.pingResponses).forEach(pingingRound::addPingResponseToCollection);
            }
        }

        @Override
        public void handleException(TransportException exp) {
            if (exp instanceof ConnectTransportException || exp.getCause() instanceof ConnectTransportException ||
                exp.getCause() instanceof AlreadyClosedException) {
                // ok, not connected...
                logger.trace(() -> new ParameterizedMessage("failed to connect to {}", node), exp);
            } else if (closed == false) {
                logger.warn(() -> new ParameterizedMessage("failed to send ping to [{}]", node), exp);
            }
        }
    };
}

```

那么pingResponses到底包含什么内容呢？这需要看一个节点收到 pingRequest 后是如何构造 response 的，对应逻辑在UnicastZenPing.handlePingRequest函数：

* 每收到一个 ping 请求，就把发送方节点信息加入到temporalResponses中，下次本节点发送 ping 的时候，会向这些节点发送
* 构造返回的节点信息，节点信息包含了两部分：1. 曾经给自己发送过 ping 请求的节点；2. 节点本身

```
// UnicastZenPing.handlePingRequest
private UnicastPingResponse handlePingRequest(final UnicastPingRequest request) {
    assert clusterName.equals(request.pingResponse.clusterName()) :
        "got a ping request from a different cluster. expected " + clusterName + " got " + request.pingResponse.clusterName();

    /*
     * 每收到一个请求加到 temporalResponses 中
     */
    temporalResponses.add(request.pingResponse);
    // add to any ongoing pinging

    /*
     * 加到 pingingCollection 中
     */
    activePingingRounds.values().forEach(p -> p.addPingResponseToCollection(request.pingResponse));

    /*
     * temporalResponse 中的 response 会定时过期
     */
    threadPool.schedule(TimeValue.timeValueMillis(request.timeout.millis() * 2), ThreadPool.Names.SAME,
        () -> temporalResponses.remove(request.pingResponse));

    /*
     * 返回给请求节点的 responses 集合包含两部分
     * 1. 自己曾经收到ping 请求的节点
     * 2. 本身节点
     */
    List<PingResponse> pingResponses = CollectionUtils.iterableAsArrayList(temporalResponses);
    pingResponses.add(createPingResponse(contextProvider.clusterState()));

    return new UnicastPingResponse(request.id, pingResponses.toArray(new PingResponse[pingResponses.size()]));
}

```

通过以上代码分析，整个过程就比较清晰了，请求的发送和响应都是典型的 Gossip 的模式

* 发送方会向 seedNodes 和曾经ping 过自己的节点发送 ping 请求，发送方包含了本节点的信息
* 接收方会向发送方返回所有 ping 过自己的节点信息

一共进行了3轮 Gossip 的通信，3轮通讯完毕后返回收到的所有节点信息。这三次通信过程收到的节点信息是指数增长的，所以在没有网络分区的情况下，3轮通信就可以获得集群的所有节点信息。为了更直观的说明这个过程，本文举例如下：

* initial state：seedNodes 是 node1，初始状态，每个节点只知道自己和 seedNodes 节点
* round 1： node 1只知道自己，不需发送ping； node 2 向 node 1发送 ping，node 1 知道了 node 2的存在；node 3 向 node 1 发送 ping，node 1 知道了 node 3 的存在
* round 2： node 1 向 node 2， node 3 发送 ping，node 2， node 3知道了 node 1的存在(本身通过 seedNodes 已经知道)；另外，node 2 向 node 1发送 ping，收到的返回包含 node 1，node 2，node 3，所以，node 2 知道了所有的节点。类似的，node 3 向 node 1 发送 ping， 收到的返回包含了 node 1， node 2，node 3，node 3 也知道集群的所有节点信息。
* round 3： 任一个节点向其他两个节点发送ping，ping 中包含本节点信息。任一节点收到 ping 请求后，将三个节点的信息返回给请求节点。

本例中，经过两轮 ping，所有节点已经知道了集群中全部节点信息。

![](/images/post/es_gossip.png)

## 基于集群信息选主

基于集群信息选主分为两步，1. 选出当前的 master 2. 如果 master 不是自己，则给 master发请求加入集群；如果 master 是自己，则等待其他节点给自己发请求。

选出当前 master 节点主要在ZenDiscovery.findMaster 函数中：

* 找到集群已有的 master 节点和 candidate 节点
* 如果当前集群没有 master 节点，并且自己与足够的 candidate 保持连接，则从 candidate 中选出 master，否则返回 null
* 如果集群中有 master 一个或者多个节点，选择 id 较小的节点作为 master

```
//ZenDiscovery.findMaster 函数
private DiscoveryNode findMaster() {
    logger.trace("starting to ping");
    List<ZenPing.PingResponse> fullPingResponses = pingAndWait(pingTimeout).toList();
    if (fullPingResponses == null) {
        logger.trace("No full ping responses");
        return null;
    }
    if (logger.isTraceEnabled()) {
        StringBuilder sb = new StringBuilder();
        if (fullPingResponses.size() == 0) {
            sb.append(" {none}");
        } else {
            for (ZenPing.PingResponse pingResponse : fullPingResponses) {
                sb.append("\n\t--> ").append(pingResponse);
            }
        }
        logger.trace("full ping responses:{}", sb);
    }

    final DiscoveryNode localNode = transportService.getLocalNode();

    /*
     * 没有把自己加到 pingCollections 中
     */
    // add our selves
    assert fullPingResponses.stream().map(ZenPing.PingResponse::node)
        .filter(n -> n.equals(localNode)).findAny().isPresent() == false;

    fullPingResponses.add(new ZenPing.PingResponse(localNode, null, this.clusterState()));

    // filter responses
    final List<ZenPing.PingResponse> pingResponses = filterPingResponses(fullPingResponses, masterElectionIgnoreNonMasters, logger);

    /*
     * 找到当前集群的 master
     */
    List<DiscoveryNode> activeMasters = new ArrayList<>();
    for (ZenPing.PingResponse pingResponse : pingResponses) {
        // We can't include the local node in pingMasters list, otherwise we may up electing ourselves without
        // any check / verifications from other nodes in ZenDiscover#innerJoinCluster()
        if (pingResponse.master() != null && !localNode.equals(pingResponse.master())) {
            activeMasters.add(pingResponse.master());
        }
    }

    /*
     * 找到当前集群的 master candidate
     */
    // nodes discovered during pinging
    List<ElectMasterService.MasterCandidate> masterCandidates = new ArrayList<>();
    for (ZenPing.PingResponse pingResponse : pingResponses) {
        if (pingResponse.node().isMasterNode()) {
            masterCandidates.add(new ElectMasterService.MasterCandidate(pingResponse.node(), pingResponse.getClusterStateVersion()));
        }
    }

    /*
     * 当前集群没有 master
     */
    if (activeMasters.isEmpty()) {
        /*
         * 当前节点与超过 minimum candidate 个节点保持联系
         * 从候选节点中选出 master
         */
        if (electMaster.hasEnoughCandidates(masterCandidates)) {
            final ElectMasterService.MasterCandidate winner = electMaster.electMaster(masterCandidates);
            logger.trace("candidate {} won election", winner);
            return winner.getNode();
        } else {

            /*
             * 无法选出 master 返回 null
             */
            // if we don't have enough master nodes, we bail, because there are not enough master to elect from
            logger.warn("not enough master nodes discovered during pinging (found [{}], but needed [{}]), pinging again",
                masterCandidates, electMaster.minimumMasterNodes());
            return null;
        }
    } else {
        assert !activeMasters.contains(localNode) : "local node should never be elected as master when other nodes indicate an active master";
        // lets tie break between discovered nodes
        /*
         * 集群存在多个 master 节点, 选择 id 较小的当做 master
         */
        return electMaster.tieBreakActiveMasters(activeMasters);
    }
}
```

选出当前 master 后则需要发请求加入集群或等待其他节点加入，主要逻辑在ZenDiscovery.innerJoinCluster 中：

* 如果master 是当前节点，则等待minimumMasterNodes() - 1个节点向自己发送 joinRequest，如果超时则重新开始innerJoinCluster过程
* 如果master 不是当前节点，则向 master 节点发送 joinRequest，阻塞等待，master 会在收集到足够的选票后统一回复

收集 joinRequest 这个事情由nodeJoinController负责，nodeJoinController每收到一个 joinRequest 都会统计一下总数，超过就会触发一个 callBack，阻塞等待这个逻辑就是借助这个 callBack 实现的。这种实现方式感觉各模块的职责划分非常清晰：统计选票的模块只负责统计选票，至于在统计选票过程中，外部模块需要等待，则完全由外部模块通过 callBack 去实现。Es 很多地方都有阻塞等待这种逻辑，不同地方的实现也不尽相同。

```
// ZenDiscovery.innerJoinCluster函数
private void innerJoinCluster() {
    DiscoveryNode masterNode = null;
    final Thread currentThread = Thread.currentThread();
    nodeJoinController.startElectionContext();
    while (masterNode == null && joinThreadControl.joinThreadActive(currentThread)) {
        masterNode = findMaster();
    }

    if (!joinThreadControl.joinThreadActive(currentThread)) {
        logger.trace("thread is no longer in currentJoinThread. Stopping.");
        return;
    }

    /*
     * 当前节点被选为 master
     */
    if (transportService.getLocalNode().equals(masterNode)) {
        final int requiredJoins = Math.max(0, electMaster.minimumMasterNodes() - 1); // we count as one
        logger.debug("elected as master, waiting for incoming joins ([{}] needed)", requiredJoins);
        nodeJoinController.waitToBeElectedAsMaster(requiredJoins, masterElectionWaitForJoinsTimeout,
            new NodeJoinController.ElectionCallback() {
                @Override
                public void onElectedAsMaster(ClusterState state) {
                    synchronized (stateMutex) {
                        joinThreadControl.markThreadAsDone(currentThread);
                    }
                }

                @Override
                public void onFailure(Throwable t) {
                    logger.trace("failed while waiting for nodes to join, rejoining", t);
                    synchronized (stateMutex) {
                        joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
                    }
                }
            }

        );
    } else {
        // process any incoming joins (they will fail because we are not the master)
        nodeJoinController.stopElectionContext(masterNode + " elected");

        // send join request
        /*
         * 此处会阻塞等待,并有多次重试,
         */
        final boolean success = joinElectedMaster(masterNode);

        synchronized (stateMutex) {
            if (success) {
                DiscoveryNode currentMasterNode = this.clusterState().getNodes().getMasterNode();
                if (currentMasterNode == null) {
                    // Post 1.3.0, the master should publish a new cluster state before acking our join request. we now should have
                    // a valid master.
                    logger.debug("no master node is set, despite of join request completing. retrying pings.");
                    joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
                } else if (currentMasterNode.equals(masterNode) == false) {
                    // update cluster state
                    joinThreadControl.stopRunningThreadAndRejoin("master_switched_while_finalizing_join");
                }

                joinThreadControl.markThreadAsDone(currentThread);
            } else {
                // failed to join. Try again...
                joinThreadControl.markThreadAsDoneAndStartNew(currentThread);
            }
        }
    }
}
```