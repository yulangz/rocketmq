# Status
- Current State: Merged
- Authors: [tianliuliu](https://github.com/tianliuliu)
- Shepherds: RongtongJin
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: [#3694](https://github.com/apache/rocketmq/pull/3694)
- Released: 4.9.3
- Related Docs: [Light message queue (LMQ)](https://docs.google.com/document/d/1wq7crKF67fWv5h13TPHtCpHs-B9X8ZmaA-RM6yVbVbY/edit#)

# Background & Motivation
## What do we need to do

Some messaging scenarios require light message queue to support a large number of topics, such as MQTT, AMQP protocol, their MQTT multi-level topic or AMQP lightweight queue can be set at will by users when sending and subscription message. Let's call them light message queue(LMQ) for now ,and the number of LMQ is very large, the original RocketMQ topic is resource-heavy, and it is difficult to support millions or more of LMQ.

We are providing a new solution about for reliable and real-time message service for IoT devices and AMQP protocol users. We are addressing the issues about reliability, latency and availability.
About the Apache RocketMQ project, we will build a new feature which enables largelager number of LMQ to support MQTT and AMQP functions.


# Goals
- What problem is this proposal designed to solve?  

1.Light message queue (LMQ) needs to implement multi-dimensional dispatch of a message based on ConsumeQueue, and a message can support multiple lightweight queue consumption.

2.Light message queue (LMQ) needs to implement multi-level topic management, so topic metadata also needs adaptation management, such as topic verification, data cleaning and so on.

3.Light message queue (LMQ) needs to implement lightweight queue consumption offset management.

# Non-Goals.
- What problem is this proposal NOT designed to solve?  
  LMQ does not register nameServer, which is not visible to users and does not support finding routes to send messages.
- Are there any limits of this proposal?  
  The proposed solution LMQ deployed in a broker process that uses some switches to turn on. 

# Changes

We will provide a new feature that enables a larger number of LMQ to support MQTT and AMQP functions.

## Architecture

[![Tx1P39.png](https://s4.ax1x.com/2022/01/06/Tx1P39.png)](https://imgtu.com/i/Tx1P39)

As shown in the figure above, multi-consumeQueue atomic dispatch which is marked as red color will dispatch a normal topic message to many consumer queues.


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
 
  Regardless of whether the LMQ switch is turned on or off, the original RocketMQ client and message sending and receiving will not be affected.


## Implementation Outline
We will implement the proposed changes by 2 phases.
### Phase 1 

Light message queue (LMQ) support multi-dimensional dispatch of a message based on ConsumeQueue.
Light message queue (LMQ) support multi-level topic management, so topic metadata also needs adaptation management, such as topic verification, data cleaning and so on.
Light message queue (LMQ) support lightweight queue consumption offset management.
Light message queue (LMQ) support pull message

### Phase 2 
Light message queue (LMQ) support pop message
