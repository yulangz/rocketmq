# Status
- Current State: Proposed
- Authors: [focus-rth](https://github.com/Focus-rth)
- Shepherds: [fuyou001](https://github.com/fuyou001), [zhouxinyu](https://github.com/zhouxinyu), [lizhanhui](https://github.com/lizhanhui),不铭
- Mailing List Discussion: dev@rocketmq.apache.org
- Pull Request: #PR_NUMBER
- Released: <released_version>
# Background & Motivation
## What do we need to do
- Will we add a new module? -- No.
- Will we add new APIs? -- No.
- Will we add a new feature? -- Yes.
##  Why should we do that
- Are there any problems with our current project?
Apache RocketMQ provides a distributed transaction feature that is similar to X/Open XA, ensuring transaction consistency. We propose some solutions to optimize.
1. one OP message corresponds to one half message, which will cause write amplification
2. the sender commit or rollback message in a short time before half message placed, so the commit/rollback message will fail beacause of half message may not be read in asynchronous flush mode.

- What can we benefit from proposed changes?
The feature of transaction message improved the above problems.
# Goals
- What problem is this proposal designed to solve?
see the previous section, problems stated will be solved.
# Non-Goals
- What problem is this proposal NOT designed to solve?
N/A
# Changes
## Architecture
1. support batch OP message
We realize that one OP message can correspond to multiple half messages, that is, write the queueOffset of multiple half messages in the body of the OP message.

![Transaction Architecture](https://user-images.githubusercontent.com/685537/195086525-969eb818-a1ae-40d5-a3ec-eabe3e572d21.png)


The implementation of batch OP messages is similar to the Nagle algorithm of tcp transmission, using two parameters of timing and specified size to control the writing of OP messages. First, the content corresponding to the OP message is aggregated in memory. If the body of the OP message exceeds the specified size (transactionOpMsgMaxSize defaults to 4096), the OP message is written. Otherwise, the timed (transactionOpBatchInterval defaults to 3 seconds) flushes the written OP message.
Specific to the online optimization effect, the writing TPS of Half messages and OP messages can be up to 100:1. That is, one OP message corresponds to 100 Half messages. Of course, if the TPS of the Half message is higher, this ratio will be larger.
For specific implementation, please refer to TransactionalMessageServiceImpl#batchSendOpMessage (timed write) and TransactionalMessageServiceImpl#deletePrepareMessage (specified size write)



2. add half message memory cache
Currently, Broker's default flushing mode is asynchronous flush, and data cannot be read from memory. If the sender commit/rollback in a short time, but the previously written Half message has not been placed on the disk and cannot be read, the commit/rollback will fail. Therefore, the half message is cached.
The cache size is 8000，allow the map to grow up to 8000 entries and then delete the eldest entry each time a new entry is added, maintaining a steady state of 8000 entries.The addition logic of the cache is to write to the disk asynchronously or synchronously, and then add it to the cache; the read logic of the cache is to read the cache first, and then read the disk if it does not hit; if it hits, read the cache and delete it; if the broker restarts, all the cache is lost.The specific implementation is relatively simple, you can refer to TransactionCacheManage.

## Interface Design/Change
- Method signature changes? -- Nothing specific.
- Method behavior changes? -- Nothing specific.
- CLI command changes? -- Nothing specific.
- Log format or content changes? -- Nothing specific.

## Compatibility, Deprecation, and Migration Plan
- re backward and forward compatibility taken into consideration? -- No.
- Are there deprecated APIs? -- No.
- How do we do migration? -- No.

## Implementation Outline
We will implement the proposed changes in two pull requests.

# Rejected Alternatives

## How do alternatives solve the issue you proposed?

## Pros and Cons of alternatives

## Why should we reject above alternatives
