# Status
- Current State: PR Review
- Authors:  fujian-zfj
- Shepherds: fuyou001, hill007299
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: https://github.com/apache/rocketmq/pull/7065
- Released: <relased_version>


# Background & Motivation
## What do we need to do

- Will we add a new module?

No new module will be added.

- Will we add new APIs?

No additions or modifications to any client-level and admin-tool APIs.
There will be some new interfaces and APIs.

- Will we add a new feature?

Yes, KV(Rocksdb) storage is a new feature.

## Why should we do that

- Are there any problems of our current project?

Yes, and we have encountered and been plagued by these probelms in the actual production environment. In some actual production scenarios, millions of topics and subsriptions will be created, and both of them may be deleted and created frequently. In current architecture, both topics and subcriptions are persisted in real time, which means each request of topics and subsciptions updating will trigger the persist interface, and the persistence of such metadata is written in full rather than append-only. 
 
In the scenario of millions of topics, frequent persistence generates the large memory object jsonString of the topicConfigTable. When the memory is tight, the large memory object jsonString will be directly allocated to the old generation, resulting in frequent Full GC.

In addition, millions of topics will inevitably bring millions of indexed ConsumeQueue small files, a large number of indexed small files will destroy the advantages of rocketmq sequential writing, and the random writing of ConsumeQueue will make the performance drop sharply.

- What can we benefit proposed changes?

KV(Rocksdb) storage can bring the following enhancements to the storage layer:

  a. Support metadata at the level of millions, and the additional writing of rocksdb avoids the Full GC that may be caused by large object memory allocation when metadata (topic, subscription, consumerOffset) is persisted.

  b. Support millions of ConsumeQueue index files, avoiding the sharp drop in performance caused by random writing of millions of index files

# Goals

- What problem is this proposal designed to solve?  

  a. Support metadata at the level of millions, and the additional writing of rocksdb avoids the Full GC that may be caused by large object memory allocation when metadata (topic, subscription, consumerOffset) is persisted.

  b. Support millions of ConsumeQueue files, avoiding the sharp drop in performance caused by random writing of millions of ConsumeQueue files

# Non-Goals.

- What problem is this proposal NOT designed to solve? 
 
Nothing specific.

# Changes
## Architecture

This proposal mainly optimizes and solves the existing perfermance problems in the million-topic scenario from two levels:

1. Metadata like topic, subscription, consumerOffset implement kv storage

2. ConsumeQueue index file implements kv storage

### Metadata KV(Rocksdb) Storage

As shown in the figure below, Topic, Subscription and ConsumerOffset all have such a ConfigManager. The characteristics of the two types of metadata, Topic and Subscription, are not allowed to be lost, which is why the current architecture calls the persistent interface every time an update request is made. Therefore, in the rocksdb storage, we enable the write-ahead log(WAL), so that the metadata will be first written into the WAL, and the data will not be lost even if the node crashes abnormally.

![image](https://github.com/apache/rocketmq/assets/21963954/17090157-f98a-45cb-b3d1-4a30e3437df4)

### ConsumeQueue Index File KV(Rocksdb) Storage

Unlike Topic and Subscription, which cannot tolerate data loss and enable write-ahead log(WAL), the ConsumeQueue index file can tolerate data loss after node crashes abnormally, because after the broker restarts, it will try to recover the commitlog and rebuild the missing ConsumeQueue.

Different from the current CqUnit which only stores triplets phyoffset, msgSize, tagsCode, we also record storeTime in KV storage for querying messages in the time dimension.

![image](https://github.com/apache/rocketmq/assets/21963954/e52f2491-0b43-4a59-8770-4b264fe61817)

## Compatibility, Deprecation, and Migration Plan

Regardless of whether the rocksdb switch is turned on or off, the original RocketMQ client and message sending and receiving will not be affected.

## Implementation Outline

We split this proposal into several tasks:

1. Task1: Implement TopicConfigManager, SubscriptionConfigManger, ConsumerOffsetManager in rocksdb mode

2. Task2: Add ConsumeQueueStoreInterface and implement RocksdbConsumeQueueStore

3. Task3: Add AbstractConsumeQueue and implement RocksdbConsumeQueue

4. Task3: Implement RocksdbMessageStore

# Rejected Alternatives

No other alternatives
