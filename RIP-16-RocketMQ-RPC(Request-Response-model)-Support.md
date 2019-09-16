# Status
Current State: Proposed    
Authors: qqeasonchen    
Shepherds: duhengforever    
Mailing List discussion: dev, users    
Pull Request: none    
Released: 4.6.0   

# Background & Motivation
RPC is a powerful technique for constructing distributed, client-server based applications. Many companies use the MQ system to constructing their service bus, especially in the financial industry. We need to implement an RPC client based on MQ system ourselves because the MQ system doesn’t support RPC naturally. In the current version of rocketmq project, producer client only support to send a message to the broker and cannot wait a response message replied from the consumer. We wish RocketMQ can provide an RPC client implement to help developers building their applications conveniently and effectively.    

# Goals

## What problem is this proposal designed to solve?
(1) Design and implement an RPC interface based on RocketMQ Producer, which support send a message and then wait a response message replied from the consumer.

(2) Design and Implement a consumer which can automatically or manually send a reply message.

(3) Enable the broker to push a message to an explicit producer.

 

## To what degree should we solve the problem?
We wish developers can use RocketMQ easily to constructing their distributed applications which need RPC call.

# Non-Goals

## What problem is this proposal NOT designed to solve?
In this phase, this rip only supports a usable RPC call in most common scenarios.

## Are there any limits to this proposal?
Users may need to tune some client and broker’s configurations to achieve better performance.

# Changes

## Architecture

We plan to add RPC implement into client and broker modules. In client modules, we will add an RPC interface into DefaultMQProducer, and add reply logic into DefaultMQPushConsumer. In broker modules, we will add a processor to push reply message from consumer to explicit producer.

## Interface Design/Change

- Method signature changes

- Plan to add interfaces as follow:  
```
Message request(final Message msg, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException;
Message request(final Message msg, final SendCallback sendCallback, final long timeout) throws MQClientException, RemotingException, InterruptedException;
Message request(final Message msg, final MessageQueueSelector selector, final Object arg, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException;
Message request(final Message msg, final MessageQueueSelector selector, final Object arg, final SendCallback sendCallback, final long timeout) throws MQClientException, RemotingException, InterruptedException;
Message request(final Message msg, final MessageQueue MQ, final long timeout) throws MQClientException, RemotingException, MQBrokerException, InterruptedException;
Message request(final Message msg, final MessageQueue mq, final SendCallback sendCallback, long timeout) throws MQClientException, RemotingException, InterruptedException;

 
```
 


## Method behavior changes
No issue

## CLI command changes
No issue

## Log format or content changes
No issue

## Compatibility, Deprecation, and Migration Plan
 No issue

#Implementation Outline

We will implement the proposed changes by 1 phase.    

Phase 1
- Implement RPC feature in Producer
- Implement RPC feature in Consumer
- Implement RPC feature in Broker