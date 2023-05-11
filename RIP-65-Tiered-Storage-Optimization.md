# Status

- Current State: Draft
- Authors:  lizhimins
- Shepherds: aaron-ai
- Mailing List discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)
- Pull Request:
- Released: <relased_version>

# Background & Motivation

## What do we need to do

- Will we add a new module? No.
- Will we add new APIs? Yes.
- Will we add a new feature? No.

## Why should we do that

- Are there any problems with our current project?

I have reviewed the code related to tiered storage and found that the existing model and implementation are very messy. And I hope to improve the code quality of the tiered storage. Actually turned out to hava a couple of small bugs in it too.

- What can we benefit from proposed changes?

Better code quality with better readability and maintainability, and fix some bugs.

# Goals

- What problem is this proposal designed to solve?

Improve the code quality of the tiered storage and fix some bugs.

- To what degree should we solve the problem?

The first stage will involve some API modifications, and the second stage will improve the internal threading model and code details.

# Non-Goals

- What problem is this proposal NOT designed to solve?

The data model and format of tiered storage will not change.

# Changes

These modifications include:

1. Model abstraction

   1. Remove abstract coupling: In the original design, the concept of MessageQueue is strongly coupled with all models. For example, the abstraction of File and File Queue in the local default message store combines multiple mmap files into a linked list, which belongs to the file system side and is irrelevant to the MessageQueue concept. Memory-based objects like FileSegment and FileSegmentQueue should only care about the file path (filePath) and not have their design influenced by other components. This design also disrupts the code hierarchy in TieredIndexFile, and the mismatch between the persistent model and the actual running model is not user-friendly.
   2. Encapsulation, god class, and observability issues: In the original implementation, FileSegment and FileSegmentQueue were the smallest granularity data managers and should not mix metadata store and attribute information into these simple objects, which is a serious breach of encapsulation. For example, if a class has a counter and needs to expose that value to other classes, it should not pass the upper-level object down but rather expose the current value. The implementation of offset management should looks like messageStore.getMinOffset(), getMaxOffset() to monitor.
   3. Use composition to shield lower levels: For example, the getFileToWrite and getFileSegment methods in FileSegmentQueue expose the underlying implementation, which should be shielded with a middle-level composition implementation (in a sense, a bridge).

2. Thread model and lock granularity: The current commit and append thread model design is unreasonable, causing the lock granularity to be too large. The swap queue's minimum granularity is tied to timed tasks. During uploading, commitlog is persisted before consumequeue, and the thread waiting design here is unreasonable and needs to reduce the granularity of the swap lock.

3. Object construction and metadata management:

   1. Reflection exists in the constructor: In the original design, FileSegment and SegmentQueue are initialized every time with a dependency on storeConfig, which is suitable for using the factory pattern to reuse configuration rather than passing references. The better implementation is to make it SPI-based.
   2. Metadata management improvement: The metadata management structure is messy, and the model hierarchy is not clear, with domain knowledge scattered. For example, TieredMetadataStore should not care about FileSegment and only needs to focus on FileSegmentMetadata. Just like TopicConfigManager only cares about TopicConfig and does not need to care about MappedFile. For example, the following two implementations should be in SegmentQueue instead of MetadataStore: void updateFileSegment(TieredFileSegment fileSegment); FileSegmentMetadata getOrCreateFileSegment(TieredFileSegment fileSegment).


4. Other optimization parts:
   1. getFileToWrite() actually includes many write behaviors, so remove the side effect of read-only methods.
   2. Fix the offset not declared as volatile, config manager serialization not working.
   3. Fix bugs in destroy duplicate deletion and optimize protective logic.
   4. Add unit tests and add explanations for return value error codes.

Of course, the improved design is not perfect.


## Interface Design/Change

for metadata manager:

```java
public interface TieredMetadataStore {

    void setMaxTopicId(int maxTopicId);
	
    TopicMetadata getTopic(String topic);

    void iterateTopic(Consumer<TopicMetadata> callback);

    TopicMetadata addTopic(String topic, long reserveTime);

    void updateTopicReserveTime(String topic, long reserveTime);

    void updateTopicStatus(String topic, int status);

    void deleteTopic(String topic);

    QueueMetadata getQueue(MessageQueue queue);

    void iterateQueue(String topic, Consumer<QueueMetadata> callback);

    QueueMetadata addQueue(MessageQueue queue, long baseOffset);

    void updateQueue(QueueMetadata metadata);

    void deleteQueue(MessageQueue queue);

    FileSegmentMetadata getFileSegment(TieredFileSegment fileSegment);

    void iterateFileSegment(Consumer<FileSegmentMetadata> callback);

    void iterateFileSegment(TieredFileSegment.FileSegmentType type, String topic, int queueId,
        Consumer<FileSegmentMetadata> callback);

    FileSegmentMetadata updateFileSegment(TieredFileSegment fileSegment);

    void deleteFileSegment(MessageQueue mq);

    void deleteFileSegment(TieredFileSegment fileSegment);

    void destroy();
}
```

for CompositeAccess:

```java
public interface CompositeAccess {

    /**
     * Initializes the offset for the flat file.
     * Only affect the dipsatch offset if the file has already been initialized.
     *
     * @param offset init offset for consume queue
     */
    void initOffset(long offset);

    /**
     * Appends a message to the commit log file, but does not commit it immediately
     *
     * @param message the message to append
     * @return append result
     */
    AppendResult appendCommitLog(ByteBuffer message);

    /**
     * Appends a message to the commit log file
     *
     * @param message the message to append
     * @return append result
     */
    AppendResult appendCommitLog(ByteBuffer message, boolean commit);

    /**
     * Append message to consume queue file, but does not commit it immediately
     *
     * @param request the dispatch request
     * @return append result
     */
    AppendResult appendConsumeQueue(DispatchRequest request);

    /**
     * Append message to consume queue file
     *
     * @param request the dispatch request
     * @param commit  whether to commit
     * @return append result
     */
    AppendResult appendConsumeQueue(DispatchRequest request, boolean commit);

    /**
     * Persist commit log file
     */
    void commitCommitLog();

    /**
     * Persist the consume queue file
     */
    void commitConsumeQueue();

    /**
     * Persist commit log file and consume queue file
     */
    void commit(boolean sync);

    /**
     * Asynchronously retrieves the message at the specified consume queue offset
     *
     * @param consumeQueueOffset consume queue offset.
     * @return the message inner object serialized content
     */
    CompletableFuture<ByteBuffer> getMessageAsync(long consumeQueueOffset);

    /**
     * Get message from commitlog file at specified offset and length
     *
     * @param offset the offset
     * @param length the length
     * @return the message inner object serialized content
     */
    CompletableFuture<ByteBuffer> getCommitLogAsync(long offset, int length);

    /**
     * Asynchronously retrieves the consume queue message at the specified queue offset
     *
     * @param consumeQueueOffset consume queue offset.
     * @return the consumer queue unit serialized content
     */
    CompletableFuture<ByteBuffer> getConsumeQueueAsync(long consumeQueueOffset);

    /**
     * Asynchronously reads the message body from the consume queue file at the specified offset and count
     *
     * @param consumeQueueOffset the message offset
     * @param count              the number of messages to read
     * @return the consumer queue unit serialized content
     */
    CompletableFuture<ByteBuffer> getConsumeQueueAsync(long consumeQueueOffset, int count);

    /**
     * Return the consensus queue site corresponding to the confirmed site in the commitlog
     *
     * @return the maximum offset
     */
    long getCommitLogDispatchCommitOffset();

    /**
     * Gets the offset in the consume queue by timestamp and boundary type
     *
     * @param timestamp    search time
     * @param boundaryType lower or upper to decide boundary
     * @return Returns the offset of the message
     */
    long getOffsetInConsumeQueueByTime(long timestamp, BoundaryType boundaryType);

    /**
     * Mark some commit log and consume file sealed and expired
     *
     * @param expireTimestamp expire timestamp, usually several days before the current time
     */
    void cleanExpiredFile(long expireTimestamp);

    /**
     * Destroys expired files
     */
    void destroyExpiredFile();

    /**
     * Shutdown process
     */
    void shutdown();

    /**
     * Delete file
     */
    void destroy();
}
```

## Implementation

The data model provided by RocketMQ tiered storage is similar to the local model, but it changes the concept of CommitLog and ConsumeQueue as follows:

- TieredFileSegment: Similar to MappedFile, it describes a handle for a file in a tiered storage system. Additionally, the TieredFileSegment is an AppendOnly fixed-length byte stream that supports byte-granularity append writing and random reading. Each TieredFileSegment has its own metadata, such as type, write position, creation and update time
- TieredFlatFile: Similar to MappedFileQueue. The TieredFlatFile can be viewed as a linked list of zero or more fixed-length TieredFileSegment, providing unbounded semantics for the stream. Only the last file in the queue can be in an unseal state (writable), while all previous files must be in a sealed state (read-only). Once the Seal operation is completed, the TieredFileSegment becomes immutable.
- TieredCommitLog: Unlike the local CommitLog, it is written by mixing multiple CommitLogs split at the granularity of a single Topic and a single queue. 
- TieredConsumeQueue: It is an index that points to the offset of TieredCommitLog, and is strictly and continuously increasing. The actual position of the index changes from pointing to the position of the CommitLog to the offset of TieredCommitLog. 
- CompositeFlatFile: It combines TieredCommitLog and TieredConsumeQueue objects and provides encapsulation of the concept.

![image](https://github.com/apache/rocketmq/assets/22487634/2e2a61aa-3aa7-433a-a8fa-e224724b7fdc)

RocketMQ's storage implements a Pipeline, similar to an interceptor chain or Netty's handler chain, where read and write requests go through multiple processors in the Pipeline. The concept of the Dispatcher is to build an index for the written data. When the tiered storage module is initialized, a TieredDispatcher is created and registered as a processor in the CommitLog's dispatcher chain. Whenever a message is sent to the Broker, TieredDispatcher is called for message distribution. Let's trace the process of a single message entering the storage layer: 

![image](https://github.com/apache/rocketmq/assets/22487634/4c0d184a-82bf-4a83-a631-04381cb3486a)

1. Messages are sequentially appended to the local commitlog and the local max offset is updated (yellow section in the figure). To prevent "read oscillation" （读摆动） caused by multiple replicas during a crash, the minimum position of the majority of replicas is confirmed as the "low watermark," which is referred to as the commit offset (2500 in the figure). In other words, the data between the commit offset and the max offset is waiting for multi-replica synchronization.
2. After the commit offset >= message offset, the message is uploaded to the cache of the secondary storage commitlog (green section in the figure) and the max offset of this queue is updated.
3. The index of the message is appended to the consume queue of this queue, and the max offset of the consume queue is updated.
4. Once the cache size in the commitlog file buffer exceeds the threshold or waits for a certain period, the cache of the message would be uploaded to the commitlog in distribution storage system, then the index information is submitted. There is an implicit data dependency here that causes the index to be updated later than the original data. This mechanism ensures that all data in the cq index can be found in the commitlog. In a crash scenario, the commitlog in the tiered storage may be redundantly built (重复构建), and there will be no cq pointing to this data. Since the file itself is still managed by the queue model, the entire data segment can be recycled when it reaches its TTL, and there is no "data leakage" in the data stream.
5. When the index upload is also completed, update the commit offset in the tiered storage (green section is submitted).
6. When the system restarts or crashes, multiple dispatchers' minimum positions will be selected to redistribute to the max offset to ensure that no data is lost.

## Compatibility, Deprecation, and Migration Plan

- re backward and forward compatibility taken into consideration? No
- Are there deprecated APIs? Yes. Actually, the old API was not used and should be removed.
- How do we do migration? No need

## Implementation Outline

I will submit the modifications to the API and model in the first phase as soon as possible, and there will be some improvements in the second phase that do not affect the API.

# Rejected Alternatives

## How do alternatives solve the issue you proposed?

## Pros and Cons of alternatives

## Why should we reject above alternatives