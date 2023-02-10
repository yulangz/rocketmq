## Status

- Current State: Implementing
- Authors: [nowinkeyy](https://github.com/nowinkeyy)
- Shepherds: [lizhanhui](https://github.com/lizhanhui)
- Mailing List discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)
- Pull Request: #5887
- Released:
- Related Docs: [English](https://yu7y22ce7k.feishu.cn/docx/doxcn9A5lPA05AD0oISoB11FL2f) [中文版](https://yu7y22ce7k.feishu.cn/docx/doxcnltrB7VzUKCqx0yxqMgJ4RM)

 

## Background & Motivation

### What do we need to do

 

- Will we add a new module? No.
- Will we add new APIs? Yes.
- Will we add a new feature? Yes.

 

### **Why should we do that**

- Are there any problems of our current project?

 

Messages published by the producers are eventually appended to CommitLog files on broker nodes.The ReputMessageService thread uses CommitLog to build ConsumeQueue.Building ConsumeQueue is single-threaded.

The process of building the ConsumeQueue turns out to be both IO and CPU intensive:  when message production traffic is heavy, building of the ConsumeQueue hits bottleneck, causing significant commit log dispatch lag warning and further delay in message consumption among consumers. For application that is sensitive to end-to-end latency, a spike is observed in APM(application performance monitor).

 

- What can we benefit from proposed changes?

Create multiple child threads to support the ReputMessageService main thread and maintain the order of the original message with a uniquely ordered logical number.So that we can generate ConsumeQueue messages quickly and ensure that the messages are orderly.

 

## Goals

- What problem is this proposal designed to solve?



The messages produced by the producer are written to CommitLog, and the ReputMessageService thread generates the ConsumeQueue messages with the CommitLog messages. The generation of ConsumeQueue messages depends on the ReputMessageService thread. When the message production traffic is heavy, the ConsumeQueue message is generated slowly, resulting in a delay in message consumption.

 

- To what degree should we solve the problem?



**There are two main bottlenecks in building ConsumeQueue:**



**Bottleneck 1**: Checking CommitLog to generate DispatchRequest takes too long;



**Solution**: Add a pool of threads to concurrently check CommitLog messages and wrap as DispatchRequest. The ReputMessageService thread first divides CommitLog messages into a batch of CommitLog messages (about 4MB) and assigns CommitLog messages to an orderly logical number, then wraps each batch of Commitlog messages as a task and distributes them to the thread pool. Performers in the thread pool execute these tasks concurrently: 1) check CommitLog messages and generate DispatchRequest, 2) place them into DispatchRequest collection in sequence.



**Bottleneck 2 (No need for the time being)**: Processing DispatchRequest to generate ConsumeQueue takes too long.



**Solution**: Add a pool of threads to process DispatchRequest concurrently. The executor in the thread pool executes these tasks concurrently: 1) Fetch DispatchReqeust from the DispatchReqeust collection, 2) process the DispatchRequest as a ConsumeQueue message and write to the ConsumeQueue.



## **Non-Goals**

- What problem is this proposal NOT designed to solve?
- Are there any limits of this proposal?

 

## Changes

### Architecture

- **The old way**

1.The producer sends messages to the broekr

2.The broker writes messages to CommitLog

3.The ReputMessageService thread scans CommitLog messages sequentially and checks them, converting CommitLog messages to ConsumeQueue messages and writing them to ConsumeQueue after successful checks

![](https://raw.githubusercontent.com/wiki/apache/rocketmq/assets/RIP_52_old_way.png)

- **The new way**

1.The producer sends messages to the broekr

2.The broker writes messages to CommitLog

3.The ConcurrentReputMessageService thread scans CommitLog messages sequentially and splits them into Batch CommitloLog messages of about 4MB, and then assigns an ordered number to each Batch CommitLog

4.Threads in the BatchDispatchRequestService thread pool check CommitLog messages concurrently and put them into the corresponding location of the collection according to the ordered number after the check is successful

5、The DispatchService thread takes out the checked CommitLog messages sequentially, converts the CommitLog messages into ConsumeQueue messages and writes them to ConsumeQueue

![img](https://raw.githubusercontent.com/wiki/apache/rocketmq/assets/RIP_52_new_way.png)



### **Interface Design/Change**



### **Compatibility, Deprecation, and Migration Plan**

### **Implementation Outline**

### Phase 1

### Phase 2

### Phase 3

### Phase 4