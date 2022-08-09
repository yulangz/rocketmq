# Status

* Current State: Proposed {Proposed, Discussing, Voting, Implementing, Implemented, Discard}
* Authors: [https://github.com/lizhiboo](https://github.com/lizhiboo)
* Shepherds: duhengforever [duhengforever@apache.org](mailto:duhengforever@apache.org)
* Mailing List discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)
* Pull Request: #PR_NUMBER
* Released: <relased_version>

# Background & Motivation

### What do we need to do

* Will we add a new module?
  No

* Will we add new APIs?
  No

* Will we add new feature?
  Yes

### Why should we do that

* Are there any problems of our current project?
  Although RocketMQ has capability of data replication between RocketMQ clusters using RmqSourceReplicator, RmqSourceReplicator does not have at least once delivery semantic. RmqSourceReplicator updates the next pull offset immediately after the current pull action. Besides, RmqSourceReplicator replicate all topics from source cluster to destination cluster. We need configurable topic replication connector, RocketMQSourceConnector.

RmqSourceReplicator undertakes many additional responsibilities, such as configuration replication. We can separate configuration replication function from RmqSourceReplicator to RocketMQConfigConnector Connectors.

Replication links connectivity check is needed for health check. 

虽然RmqSourceReplicator可以实现RocketMQ集群之间的数据复制，但是RmqSourceReplicator不能保证 最小一次投递 的语义。RmqSourceReplicator在拉取数据后立即更新位点。另外，RmqSourceReplicator默认是复制源端集群全量的topic到目标端集群。我们需要一个可配置topic复制链路的connector, RocketMQSourceConnector。

RmqSourceReplicator承担了太多附加在的功能，例如配置同步。我们可以将配置同步功能从RmqSourceReplicator拆分出来到RocketMQConfigConnector中。

* What can we benefit proposed changes?
  Replicate data from one RocketMQ cluster to another in at least once delivery semantic.

从一个RocketMQ集群复制数据到另外一个RocketMQ集群要保持 最少一次投递 的语义。

# Goals

* What problem is this proposal designed to solve?
  Data replication enhancement.

提升数据复制的能力。

* To what degree should we solve the problem?
  At least once delivery.

保证最少一次投递的语义。

# Non-Goals

* What problem is this proposal NOT designed to solve?
  Exactly once delivery.

精准一次投递。

* Are there any limits to this proposal?
  Depends on RocketMQ connector architecture.

需要依赖RocketMQ connector框架。

# Changes

## Architecture

RocketMQ Replicator can replicate  messages from Primary RocketMQ cluster to Standby RocketMQ cluster based on RocketMQ connector architecture. Usually RocketMQ connector architecture runs in the destination cluster. Business system can support disaster recovery easily by using RocketMQ Replicator. 

RocketMQ Replicator能够基于RocketMQ connector架构从主RocketMQ集群复制消息到备RocketMQ集群。通常RocketMQ connector架构运行在目标端集群。业务系统能够很容易的实现容灾恢复通过RocketMQ Replicator。





## Interface Design/Change

**Add 2 method in SourceTask:**

    public boolean dismissFailedMsgs(final List<ConnectRecord> failedRecords) {

        // just log

        return false;

    }

    public boolean customerFailedMsgs(final List<ConnectRecord> failedRecords) {

        // just log

        return false;

    }

These 2 methods used for notify SubSourceTask to conduct of messages that sink failed to RocketMQ.

在SourceTask类中新增2个方法：

这两个方法用于通知SourceTask的具体实现子类来处理写入到RocketMQ失败的消息。

**Add 3 different type connectors:**

RocketMQSourceConnector, RocketMQHeartbeatConnector, RocketMQConfigConnector.

新增四种类型的connector：RocketMQSourceConnector, RocketMQHeartbeatConnector,  RocketMQCheckpointConnector, RocketMQConfigConnector.

### Replication Topic

Define the topic in the primary cluster as Source Topic and the topic in the standby cluster as Destination Topic  in one replication link. Two replication links can be built on a couple of Source Topic and Destination Topic, such as replicate Topic-P to Topic-S as well as replicate Topic-S to Topic-P. Using two couples of Source Topic and Destination Topic for two replication links is also allowed.

在一个复制链路中，定义在主机房的topic作为源topic，在备机房的topic作为目标topic。两条复制链路可以建议在一对源topic和目标topic上，比如从Topic-P复制到Topic-S，并且从Topic-S复制到Topic-P。 使用两对源topic和目标topic构建两条复制链路也是可以的。

### Connector

**RmqSourceReplicator** used for data replication of cluster. Each RmqSourceReplicator task can replicate source cluster's all topics data to destination cluster. We can simply RmqSourceReplicator functions, moving configuration replication to RocketMQConfigConnector.

**RocketMQSourceConnector** used for data replication for user topics. Each RocketMQSourceConnector task can replicator one or more topics in source cluster to one topic in destination cluster.

**RocketMQHeartbeatConnector** used for connectivity check from primary RocketMQ cluster to standby. RocketMQHeartbeatConnector runs in standby cluster,  produces a message to rmq_sys_replicator_heartbeat inner topic in the primary cluster every 10 seconds, and then consumes from rmq_sys_replicator_heartbeat topic in the primary cluster, produces a message to rmq_sys_replicator_heartbeat topic in the standby cluster finally.

**RocketMQConfigConnector** used for topic configuration replication, such as queue number, acl config, etc.

**RmqSourceRepicator**用于用户topic的数据复制。每个RmqSourceReplicator任务能够源集群的所有topic数据到目标集群中。我们可以简化RmqSourceReplicator功能，将配置同步功能挪到RocketMQConfigConnector中。

**RocketMQSourceConnector**用于用户topic的数据复制。每个RocketMQSourceConnector任务可以复制一个或多个源集群的topic到一个目标集群的topic。

**RocketMQHeartbeatConnector**用于主集群到备集群的连通性检查。RocketMQHeartbeatConnector运行在备机房中，每10秒生产一条消息到主机房的rmq_sys_replicator_heartbeat内部topic中，然后从主机房的rmq_sys_replicator_heartbeat中消费消息，最后生产消息到备机房的rmq_sys_replicator_heartbeat中。

**RocketMQConfigConnector**用于topic配置复制，例如队列数，acl信息等。

### Delivery Semantic

RocketMQ Replicator 2.0 guarantees at least once delivery. Each message will be replicated to destination cluster at least once, that is some messages may be replicated more than once. The reason is that RocketMQSourceConnector will reconsume source cluster messages when some RocketMQ connector workers crash.

RocketMQ Replicator 2.0保证最少一次投递。每条消息至少被同步一次到目标集群，也就是说一些消息可能会被复制多次。因为在RocketMQ connector的worker宕机的情况下，RockerMQSourceConnector将会重复消费源端集群的消息。

### Infinite replication check

The message that was born in one RocketMQ cluster such as cluster-A and then replicated to another RocketMQ cluster such as cluster-B is not allowed to be replicated again from cluster-B to cluster-A. Adding system property ROCKETMQ_REPLICATOR_SOURCE to replicated message properties indicates where the message was born, thus avoiding infinite replication.

一条消息生产在集群A中，之后被复制到另外一个集群B中，这条消息是不允许再被从集群B复制到集群A中。在被复制的消息属性中增加系统属性ROCKETMQ_REPLICATOR_SOURCE来标识消息来源，这样可以避免消息无限循环复制。

##  Compatibility, Deprecation, and Migration Plan

* Are backward and forward compatibility taken into consideration?
  No

* Are there deprecated APIs?
  No

* How do we do migration?
  No

## Implementation Outline

We will implement the proposed changes by **3** phases. 

## Phase 1

1. Design RocketMQSourceConnector
2. Coding

## Phase 2

1. Design RocketMQHeartbeatConnector
2. Coding

## Phase 3

1. Design RocketMQCheckpointConnector
2. Coding 

# Rejected Alternatives 

## How does alternatives solve the issue you proposed?

## Pros and Cons of alternatives

## Why should we reject above alternatives