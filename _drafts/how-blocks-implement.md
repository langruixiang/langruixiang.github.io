---
layout: post
title: 实现阻塞等待的几种写法
categories: [Java]
description: 实现阻塞等待的几种写法
keywords: java，blocking
---

分布式编程中经常遇到的一个场景是当前线程需要阻塞等待至某个条件发生然后继续运行，分享一下看到的几种写法和对应场景。

# sleep 写法

场景：调用了一个异步接口，需要不断调用状态查询接口检查对应异步流程的状态，根据状态决定是否停止阻塞：

* SUCCESS：成功，停止阻塞，执行成功对应的业务逻辑
* FAIL：失败，停止阻塞，执行失败对应的业务逻辑
* DOING：正在执行中，休眠一段时间继续查询
* 异常：调用查询接口出现异常，忽略，休眠一段时间继续查询

整个阻塞过程最多阻塞 TIMEOUT_PERIOD 时间，超时则返回执行对应业务逻辑

```
long startTime = System.currentTimeMillis();


while (true) {
    try {
        // 调用状态查询接口
        String status = describeStatus();
        
        if(isSuccess()) {
            return success;
        }else if(isFaile()) {
            return fail;
        }else if(isDoing()) {
            logger.info("task is doing, try again...");
        }
    // 状态查询接口异常    
    } catch (Exception e) {
        logger.info("describeStatus error, try again...", e);
    }
      
    long currentTime = System.currentTimeMillis();
    if (currentTime - startTime > TIMEOUT_PERIOD) {
        throw new TimeoutException(time out");
    }
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
         Thread.currentThread().interrupt();
    }
}

```

# wait notify写法

场景：摘自 zookeeper 实现，leader 和 所有 folower 等待过半的 folower 返回 ack 后继续往下运行

```
public void waitForEpochAck(long id, StateSummary ss) throws IOException, InterruptedException {
        // 不能忘记加锁
        synchronized (electingFollowers) {
            if (electionFinished) {
                return;
            }
            if (ss.getCurrentEpoch() != -1) {
                if (ss.isMoreRecentThan(leaderStateSummary)) {
                    throw new IOException("Follower is ahead of the leader, leader summary: "
                            + leaderStateSummary.getCurrentEpoch()
                            + " (current epoch), "
                            + leaderStateSummary.getLastZxid()
                            + " (last zxid)");
                }
                if (ss.getLastZxid() != -1 && isParticipant(id)) {
                    electingFollowers.add(id);
                }
            }
            QuorumVerifier verifier = self.getQuorumVerifier();
            
            // 满足条件，停止阻塞
            if (electingFollowers.contains(self.getId()) && verifier.containsQuorum(electingFollowers)) {
                electionFinished = true;
                electingFollowers.notifyAll();
            // wait休眠，继续检查条件
            } else {
                long start = Time.currentElapsedTime();
                long cur = start;
                long end = start + self.getInitLimit() * self.getTickTime();
                while (!electionFinished && cur < end) {
                    electingFollowers.wait(end - cur);
                    cur = Time.currentElapsedTime();
                }
                if (!electionFinished) {
                    throw new InterruptedException("Timeout while waiting for epoch to be acked by quorum");
                }
            }
        }
    }

```
