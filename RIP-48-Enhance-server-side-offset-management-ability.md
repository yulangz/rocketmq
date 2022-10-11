# Status

- Current State: Discussing
- Authors: [lizhimins](https://github.com/lizhimins)
- Shepherds: [lizhanhui](https://github.com/lizhanhui), [zhouxinyu](https://github.com/zhouxinyu), [duhenglucky](https://github.com/duhenglucky)
- Mailing List Discussion: dev@rocketmq.apache.org
- Pull Request: #PR_NUMBER
- Released: <released_version>

# Background & Motivation

## What do we need to do

- Will we add a new module? No.

- Will we add new APIs? No.

- Will we add a new feature? Yes.

## Why should we do that

- Are there any problems with our current project?

There are some problems in the current implementation of rocketmq offset management.

1. Restarting the client in broadcast mode will consume message from max offset by default
2. The accumulation amount is virtual-high when we uses "ConsumeProgress" command to observe the consumption progress.
3. Reset offset by rpc is not reliable.

- What can we benefit from proposed changes?

The implementation of enhance server-side offset management ability improved the above problems.

# Goals

- What problem is this proposal designed to solve?

RIP X Enhance server-side offset management ability

# Non-Goals

- What problem is this proposal NOT designed to solve?

# Changes

## Architecture

1. Support server-side offset management in broadcast consumption mode

RocketMQ's broadcast mode means that messages are subscribed to all client which belong to a single consumer group, and is generally used to clear the cache or build the cache on the client for business logic. In the original design, if the local offset does not exist, the client cannot consume messages within the restart interval because the client starts to consume from the max(latest) offset after application restarting. We introduce a feature to avoid this situation by using the minimum consumption offset of all clients as the offset of this consumer group in broadcast mode.

2. Introduces the concept of ready and inflight messages

In the old version of RocketMQ, if the user's consumption logic takes a long time, there will be a large number of messages being processed. The broker only saves the offset that have been successfully processed (clients returning success or failure), the accumulation amount is virtual-high when the user uses "ConsumeProgress" command to observe the consumption progress. Therefore, we define two new concepts, called ready and Inflight messages, to observe the consumption progress more accurately.

The ready state means that the message is ready on the server side, that is the ConsumeQueue is successfully constructed and can be pulled and consumed by the client. The inflight state refers to the state in which the message is acquired by the consumer client, and the consumption result has not been returned in the process of consumption. So how do we estimate the amount of messages being processed? In fact, it is the difference between the position of the latest news obtained by the pull request and the submitted position.

![image-20220928172259995](https://user-images.githubusercontent.com/22487634/194984214-788614f9-afde-4646-90dd-5ce3fea35702.png)

3. Support reset offset in server side to improve the success rate.

There are multiple links related to update offset in RocketMQ

1. Pull Message（non-descending sequence）, org.apache.rocketmq.broker.processor.PullMessageProcessor#processRequest
2. Consumer persist offset to remote（non-descending sequence） org.apache.rocketmq.broker.processor.ConsumerManageProcessor#updateConsumerOffset
3. Use admin API to reset offset, org.apache.rocketmq.client.impl.ClientRemotingProcessor#resetOffset
4. HA offset synchronize, org.apache.rocketmq.broker.slave.SlaveSynchronize#syncConsumerOffset

Due to complex concurrency issues in a distribution systen, these links may overlap each other. For example, link 1 2 and 3 may conflict, causing the reset offset API fail. We add a map on the server-side to temporarily store the request to reset the offset, and when the client pulls the message next time, the requested offset is overwritten. And This operation does not require the client to implement the reset offset interface, which reduces the complexity of the client.

![image-20221008152546251](https://user-images.githubusercontent.com/22487634/194984240-30bce8c9-2d3e-4d48-a23c-adf89711e88f.png)

## Interface Design/Change

The specific pull request is still in progress, but it will be seen by everyone soon.

## Compatibility, Deprecation, and Migration Plan

- re backward and forward compatibility taken into consideration? No
- Are there deprecated APIs? No
- How do we do migration? No

## Implementation Outline

We will implement the proposed changes in three pull requests. 



# Rejected Alternatives



## How do alternatives solve the issue you proposed?



## Pros and Cons of alternatives



## Why should we reject above alternatives