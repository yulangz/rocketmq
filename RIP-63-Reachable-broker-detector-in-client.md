# Status
- Current State: Draft
- Authors:  GenerousMan
- Shepherds: fuyou001, hill007299
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: 
- Released: <relased_version>


# Background & Motivation
## What do we need to do

Design the circuit breaker when the client sends messages --- actively and dynamicly detect reachable brokers, which could isolate unavailable brokers. 

增加客户端发送消息时的断路器：主动探测可到达的broker，将不可用broker进行隔离。从客户端角度增加消息发送的流畅性。


## Why should we do that


When sending a message, the current client needs to select a broker and a queue as the destination for the message. In most cases, such selection will be normal and effective - meaning that messages typically reach the queue of the target broker. However, when some brokers' network conditions are not good, the sending process may end in failure. Due to the uncertain fluctuation time of the network, it is necessary to respond promptly when messages can not be sended correctly, such as replacing the broker to send the message. If we continue to send messages to this type of broker, it will always cause the sending of messages to fail, thereby wasting a lot of time.

当前的client在发送消息时需要选择broker以及queue作为消息的目的地。大部分情况下，这种选择都是正常且有效的——这意味着消息通常能够到达目标broker的queue中。但是，在某些broker的网络情况不好时，消息的发送可能以失败告终。由于网络的波动时间是不可确定的，当消息发送失败后需要及时作出反应，例如更换broker进行消息的发送。如果持续向这类broker发送消息，将一直导致消息的发送失败，从而浪费大量的时间。

The current client module has built-in latency related functions, but this function is passive - isolating brokers based on the sending results, and is not aware of their availability. When multiple brokers encounter problems, they may be randomly selected for transmission. Therefore, this RIP hopes to provide a client's ability to actively detect broker reachability. Subdividing the available states of brokers and turning on this function when the network fluctuates greatly will make the process of selecting brokers more intelligent.

当前的client模块已经内置了latency相关功能，但是该功能是被动的——根据发送结果隔离broker，而且对于broker的可用情况是无法感知的。在多台broker出现问题时，可能会将所有broker均置为not available的状态，从而随机选择broker进行发送。因此，本RIP希望提供一种client的能力：主动探测broker可达情况。将broker的可用状态进行细分，在网络波动较大时打开该功能，将在发送过程中选择broker的过程更加智能。


# Goals
- What problem is this proposal designed to solve?  

When clients fail to send messages, the selection strategy for mq is simple: Penalize brokers who have failed to send through unavailable time. When brokers frequently fail, all brokers may enter the FaultItemTable. Finally, the broker is randomly selected. The client may wait for a long time due to the failure to send the message.

客户端在消息发送失败时，对mq的选择策略单一：在broker频繁出现故障时，可能所有broker都进入FaultItemTable。最后随机选择broker。客户端可能因为消息发送失败导致长时间的等待。

- To what degree should we solve the problem? 

1. Add an active detection thread to the client, and periodically detect the survival status of the broker through long connections.
2. Optimize the message sending process of the client, and consider more dimensions when selecting a queue.
3. Add a switch to set whether this function is valid.

1. 客户端增加主动探测线程，通过长连接定期探测broker存活情况。
2. 优化客户端的消息发送过程，选择queue时有更多维度的考虑。
3. 增加开关设置该功能是否生效。

# Non-Goals.

- What problem is this proposal NOT designed to solve? 
 
Nothing specific.

- To what degree should we solve the problem?

Since it is a periodic detection, it is necessary to use a long connection during the detection to reuse the previous channel.
由于是定期探测，需要在探测时使用长连接，以复用此前的channel。

# Changes
## Architecture

1. Enrich the available status of brokers:
  a. available: Whether it was blown due to a failure to send.
  b. Reachable: Whether the alive status can be detected by the detector.
2. Add a detector to periodically detect the reachability of the broker. Save the reachability of the broker locally as a list.
3. Modify the relevant logic of selectOneMessageQueue:
  a. Increase the status of the broker: whether it is reachable.
  b. First select the queue in available brokers.
  c. When there is no available broker, select the queue in the reachable brokers.
  d. When there is no available broker and no reachable broker, traverse the queue list one by one and try to send.

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

# Rejected Alternatives

 
No other alternatives
