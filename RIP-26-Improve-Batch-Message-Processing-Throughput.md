# Status
- Current State: Proposed
- Authors: [Erik1288](https://github.com/Erik1288)
- Shepherds: dongeforever@apache.org
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: #PR_NUMBER
- Released: <relased_version>
- Related Docs: [Improve Batch Message Processing Throughput](https://docs.google.com/document/d/10hlfG6Jg36D8Kid6t8Zuf3g_63pb8yQZqwRqV2GZkEQ/edit)

# Background & Motivation
What do we need to do
- Will we add a new module?    
  No.
- Will we add new APIs?    
  Yes.
- Will we add a new feature?   
  Yes.


Why should we do that

- What scenarios will be covered in this article?

To increase overall message processing throughput, producer accumulates hundreds of messages into a Batch and send them to the Broker at once. Even if the Batch is sent successfully, consumers cannot read the entire Batch at once due to the speed limitation of building ConsumeQueue for sub-messages. End-to-end compression/uncompression is not supported on BatchMessage.

Chinese version:
为了提升整体消息处理吞吐，客户端将几百条消息积攒到一个Batch中，一次性发送到Broker。即使Batch发送成功，由于子消息构建ConsumeQueue速度局限性，消费者也不能立刻读取到整个Batch的数据。当前Batch消息构建ConsumeQueue的方式将决定这个消息处理系统没法支持端到端的Batch压缩。

- Are there any problems with our current project?  

The current version of RocketMQ supports batching feature, where multiple independent messages are accumulated as a BatchMessage and sent to the Broker once. However, the current storage engines do not take full advantage of the Batch feature to reduce consume queue index when processing BatchMessage.
When receiving a BatchMessage, the engine will split the batch into multiple sub-messages.
When indexing a BatchMessage, the engine will construct multi indexes for a single batch.
When consuming a BatchMessage, the engine will access index multi times for a single batch.
End-to-end compression/uncompression is not supported on BatchMessage.
The above problems cause that the throughput cannot be improved and the latency cannot be reduced for batch message processing.

Chinese version:
当前版本的RocketMQ支持将多个独立的消息攒成一个BatchMessage，再发送给Broker的特性。但是目前的存储引擎在处理BatchMessage时，没有充分利用Batch的特性来减少索引。
当处理BatchMessage时，引擎会将一个BatchMessage拆成多个独立的子消息。
当构建BatchMessage索引时，引擎会给一个BatchMessage构建多次索引。
当消费BatchMessage时，引擎会为了查找一个BatchMessage而多次访问索引。
端到端的压缩和解压缩不能被支持，因为Broker不可能为了构建索引而去解压缩。
以上问题导致处理Batch消息时吞吐率无法提高，时延无法降低。


- What can we benefit from proposed changes?  

When Batch Consume Queue is introduced, each BatchMessage is no longer treated as a collection of sub-messages, but as an independent message.
When receiving a BatchMessage, the engine treats a BatchMessage as a normal message.
When indexing a BatchMessage，the engine will construct one index item for a single batch.
When consuming a BatchMessage，the engine will access index only once for a single batch.
End-to-end compression/uncompression will be supported on BatchMessage on the fly.

Chinese version:
当Batch Consume Queue这个索引被引入后，每一个消息不再被当成多个子消息的集合，而是当成一个独立的消息。
当处理BatchMessage时，引擎会将一个BatchMessage当成一个普通消息对待。
当构建BatchMessage索引时，引擎会给一个BatchMessage构建一次索引。
当消费BatchMessage时，引擎会为了查找一个BatchMessage只会查找一次索引。
端到端的压缩和解压缩将天然被支持，因为服务端无须对Batch进行任何处理。


# Goals
- What problem is this proposal designed to solve?  
  1. Introducing BatchConsumeQueue index to improve throughput for BatchMessage.
  2. Refactor consume queue interface.
# Non-Goals.
- What problem is this proposal NOT designed to solve?  
  Convert index from ConsumeQueue to BatchConsumeQueue.
- Are there any limits of this proposal?  
  Only newer clients with changes in this proposal will benefit.

# Changes
## Architecture

### Current "Consume Queue index":

- ConsumeQueue Index Item (20 Bytes)

- Indexing


As shown in the figure above, BatchMessage1 will be split into 3 separate sub-messages. Every single sub-message needs to be indexed on consume queue.

### Proposed "Batch Consume Queue index"

BatchConsumeQueue Index Item (46 Bytes)

Indexing

As shown in the figure above, BatchMessage1 will be treated as a single message, and will be indexed only once.


## Interface Design/Change
- Method signature changes

  Nothing specific.

- Method behavior changes

  Nothing specific.

- CLI command changes

  Nothing specific.

- Log format or content changes
  Nothing specific.

## Compatibility, Deprecation, and Migration Plan
- Are backward and forward compatibility taken into consideration?
  
  Inner-batch: new batch message processing.
  Outer-batch: old batch message processing.
  old client+new broker: old clients won't make request with inner-batch flag, so broker will use old outer-batch manner.
new client+old broker: new clients will detect whether broker support the inner-batch manner, if not, it will fallback to use old outer-batch manner.

- Are there deprecated APIs?
  
  No.
- How do we do migration?
  
  new instance: migration is not needed.
  old instance: historical consume queue needs to be converted to batch consume queue.

## Implementation Outline
We will implement the proposed changes by 2 phases.
### Phase 1 
Implement client-side and server-side batch consume queue in broker.
### Phase 2 
Implement historic consume queue and batch consume queue converter.

# Rejected Alternatives
- How does alternatives solve the issue you proposed?
  
  Sparse index.
- Pros and Cons of alternatives
  
  Pros: space-saving
  Cons: lower throughput
- Why should we reject the above alternatives
  
  Starting with 5.0, RocketMQ will focus on streaming, which gives higher priority to throughput.