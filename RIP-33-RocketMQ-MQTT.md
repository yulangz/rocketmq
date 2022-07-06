# Status
- Current State: Accept
- Authors: [pingww](https://github.com/pingww)
- Shepherds: duhengforever@apache.org
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: https://github.com/apache/rocketmq-mqtt
- Released: <relased_version>
- Related Docs: https://docs.google.com/document/d/1AD1GkV9mqE_YFA97uVem4SmB8ZJSXiJZvzt7-K6Jons/edit#heading=h.i42fwfxz62gw
# Background & Motivation
## What do we need to do

We are designing a new MQTT protocol architecture model so that RocketMQ can better support messages from terminals such as IoT devices and Mobile APP.

## Why should we do that

- What scenarios will be covered in this article?

The MQTT protocol is widely used in IoT devices and Mobile APP. Currently, RocketMQ cannot support MQTT protocol access well. Users need to build a separate MQTT system, and then synchronize MQTT message data to RocketMQ for back-end data processing. Storing message data in multiple systems increases storage costs and data consistency challenges.

- Are there any problems of our current project?

We all know that RocketMQ is mainly used for message distribution between back-end services, but there are still the following problems when applied to MQTT terminal message communication:
1. MQTT standard protocol access is not yet supported.
2. In the scenario of failed re-send, the MQTT terminal message must be re-sent to a certain terminal device explicitly, while the server-side message scenario can be re-sent to any server, and the reliable re-send mechanism is not satisfied yet.
3. The broadcast ratio of MQTT terminal messages may be much larger than that of server-side messages
4. The number of MQTT terminal devices connected is much larger than the number of back-end servers.
5. The client device environment is limited and can only accept simple message Push.

- What can we benefit from proposed changes?  

After the implementation of the new MQTT protocol implementation model, RocketMQ can support both terminal and server-side scenario messages. Messages do not need to be stored in multiple systems, which not only reduces costs but also eliminates data consistency problems.

# Goals
- What problem is this proposal designed to solve?  

  Based on the RocketMQ message unified storage engine, it supports both MQTT terminal and server message sending and receiving.

# Non-Goals.

- What problem is this proposal NOT designed to solve? 
 
The adaptation and transformation of the RocketMQ storage layer is not included in this draft for the time being.

- Are there any limits of this proposal?

For authentication implementation, metadata services, etc., it does not include.

# Changes
## Architecture

![](https://s4.ax1x.com/2022/02/17/H5aGS1.png)

As shown in the figure above, we have designed the entire architecture in layers, in which the core implementation of the RocketMQ storage layer is: queue index distribution, location storage, and lightweight metadata. The protocol gateway computing layer can be independently loaded or deployed as a protocol plug-in, and interact with the storage layer based on the existing Send/Pull standard interface.

### Queue Model

![](https://s4.ax1x.com/2022/02/17/H5aawD.png)

The above figure describes the queue storage model. Messages can come from various access scenarios (such as RMQ on the server and MQTT on the client), but only one copy is written and stored in the commitlog, and then the queue index (ConsumerQueue) of multiple demand scenarios is distributed. ), for example, the server-side scenario (RMQ) can be used for traditional server-side consumption according to the first-level topic queue, and the client-side MQTT scene can consume messages according to MQTT multi-level topics and wildcard subscriptions.

Such a queue model can support the access and message sending and receiving of server and terminal scenarios at the same time, achieving the goal of integration.

Regarding the above queue distribution characteristics, RocketMQ RIP-28 is described in detail.

### Push-Pull Model

![](https://s4.ax1x.com/2022/02/17/H5asSI.png)

### Message Flow

The P node in the figure is a protocol gateway or broker plug-in, and the terminal device is connected to this gateway node through the MQTT protocol. Messages can be sent from various scenarios (RMQ/MQTT). After being stored in the topic queue, there will be a notify logic module to sense the arrival of the new message in real time, and then a message event (that is, the topic name of the message) will be generated to push the event to the gateway node. The gateway node performs internal matching according to the subscription status of the connected terminal devices, finds which terminal devices can match, and then triggers a pull request to the storage layer to read the message and push the terminal device.

### Real-time And Reliability

In terms of reliable reach and real-time performance, in the push-pull process in the above figure, the event notification mechanism is used to notify the gateway node in real time, and then the gateway node exchanges the message through the Pull mechanism, and then pushes it to the terminal device. The Pull+Offset mechanism can ensure the reliability of the message. This is the traditional model of RocketMQ. The terminal node passively accepts the Push from the gateway node, which solves the problem of lightweight terminal equipment.


### Queue Cache

There is also a Cache module in the above figure for the message queue cache, because in the large broadcast ratio scenario, if a queue Pull request is initiated for each terminal device, the read pressure on the broker is large, since each request is read for the same topic queue, the local queue cache can be reused.


## Interface Design/Change
- Method signature changes

Nothing specific.

- CLI command changes

Nothing specific.

- Log format or content changes

Nothing specific.


## Compatibility, Deprecation, and Migration Plan
- Are backward and forward compatibility taken into consideration?
  
  Currently, the MQTT 3.1.1 protocol version is supported, which is compatible with the 3.1 protocol version. We will consider supporting the MQTT 5.0 protocol in the future.

- Are there deprecated APIs?
  
  Nothing specific.

- How do we do migration?
  
The MQTT protocol is standard and open, and any client that meets the protocol standard can access it.

## Implementation Outline

We will implement the proposed changes by 2 phases.
### Phase 1 

Preliminarily implement the MQTT protocol model, and complete the unified sending and receiving of terminal and server messages based on RocketMQ.

### Phase 2
Transform the queue storage capacity of RocketMQ, support millions of queues, and improve storage performance.

# Rejected Alternatives

![](https://s4.ax1x.com/2022/02/17/H5dN3n.png)

## How does alternatives solve the issue you proposed?

Simply publish/subscribe based on RocketMQ's Topic to realize the publish and subscribe of MQTT client messages.

## Pros and Cons of alternatives

Pros: Based on the existing RocketMQ topic subscription and publishing, simple and fast.

Cons: Can only be pushed online, offline messages cannot be implemented, and the order cannot be guaranteed.

## Why should we reject above alternatives

We hope that RocketMQ has complete MQTT protocol support capabilities.
