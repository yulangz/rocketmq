## Status

- Current State: Discussing
- Authors: [nowinkeyy](https://github.com/nowinkeyy)
- Shepherds: [lizhanhui](https://github.com/lizhanhui)
- Mailing List discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)
- Pull Request:
- Released:
- Related Docs: [English](https://yu7y22ce7k.feishu.cn/docx/doxcn9A5lPA05AD0oISoB11FL2f) [中文版](https://yu7y22ce7k.feishu.cn/docx/doxcnltrB7VzUKCqx0yxqMgJ4RM)

 

## Background & Motivation

### What do we need to do

 

- Will we add a new module? No.

- Will we add new APIs? Yes.

- Will we add a new feature? Yes.

 

### **Why should we do that**

- Are there any problems of our current project?

    Messages published by the producers are eventually appended to CommitLog files on broker nodes, out of which the ReputMessageService builds consume queues. The building procedure turns to be both IO and CPU intensive: when message production traffic is heavy, building of the ConsumeQueue hits bottleneck, causing significant commit log dispatch lag warning and further delay in message consumption among consumers. For application that is sensitive to end-to-end latency, a spike is observed in APM(application performance monitor).

 

- What can we benefit from proposed changes?

    Prior to the ReputMessageService thread, add a thread pool to process messages concurrently. 

    The ReputMessageService first takes CommitLog message in turn and assigns a unique and sorted logical number to it, then wraps it as a task and dispatch it into the thread pool. Workers in the thread pool execute these tasks concurrently to 1) build ConsumeQueue entries, 2) group commit entries for each message queue.

    Even if building consume queues for each message queue are parallelized, we can ensure that entries in ConsumeQueue for the messages are still in-order.

## Goals

- What problem is this proposal designed to solve?

    To solve the problem that ReputMessageService is the single thread and generation of ConsumeQueue messages is too slowly. Refer to above for details.

- To what degree should we solve the problem?

## **Non-Goals**

- What problem is this proposal NOT designed to solve?

- Are there any limits of this proposal?

 

## Changes

### Architecture

- **The old way**

    1.The producer sends message to the broker
    
    2.The broker writes the message to CommitLog
    
    3.The ReputMessageService thread scans CommitLog messages sequentially and wraps them as DispatchRequest

    4.The ReputMessageService thread executes DispatchRequest sequentially, generates ConsumeQueue messages and write them to ConsumeQueue

    ![img](https://yu7y22ce7k.feishu.cn/space/api/box/stream/download/asynccode/?code=N2Y4ZWM5OTFkMDA3YzljMjQ4MmVhNTE5ZTRkM2U3OGRfSGpham52aks0TnlNc1R2STdDdWx1bDNlYVBaZHpvU2hfVG9rZW46Ym94Y25vM1pMM1cxVEVvQ1B6YmRSOFNOZDFQXzE2NjU5ODQwMDQ6MTY2NTk4NzYwNF9WNA)

- **The new way**

    1.The producer sends message to the broker

    2.The broker writes the message to CommitLog

    3.The ReputMessageService thread scans CommitLog messages sequentially and wraps them as DispatchRequest

    4.The ReputMessageService thread assigns a number (unique and sorted in the same ConsumeQueue) to each DispatchRequest and wraps the DispatchRequest as a task into the thread pool

    5.Thread in the thread pool execute task to generate ConsumeQueue message and write it to the ConsumeQueue. Before committing to the ConsumeQueue, the thread needs to query bitmap to ensure that the message is committed in order (meaning that if there are messages before this message that are not written to ConsumeQueue, they will not be committed and invisible, if all messages before this message have been written, the message is committed and visible)

    ![img](https://yu7y22ce7k.feishu.cn/space/api/box/stream/download/asynccode/?code=MTM5Yzc4MDUwODVjYTA1NDJjMmIzZDBiZDFiYTYzODNfNTlRTFZ4SnczeFdIcjRRdFVYWmdtTU9jSGdPallPZm5fVG9rZW46Ym94Y25JNlk4c05abXNzbGNFOUNVamtXYU9lXzE2NjU5ODQwMDQ6MTY2NTk4NzYwNF9WNA)

 

- **Take an example from the picture above**

    The ReputMessageService thread scans the CommitLog, gets five messages (the same topic and queueId), goes to bitmap in turn to get a unique and sorted number, and puts it into the thread pool for parallel execution.

    m1, m2 and m5 have been written into ConsumeQueue, and the commit-position is m2. Although m5 has written to the ConsumeQueue, the commit-position cannot be updated, and the commit-position only can be updated when the next position of commit-position (m3) is written.

    Suppose that when m3 and m4 have been written to ConsumeQueue, the commit-position will not be updated to m3 directly, because there may be a larger committable position, so it is necessary to query bitmap to get the largest committable position.

    At the point, m4 and m5 have been written to ConsumeQueue, and m6 has not been written to ConsumeQueue, so the largest committable position is m5, and the final commit-position is updated to m5.

    **NOTE：Access to the same ConsumeQueue is serialized and may be changed to limited parallelization later.**

### **Interface Design/Change**

### **Compatibility, Deprecation, and Migration Plan**

### **Implementation Outline**

### Phase 1

### Phase 2

### Phase 3