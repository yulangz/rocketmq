# Status
- Current State: Draft
- Authors:  GenerousMan
- Shepherds: fuyou001, hill007299
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: 
- Released: <relased_version>


# Background & Motivation
## What do we need to do

Optimize the client's selection strategy for brokers when sending messages, and use multiple selection strategies as a pipeline to filter suitable queues.

Design the circuit breaker when the client sends messages --- actively and dynamicly detect reachable brokers, which could isolate unavailable brokers. 

优化client以及proxy在cluster模式下，在发送消息时对broker/queue的选择策略，并将多个选择策略作为一个pipeline，能够对合适的queue进行过滤。
此外，为了丰富broker的选择策略，本RIP增加客户端发送消息时的断路器：主动探测可到达的broker，将不可用broker进行隔离。从客户端角度增加消息发送的流畅性。


## Why should we do that

When sending a message, the current client needs to select a broker and a queue as the destination for the message. In most cases, such selection will be normal and effective - meaning that messages typically reach the queue of the target broker. However, when some brokers' network conditions are not good, the sending process may end in failure. Due to the uncertain fluctuation time of the network, it is necessary to respond promptly when messages can not be sended correctly, such as replacing the broker to send the message. If we continue to send messages to this type of broker, it will always cause the sending of messages to fail, thereby wasting a lot of time.

当前的client在发送消息时需要选择broker以及queue作为消息的目的地。大部分情况下，这种选择都是正常且有效的——这意味着消息通常能够到达目标broker的queue中。但是，在某些broker的网络情况不好时，消息的发送可能以失败告终。由于网络的波动时间是不可确定的，当消息发送失败后需要及时作出反应，例如更换broker进行消息的发送。如果持续向这类broker发送消息，将一直导致消息的发送失败，从而浪费大量的时间。

The current client module has built-in latency related functions, but this function is passive - isolating brokers based on the sending results, and is not aware of their availability. When multiple brokers encounter problems, they may be randomly selected for transmission. Therefore, this RIP hopes to provide a client's ability to actively detect broker reachability. Subdividing the available states of brokers and turning on this function when the network fluctuates greatly will make the process of selecting brokers more intelligent.

当前的client模块已经内置了latency相关功能，但是该功能是被动的——根据发送结果隔离broker，而且对于broker的可用情况是无法感知的。在多台broker出现问题时，可能会将所有broker均置为not available的状态，从而随机选择broker进行发送。同时，proxy在cluster模式下，向broker发送消息时仍然保持较为简单的轮流发送的方式，无容错机制。因此，本RIP希望提供一种client的能力：能够根据设置的规则对queue进行过滤，通过这些过滤器，能够帮助找到更加可靠的message queue。在该RIP中，还将实现一种可达broker的主动探测器，能够主动探测broker可达情况。将broker的可用状态进行细分，在网络波动较大时打开该功能，将在发送过程中选择broker的过程更加智能。


# Goals
- What problem is this proposal designed to solve?  

When clients fail to send messages, the selection strategy for mq is simple: Penalize brokers who have failed to send through unavailable time. When brokers frequently fail, all brokers may enter the FaultItemTable. Finally, the broker is randomly selected. The client may wait for a long time due to the failure to send the message.

当前客户端在消息发送失败时，对mq的选择策略单一：在broker频繁出现故障时，可能所有broker都进入FaultItemTable。若broker处于不稳定的状态，可能最后将导致随机选择broker，客户端可能出现大量发送失败。

本RIP旨在提出一套队列选择方式，能够通过pipeline的模式对队列进行多维度的过滤，以帮助客户端主动选择broker进行发送，能够优化在不稳定环境下的发送情况。此外，该能力还能复用至cluster模式下的proxy中。



- To what degree should we solve the problem? 

1. Add an active detection thread to the client, and periodically detect the survival status of the broker through long connections.
2. Optimize the message sending process of the client, and consider more dimensions when selecting a queue.
3. This capability will be reused in the proxy in cluster mode to optimize the proxy's ability to select queues.
4. Add a switch to set whether this function is valid.

1. 客户端增加主动探测线程，通过长连接定期探测broker存活情况。
2. 优化客户端的队列选择过程，设置队列选择器，选择queue时有更多维度的考虑。
3. 将该能力复用至cluster模式下的proxy中，优化proxy选择队列的能力。
4. 增加开关设置该功能是否生效。

# Non-Goals.

- What problem is this proposal NOT designed to solve? 
 
Nothing specific.

- To what degree should we solve the problem?

The queue selector needs to have a bottom-up design to avoid no queues available after the function is enabled.

Since it is a periodic detection, it is necessary to use a long connection during the detection to reuse the previous channel.

队列选择器需要有兜底设计，避免开启功能后无队列可用。

由于是定期探测，需要在探测时使用长连接，以复用此前的channel。

# Changes
## Architecture

First, the queue selection process is converted into a pipeline of several queue selector groups

Then selectors can be designed to make the selection process more flexible:

1. Enrich the available status of brokers:

  a. available: Whether it was blown due to a failure to send.

  b. Reachable: Whether the alive status can be detected by the detector.

2. Add a detector to periodically detect the reachability of the broker. Save the reachability of the broker locally as a list.

3. Modify the relevant logic of selectOneMessageQueue:

  a. Increase the status of the broker: whether it is reachable.

  b. First select the queue in available brokers.

  c. When there is no available broker, select the queue in the reachable brokers.

  d. When there is no available broker and no reachable broker, traverse the queue list one by one and try to send.

首先将队列选择过程转化为若干队列选择器组成的pipeline.

之后可以设计新的队列选择器，对broker的选择过程进行细分：

1. 丰富broker可用状态：

  a. available: 是否因为发送失败被熔断。

  b. reachable: 是否能被detector探测到存活状态。

2. 增加检测器，周期性探测broker的可到达情况。在本地保存broker的可达情况为一个列表。

3. 修改selectOneMessageQueue的相关逻辑：

  a. 增加broker的状态：是否可到达(reachable)。

  b. 首先选择available broker中的queue。

  c. 在无available broker时，选择reachable broker中的queue。

  d. 在无available broker且无reachable broker时，从queue list中逐个遍历尝试发送。


Here are the selection preferences for the two cases:

a. When there is an available broker, the available broker is preferred.

下面是两种情况的选择偏好情况：

a. 在有available broker时，优先选择available broker.

![image](https://user-images.githubusercontent.com/21963954/227834501-03a73f7c-6e08-47c7-92ff-f0e20840e854.png)

b. When a large number of brokers fail to send, they all enter the FaultItemTable, and may be in the state of not available. At this point, clients will select reachable brokers.

b.当大量broker出现发送失败时，均进入FaultItemTable，可能都处于 not available的状态。此时选择reachable brokers.

![image](https://user-images.githubusercontent.com/21963954/227834571-aa0816ed-63af-4b51-bd11-7003a9e32203.png)

# Plan

As Xinyu Zhou(@yukon) said, such detector can be one of queue selectors. Therefore, to make this RIP more scalable, this detector will be implemented as a queue selector. Other queue selectors can be implemented on this basis and function in the form of pipelines.
The first stage: complete the queue selector of the common client.

  a. Increase the filter pipeline design of the queue.

  b. Add the scheduled detection task of the detector as a reachable filter.

  c. Set the switch for this feature, and check remoteClientConfig regularly to control the detection switch of the client.

  d. In the normal message flow, improve the selectOneMessageQueue() method and add filter design.

The second stage: complete the queue selector of the client in the proxy.

  a. Reuse the filter pipeline and detector of the client.

  b. Switch this feature in the proxy.

  c. Improve the select() method to make queue selection more reliable.

正如 Xinyu Zhou(@yukon) 所说，这种检测器可以是队列选择器之一。因此，为了使这个 RIP 更具可扩展性，这个检测器将被实现为一个队列选择器。其他队列选择器可以在此基础上实现，以流水线的形式发挥作用。

第一阶段：完成普通client的队列选择器。

1. 增加queue的filter pipeline设计。

2. 增加detector的定时探测任务，作为reachable的filter。

3. 对该特性设置开关，定时检查remoteClientConfig，以便操控client的检测开关。

4. 在正常的消息流程中，改进selectOneMessageQueue()方法，加入filter设计。

第二阶段：完成proxy中的client的队列选择器。

1. 复用client的filter pipeline，以及detector。

2. 对proxy中的该特性作开关。

3. 改进select()方法，让队列选择更可靠。

# Rejected Alternatives

No other alternatives
