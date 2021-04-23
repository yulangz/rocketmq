
## Status
Current State: Proposed 

Authors: veritas2023@163.com 

Shepherds: vongosling@apache.org

Mailing List Discussion: dev@rocketmq.apache.org

Pull Request: #PR_NUMBER

Released: <released_version>   

Correlations: RIP-18 Metadata management architecture upgrade，[Thought of The Evolution of The Next Decade Architecture for RocketMQ](https://lists.apache.org/thread.html/83a0411c25868a3bc8acf949c8489604a02b76498ded0219959b7912%40%3Cdev.rocketmq.apache.org%3E)

# Background & Motivation

As user business increase ten times at peek times, such as breaking event happens/shopping festival, and decrease after the peek time to regular, which may lead to some issues

1. We have to expand brokers resource in case of storage capacity bottleneck.
2. We have to expand storage capacity in case of broker cpu bottleneck.
3. After expand brokers/storage, we cant shrink these resources, which lead 9 times resource waste.
4. Benchmark test dledger broker-cluster, when leader reach cpu bottleneck, follower only utilize 10% of cpu resource.
5. If store one topic 1 year messages, all topics store 1 year
6. If store one topic 1 year, local storage is expensive
7. If 1 queue consume 100 times, lead to the leader node bottleneck, while other node idle
8. When create new queues, it may choose the busiest broker

## Goals

### What problem is this proposal designed to solve?

Implement the next generation rocketmq architechture, make the boundaries of compute and storage layer specific, support stream and tiered storage.


## Non-Goals

### What problem is this proposal NOT designed to solve?

We will not directly implement multi-raft protocol on RocketMQ.

### Are there any limits of this proposal?

Nothing specific.

## Changes

## Architecture

New architecture graph: 


![](https://codimd.s3.shivering-isles.com/demo/uploads/upload_dd960a30ebdbc1afc85815b46041740c.png)



### broker
* Make clear compute and storage layer interface, focus on compute logic, such as transaction, delay, filter
* Handle produce/consume/admin(rocketmq_protocol) api
* Stateless
* Use the state-of-the-art Angelia rpc call storage read/write api
* Support dynamic add and shrink broker node
* Support read replica customized, crack hot queue read imblance problem

### storage cluster
* Implement storage api for compute node
* Multi-raft implement
* Support hdfs/S3/GCP
* Support dynamic add and shrink storage node
* Storage support multi-tenant, like 100w queue
* Support topic-level retention time

### nameserver cluster
* Hash(queue) to a assigned raft group, which broker determine to call the queue-leader node
* Detect storage node failure, notify to broker update queueTostorageNode map 
* Implement a intellgent module, which use machine learning to decide new queue assign to which node, and detect unhealthy node.![](https://)

## Angelia

* Support multi communication protocol such as tcp, http2
* Support multi serilization protocol such as json, Avro


## compute and store decouple
```
compute layer
SendMessageProcessor#asyncSendMessage
    this.brokerController.getMessageStore().asyncPutMessage(msgInner);


storage layer
DefaultMessageStore#putMessage
    1）commitLog storage
	CommitLog#asyncPutMessage
		mappedFile.appendMessage(msg, this.appendMessageCallback);

    2）Dledger storage
    DLedgerCommitLog#asyncPutMessage
        #io.openmessaging.storage.dledger.DLedgerServer
        dLedgerServer.handleAppend(request);         
            dLedgerStore.appendAsLeader(dLedgerEntry);
            return dLedgerEntryPusher.waitAck(resEntry, false);
    
    3）5.0 seperate storage
    SeperateStorage#asyncPutMessage
        client:appendMessage()  #wait for leader and follower write half nodes
```

## Message Store
Public interface MessageStore {

```
#basic api
putMessage(final MessageExtBrokerInner msg)
flush()
getMessage(final long offset, final int size)
...

#startup and cache for storage node
load()
...

#failover
recover(long maxPhyOffsetOfConsumeQueue)
...

#manage api
getMaxOffset()
...

for si
```

## Compatibility, Deprecation, and Migration Plan
* Are backward and forward compatibility taken into consideration?
the old producer/consumer/admin protocol not change, remains backward compatible

* Are there deprecated APIs?
Remove deprecated pull consumer apis.


## Rejected Alternatives
* How does alternatives solve the issue you proposed?

Use bookkeeper as stream storage engine

* Pros and Cons of alternatives

The advantages and disadvantages of using bookkeeper are as follows

Advantage: Simple implementation and short development cycle

Disadvantages: bookkeeper use zookkeeper store ledger metadata,  The introduction of third-party components requires maintenance of two systems, add maintaince complexity

## Why should we reject above alternatives?

The introduction of third-party components is always avoided by RocketMQ, so the proposal of using bookkeeper(which depends zookkeeper) as metadata storage is not adopted.