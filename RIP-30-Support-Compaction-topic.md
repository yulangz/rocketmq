# Status
   - Current Status: Draft
   - Authors: ltamber <https://github.com/ltamber>
   - Shepherds: duhengforever <duhengforever@apache.org>
   - Mailing List discussion: dev@rocketmq.apache.org
   - Pull Request: #PR_NUMBER
   - Released: <released_version>

# Background & Motivationwhat do we need to do
   - will we add a new module? *no*.
   - will we add new APIs? *no*.
   - will we add a new feature? *yes*.

# Why should we do that

   - Are there any problems of our current project?

Currently, Apache RocketMQ does not support the compaction topic, which results in
that if the application(such as connector/streams[1]
<https://lists.apache.org/thread/c85oj18whdjnxc331n1fxqhb8z50bpnm>) uses
messages to store state and only need the latest state data at a certain
time, it needs to pull all the messages from the broker, this process will
consume a lot of time; At the same time, a lot of data is not required by
the client. Therefore, we need a special data cleaning mechanism, which is
to compress the same type of data (same key), and only keep the latest
data, In this way, only a small amount of data is loaded when the
application starts.

Chinese version: RocketMQ 目前还不支持compation
topic，这导致应用如果使用消息来存储状态并且在某一时刻仅需要最新的状态数据时(例如connector/streams)
需要从服务端拉取所有的消息，这个过程会消耗大量时间，同时很多数据也不是客户端需要的，因此，我们需要一个特殊的数据清理机制，就是将相同类型的数据(相同的key)进行压缩，仅保留最新的数据，以此来达到应用启动时仅加载少量数据的诉求。
- What can we benefit proposed changes?

Improve the business scenarios that rocketmq can support

Chinese version: 完善rocketmq能支持的业务场景

# Goals

## What problem is this proposal designed to solve?

Support compress topic data on the server-side.

Chinese version: 支持服务端对topic数据进行压缩

   - To what degree should we solve the problem?

Supports compression of topic data by time dimension and message size
dimension

Chinese version: 支持按时间维度、消息大小维度对topic数据进行压缩

# Non-Goals

   - What problem is this proposal NOT designed to solve?
      Nothing specific.
   - Are there any limits of this proposal?
      Nothing specific.

## Changes Architecture Write

The data is first written to the commitlog like ordinary messages, and then
asynchronously written to the separate compaction log file by the report
service.

Chinese version: 数据与普通消息一样先写入commitlog，然后由reput
service异步再写入到单独的compaction日志文件

- Compaction

   1. determine the compaction position
   2. Traverse data to generate <key, offset> Map


   1. Traverse the data according to the <key, offset> Map and keep only
   the latest data for the same key

- Chinese version:

   1. 先确定compaction结束位置
   2. 遍历数据生成<key, offset> Map


   1. 根据<key, offset> Map再遍历数据将相同key的数据仅保留最新的

- Index

refer to

https://github.com/apache/rocketmq/wiki/RIP-26-Improve-Batch-Message-Processing-Throughput
Interface Design/Change

   - Method signature changes. *No*
   - Method behavior changes. *No*


   - CLI command changes. *No*
   - Log format or content changes. *No*

# Compatibility, Deprecation, and Migration Plan

   - Are backward and forward compatibility taken into consideration?

Nothing specific.

   - Are there deprecated APIs?
      Nothing specific.
   - How do we do migration?
      Nothing specific.