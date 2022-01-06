# Status
- Current State: Proposed
- Authors: [guyinyou](https://github.com/guyinyou)
- Shepherds: RongtongJin
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: 
- Released: 
- Related Docs: [Auto batching in producer](https://docs.google.com/document/d/1ydZrA-FNrjM52I4H72EM2yaNAEYpRHV5uUSmFgxhWUQ/edit#heading=h.l32rlf9jfklr)

# Background & Motivation
## What do we need to do

- Will we add a new module?
No.
- Will we add new APIs?
Yes.
- Will we add new feature?
Yes.

## Why should we do that

- Are there any problems of our current project?
Messages are sent in batches with very good throughput performance, but currently the business end needs to be packaged manually by itself, and the experience is not good enough.

---------------------中文分割线----------------------

消息批量发送有着非常好的吞吐量表现，但是目前需要业务方自行手动打包，体验不够好。

- What can we benefit proposed changes?

To realize the automatic message packaging function, the business side only needs to simply call the sending interface of a single message to realize the throughput of batch sending.

---------------------中文分割线----------------------

实现一个消息自动打包的功能，业务方只需要简单调用单条消息的发送接口，即可达到批量发送时的吞吐量。


# Goals
- What problem is this proposal designed to solve?  

Realize the auto batching function in the producer.

实现producer的消息自动打包功能。


# Changes

## Architecture

ProduceAccumulator

- Instructions for open

[![Tx31G4.png](https://s4.ax1x.com/2022/01/06/Tx31G4.png)](https://imgtu.com/i/Tx31G4)

batchMaxBytes：每个MessageAccumulation最大累积大小，超过该阈值将会打包发送

batchMaxDelayMs：每条Message在暂存区最长等待时间，超过该阈值将会打包发送

totalBatchMaxBytes：所有MessageAccumulation累积大小之和的阈值，超过该阈值后续请求将直接发送

[![Tx3adK.png](https://s4.ax1x.com/2022/01/06/Tx3adK.png)](https://imgtu.com/i/Tx3adK)

Producers with the same MQClientId shares the same ProducerAccumulator instance

---------------------中文分割线----------------------

相同 MQClientId 的 Producers 共用同一个 ProducerAccumulator

[![Tx3BJe.png](https://s4.ax1x.com/2022/01/06/Tx3BJe.png)](https://imgtu.com/i/Tx3BJe)

There are two MAP corresponding to two sending methods (syncSendBatchs and asyncSendBatchs). The key of Map is the smallest granularity of batch, the value is the staging area for messages.

There are two SERVICE for cooperating with two sending modes (GuardForSyncSendService and GuardForAsyncSendService), notify or send or cleanup regularly according to 'batchMaxDelayMs'.

---------------------中文分割线----------------------

每一个 ProducerAccumulator 中有两个 MAP 对应两种不同的发送模式（syncSendBatchs、asyncSendBatchs），MAP 的键为批量发送聚合时的最小粒度（例如MessageQueue），值为待聚合消息的缓冲区。

同时还有两个 SERVICE 用来协调两种不同的发送模式（GuardForSyncSendService、GuardForAsyncSendService），以 batchMaxDelayMs 为周期，唤醒线程、发送消息或者清理对象。

Map<String, MessageAccumulation> 中的Key，在未指定 MessageQueue 的期望它是 Topic（避免出现大量小batch的情况），在指定了发送队列的情况下期望它是 Topic+Queue。这里也许应该抽象出一个类似于“聚合粒度”的概念

这里还需要对 MessageAccumulation 个数做些限制，额外内存开销大约为 batchMaxBytes * MessageAccumulation个数

[![Tx3yQA.png](https://s4.ax1x.com/2022/01/06/Tx3yQA.png)](https://imgtu.com/i/Tx3yQA)

Assign topic, waitStoreMsgOK, and createTime every time an instance of MessageAccumulation is created. When the user calls the send interface, the getOrCreateProduceAccumulator interface is called according to the message. If a new MessageAccumulation needs to be created, mqProducer will be registered.

---------------------中文分割线----------------------

每次创建 MessageAccumulation 实例时会赋值 topic、waitStoreMsgOK 和 createTime 信息。当用户调用 send 接口时，会触发 getOrCreateProduceAccumulator 接口，如果此时需要创建 MessageAccumulation 实例，则将本次发送操作上下文中的 mqProducer 注册进去。（相当于随机挑选该 instance 下的 producer，最大化利用已申请的资源，例如线程池）

[![Tx3cLt.png](https://s4.ax1x.com/2022/01/06/Tx3cLt.png)](https://imgtu.com/i/Tx3cLt)

When the user sends a message in sync mode, the thread calling the send will check if it is ready to send or be suspended. GuardForSyncSendService will regularly notify when it is ready to send during the batchMaxDelayMs cycle and clean up long unused MessageAccumulation.

---------------------中文分割线----------------------

当用户以同步模式发送消息时，调用发送接口的线程将会检测是否满足发送条件，不满足将会挂起。GuardForSyncSendService 会以 batchMaxDelayMs 为周期唤醒满足发送条件①的线程，并清理久未使用的GuardForSyncSendService。

[![Tx3Rdf.png](https://s4.ax1x.com/2022/01/06/Tx3Rdf.png)](https://imgtu.com/i/Tx3Rdf)

When the user sends a message in async mode, it will add message and sendCallBack to the two lists of MessageAccumulation respectively, and then return directly. GuardForAsyncSendService will regularly send by mqProducer if it is ready to send during the batchMaxDelayMs cycle and clean up long unused MessageAccumulation.

---------------------中文分割线----------------------

当用户以异步模式发送消息时，待发送的消息和回调函数会添加到 MessageAccumulation 的两个 list 上，然后直接返回。GuardForAsyncSendService 会以 batchMaxDelayMs 为周期在满足发送条件①的情况下用注册进去的 mqProducer 异步发送，并清除久未使用的 MessageAccumulation。

①满足发送条件：距离 MessageAccumulation 创建时间 ≥ batchMaxDelayMs，或者 MessageAccumulation 中累积消息大小 ≥ batchMaxBytes

关于消息聚合的注意事项：

在旧版本 Broker 中，服务端会将 MessageBatch 解包后构建索引，此时一个 MessageBatch 里的多条数据可以有各自不同的 TAG、KEYS、MsgID。

但是在使用 BatchConsumerQueue 的新版本 Broker 中，一个 MessageBatch 将被当做一条独立的消息，有且只有唯一的 TAG、KEYS、MsgID。因此需要关注构建 BCQ 和 index 时的细节。

TAG：具有相同 TAG 的消息才能聚合（未设置 TAG 的也需要分开聚合）（新旧版本兼容）

KEYS：不以 KEYS 作为聚合维度限制，但是在打包成 MessageBatch 后，需要设置 MessageBatch 的 KEYS 为每条消息 KEYS 的并集。（新旧版本兼容）

MsgID：构成一个 MessageBatch 的所有消息具有相同的 MsgID。（旧版本不兼容）因此需要对返回的MsgID做特殊判断和处理


## Interface Design/Change
- Method signature changes

  Nothing specific.

- Method behavior changes

  When calling the send interface, distinguish whether it needs to be sent in batches. What is needed will go to MessageAccumulation, and the others will be sent as usual

---------------------中文分割线----------------------

调用 send 接口时区分是否需要批量发送，需要的将会进入 MessageAccumulation ，其他的照常发送

- CLI command changes

  Nothing specific.

- Log format or content changes
  Nothing specific.

## Compatibility, Deprecation, and Migration Plan
 
- Are backward and forward compatibility taken into consideration?

Reuse the ability of MessageBatch, only as an enhancement.

---------------------中文分割线----------------------

复用了 MessageBatch 的能力，只作为能力增强

- Are there deprecated APIs?
Nothing specific.

- How do we do migration?
Upgrade on the client side.

## Implementation Outline
We will implement the proposed changes by 3 phases.

### Phase 1

Implement autobatch on the client-side

在客户端侧实现 autobatch

此阶段的实现对用户完全屏蔽，无感知。由于 MessageBatch 发送时，返回的 sendResult 中会将每一条消息的 msgId 用逗号拼接起来，分割后即可得到每一条消息各自的 msgId。

### Phase 2
Enhance the ability of MessageBatch to support multiple partitions
增强 MessageBatch 的能力，使其支持不同分区的消息一起打包

此阶段的实现比较适用于普通业务消息的发送，能大大提高吞吐量，此时的 msgId 和 Phase1 的情况一致。

### Phase 3
Increase BatchConsumerQueue so that messages sent in the same batch share one msgId
增加 BatchConsumerQueue ，使同一批发送的消息共用一个 msgId

此阶段的实现比较适用于 streaming 场景，针对于特定分区的消息发送，此时同一批发送消息拥有同一个 msgId

# Rejected Alternatives 
## How does alternatives solve the issue you proposed?

Manual batch sending by users.

用户手动批量发送消息

## Pros and Cons of alternatives
Pros: higher throughput.

Cons: It is not convenient to use, users need to understand the details.

优点：高吞吐

缺点：使用不够友好，用户需要了解具体细节（比如不可打包重试消息、定时消息且必须同topic和waitStoreMsgOk）
## Why should we reject above alternatives
Users are still allowed to manually send messages in batches, and when the send interface is called, the incoming MessageBatch will be sent directly.

用户现在依然可以手动发送批量消息，当调用 send 接口入参为 MessageBatch 时将直接发送，不受本次功能影响




