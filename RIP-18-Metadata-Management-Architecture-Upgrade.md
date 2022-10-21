# Status

- Current State: Proposed
- Authors: [humkum](https://github.com/humkum), [ferrirW](https://github.com/ferrirW)
- Shepherds: duhengforever@apache.org, vongosling@apache.org
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: #PR_NUMBER
- Released: <relased_version>

# Background & Motivation

Currently, the metadata consistency of RocketMQ is maintained by full connection. For example, each broker registers with each nameserver to ensure that the view of routing information seen between nameservers is the same, and each the consumer instance sends heartbeats (carrying subscription information) to broker to ensure that the view of subscription information seen between brokers is the same. However, such consistency maintenance is weak. Unreliable network and delay may cause inconsistent views, which may lead to some issues

1. In case of network partition or network instability, the broker fails to register with some nameservers. At this time, the metadata between the nameservers is inconsistent, but each nameserver still provides services.

2. Consumer rebalance needs to obtain routing information (topic-queue list) and subscription information (cid list), the consumer instance will obtain the routing information from any nameserver, and will also obtain subscriptions from any broker (including the topic), the inconsistency of the view in consumer instance will lead to unbalanced load.

3. Some metadata needs strong consistency, such as the binding information of queues in elastic scaling, while the current architecture cannot meet the strong consistency requirements.

On the other hand, after RocketMQ 4.5.0, we have used the Raft protocol (DLedger) to solve the consistency problem of log replication. DLedger is a raft-based log storage library. At the beginning of the design, we hoped to apply it to consistent metadata storage. If the metadata of RocketMQ is stored as log and the consistency is guaranteed by using the raft protocol (dledger), the issue of metadata consistency will be solved, and it has at least the following advantages:

1. Do not introduce third-party components. The introduction of third-party components such as Etcd or ZooKeeper will make RocketMQ need to maintain multiple systems, increasing the cost of O & M and fault diagnosis.  Moreover, most of the third-party components use log to build the system state, and RocketMQ is a shared log. It is natural to complete metadata consistency on it.

2. Metadata as log has some advantages: the change of data can be incremental, reducing the network overhead. Failover will not take a long time to reload (we can use snapshot or compact topic to speed up loading).  Avoid some complicated situations caused by directly operating the memory.

3. From a security point of view, the current architecture may have a bad impact on the stability of the nameserver under very large clusters. Metadata as a log can solve these issues and facilitate subsequent expansion.

4.Strong consistent storage can provide more powerful features, such as distributed lock, election and so on, which provide development basis for new architecture evolution of RocketMQ.

# Goals

- What problem is this proposal designed to solve?

Continuously simplify RocketMQ architecture.
Provides consistent metadata management capabilities.
Enhance the horizontal expansion ability of metadata management service (nameserver).

- To what degree should we solve the problem?

The goal of the first stage is to upgrade the nameserver. The new nameserver has the following characteristics:

1. Consistency semantic support
The stronger consistency semantic guarantee also means the sacrifice of this performance. Therefore, nameserver provides different consistency semantics for different metadata. For example, routing information does not require strong consistency guarantees, and nameserver support eventual consistency. Some metadata, such as the binding information during queue expansion and contraction, support linearizable semantics.

2. High availability
The Raft protocol (DLedger) ensures the high availability of nameserver and can tolerate the failure of minor nodes.

3. High performance and easy expansion
DLedger is a high-performance CommitLog storage library. Building consistent metadata storage on the basis of DLedger ensures performance. In addition, the sharding can be used to enhance the expansion capability to ensure the stability of the nameserver.

# Non-Goals

- What problem is this proposal NOT designed to solve?

We will not directly implement raft protocol on RocketMQ.

- Are there any limits of this proposal?

Nothing specific.

# Changes

## Architecture

After RocketMQ 4.5.0, DLedger was used for log replication of RocketMQ, providing strong consistency and automatic failover capability for RocketMQ, and the performance and stability of DLedger were also verified. In the same way, this proposal plans to store metadata as log in dledger.

![](https://s1.ax1x.com/2020/04/30/JqQuyd.png)

As shown in the figure above, nameserver records the operation of metadata in the DLedger log. Leader will replicate the logs to the follower in the same order. Each nameserver will consume these logs in same order and build the system state of metadata. Compared with the original nameserver, this architecture has the following advantages:

1. The log establishes a clear ordering between metadata operations, and ensures that the nameservers always move along a single timeline. 

2. The metadata update is incremental. The new nameserver can send incremental data instead of full registration information, reducing network overhead and nameserver CPU consumption.

3. The offset in the log can naturally be used as the version, indicating the freshness of the current metadata state. We can use offset to quantify the synchronization progress between nameservers.

## Log Format

![](https://s1.ax1x.com/2020/04/30/JqlnhT.png)
	
As shown in the figure above, metadata operations will be recorded in the DLedger log. This proposal does not specify the log format, but it contains at least the following fields:

- Tpye: the type of operation on metadata.
- Key and Vaule:  metadata key and vaule (corresponding content to operate).

The Metadata log example is shown below:

![](https://s1.ax1x.com/2020/04/30/JqlW8S.png)


## Metadata State Building

DLedger has guaranteed the consistency of log replication. In addition to log replication, each nameserver needs to consume logs in order to build the system state of metadata.

As shown in the figure below, the metadata log has three important offsets. The end offset is the offset of the last log entry, the commit offset is the maximum offset of log entry that has been confirmed to be replicated to major nodes, and the apply offset  is the maximum offset of log entry that has been built into the metadata state. The three offsets must satisfy apply offset <= commit offset <= end offset.

![](https://s1.ax1x.com/2020/04/30/Jq17sH.png)

In addition, each nameserver will have a apply thread to process the log entries in order to build the status of the metadata. Another snapshot thread periodically stores the metadata snapshot to disk.

## Interactions

There are two main changes to the interaction between the client, broker and nameserver:

1. Connection 
The original nameservers are equivalent to each other, and the client or broker will randomly select a nameserver node to connect. The new nameserver has the Leader and Follower, so the connection logic will be changed. 

2. Request processing
The original nameserver handles the write requests by manipulating the data in the memory , and  the new nameserver handles the write requests by appending the log.

This proposal maintains backward compatibility. The interaction between the components and new nameserver is as follows

Client interactions: The interaction between the nameserver and the old clients (Producer, Consumer, Admin Client) remains backward compatible. The write request sent to the follower will be forwarded to the leader, and the old write request code would be interpreted as a batch-record appends.

Broker interactions: The interaction with the old Broker remains backward compatible.  The write request sent to the follower will be forwarded to the leader, and the old write request code would be interpreted as a batch-record appends.

## Compatibility, Deprecation, and Migration Plan

- Are backward and forward compatibility taken into consideration?

The consistency metadata store (nameserver) in this proposal remains backward compatible. The old client and broker can still connect to the new nameserver, but for a good experience, it is recommended to upgrade to the new client and broker.

- Are there deprecated APIs?

Nothing.

- How do we do migration?

This proposal cannot replace the nameserver during operation, so the old cluster needs to be shut down to switch new nameservers.

# Rejected Alternatives 

- How does alternatives solve the issue you proposed?

Use Etcd or ZooKeeper as metadata storage to solve the metadata consistency issue.

- Pros and Cons of alternatives

The advantages and disadvantages of using Etcd or ZooKeeper as metadata storage are as follows

Advantage: Simple implementation and short development cycle

Disadvantages: The introduction of third-party components requires maintenance of two systems, which increases the cost of O & M and fault diagnosis. A large amount of code on the client and server needs to be modified, and backward compatibility is not possible.

- Why should we reject above alternatives?
	
The introduction of third-party components is always avoided by RocketMQ, so the proposal of using Etcd or ZooKeeper as metadata storage is not adopted.
