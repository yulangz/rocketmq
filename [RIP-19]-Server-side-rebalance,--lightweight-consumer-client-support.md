# Status


- Current State: Proposed
- Authors: [ayanamist]()
- Shepherds: [duhengforever]()
- Mailing List discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org); [users@rocketmq.apache.org](mailto:users@rocketmq.apache.org)
- Pull Request: RIP-19
- Released: - 5.0.0-PREVIEW



<a name="4050cdb7"></a>
# Background & Motivation


<a name="985d5680"></a>
### What do we need to do


- Will we add a new module?<br />No.
- Will we add new APIs?<br />Yes.
- Will we add new feature?<br />Yes.



<a name="14982109"></a>
### Why should we do that


- Are there any problems of our current project?<br />The current subscription load balancing strategy is based on the dimension of message queue. All behaviors are owned by the client side. There are three main steps:
   1. Each consumer regularly obtains the total number of topic message queues and all consumers.
   1. Using a general algorithm to sort the queues by consumer ip and queue index to calculate which message queue is allocated to which consumer.
   1. Each consumer pulls messages using allocated orders described above.


<br />According to this allocation method, if an abnormality occurs in a consumer (the application itself is abnormal, or a broker is upgrading) so that it causes slow subscription, messages will be accumulated, but this queue will not be re-allocated to another consumer, so the accumulation will become more and more serious.<br />Chinese version:<br />当前的消费负载均衡策略是以队列的维度来进行，所有行为全部是由客户端主动来完成，主要分为三步：

   1. 每个consumer定时去获取消费的topic的队列总数，以及consumer总数
   1. 将队列按编号、consumer按ip排序，用统一的分配算法计算该consumer分配哪些消费队列
   1. 每个consumer去根据算法分配出来的队列，拉取消息消费


<br />按照这个分配方式，如果有一个队列有异常（应用自身异常，或某个broker在升级）导致消费较慢或者停止，该队列会出现堆积现象，因为队列不会被分配给其他机器，因此如果长时间不处理，队列的堆积会越来越严重。

- What can we benefit proposed changes?<br />The accumulated messages will be subscribed by other consumers if one consumer behaves abnormally.<br />Chinese version:<br />在某个队列消费异常的情况下，可以快速的由其它消费者接手进行消费，缓解堆积状态。



<a name="Goals"></a>
# Goals


- What problem is this proposal designed to solve?<br />The accumulated messages will be subscribed by other consumers if one consumer behaves abnormally.<br />Chinese version:<br />在某个队列消费异常的情况下，可以快速的由其它消费者接手进行消费，缓解堆积状态。
- To what degree should we solve the problem?<br />This RIP must guarantee below point:
   1. High availablity: Subscription of one message queue will not be affected by single consumer failure.
   1. High performance: This implementation affects latency and throughput less than 10%.


<br />Chinese version:<br />新方案需要保证两点：

   1. 高可用：单一队列的消费能力不受某个消费客户端异常的影响
   1. 高性能：POP订阅对消息消费的延迟和吞吐的影响在10%以内



<a name="Non-Goals"></a>
# Non-Goals


- What problem is this proposal NOT designed to solve?<br />Improve client-side load balancing.
- Are there any limits of this proposal?<br />Nothing specific.



<a name="Changes"></a>
# Changes


<a name="Architecture"></a>
## Architecture

<br />Current "Pull mode":<br />![](https://user-images.githubusercontent.com/406779/103756075-cc93b900-5049-11eb-8fae-cfe5398ebaad.png#align=left&display=inline&height=679&margin=%5Bobject%20Object%5D&originHeight=679&originWidth=1207&status=done&style=none&width=1207)<br />
<br />Proposed "Pop mode":<br />![](https://user-images.githubusercontent.com/406779/103757230-6d36a880-504b-11eb-95d5-7e8cff8cdef1.png#align=left&display=inline&height=675&margin=%5Bobject%20Object%5D&originHeight=675&originWidth=1217&status=done&style=none&width=1217)<br />
<br />Move inter-queue balance of one topic from client side to server side. Clients make pull request without specified queues to broker, and broker fetch messages from queues internally and returns, which ensures one queue will be consumed by multiple clients. The whole behavior is like a queue pop process.<br />
<br />It will add a new request command querying queue assignments in broker, and add pop-feature-support flag to pull request which makes broker use pop mode.<br />

<a name="1d452184"></a>
## Interface Design/Change


- Method signature changes<br />Nothing specific.
- Method behavior changes<br />Nothing specific.
- CLI command changes<br />Add `setConsumeMode` for admin to switch between old pull mode and new pop mode for one subscription.
- Log format or content changes<br />Nothing specific.



<a name="b0b50ec3"></a>
## Compatibility, Deprecation, and Migration Plan


- Are backward and forward compatibility taken into consideration?<br />New RequestCode between client and broker are added, so there are 2 compatibility situations:
   1. old client+new broker: old clients won't make request with pop-feature-support flag, so broker will not enable pop mode, which keep all things as before.
   1. new client+old broker: new clients will detect whether broker support the new request command querying queue assignments, if not, it will fallback to use old pull mode.
- Are there deprecated APIs?<br />Nothing specific.
- How do we do migration?<br />Nothing specific.



<a name="80d48d58"></a>
## Implementation Outline

<br />We will implement the proposed changes by **2** phases.<br />

<a name="ee4650e5"></a>
## Phase 1


1. Implement server-side balance capability in broker
1. Implement client-side request using new pop-mode



<a name="56e0338b"></a>
## Phase 2


1. Implement new sdk compatibility with old broker.
1. Implement feature detection in broker and client.



<a name="434c271d"></a>
# Rejected Alternatives


<a name="3030f949"></a>
## How does alternatives solve the issue you proposed?

<br />Improve client rebalance logic? I don't get a quite good idea.<br />

<a name="1b4f112a"></a>
## Pros and Cons of alternatives

<br />Client rebalance logic will become quite complicated.<br />

<a name="fe22d296"></a>
## Why should we reject above alternatives
