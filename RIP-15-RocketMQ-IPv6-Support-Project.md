# Status
Current State: Implemented    
Authors: fubaoyu,chenyongming    
Shepherds: duhengforever    
Mailing List discussion: users@rocketmq.apache.org;dev@rocketmq.apache.org    
Released: <4.6.0>    
 

# Background & Motivation
## What do we need to do
RocketMQ does not currently support running in an IPv6 network environment. Therefore, we plan to make certain modifications to RocketMQ so that it can be deployed in both IPv4 and IPv6 network environments.

# Goals
## What problem is this proposal designed to solve?
We plan to modify the RocketMQ system flag so that the Broker can choose different stored procedures based on the message IP version. For messages from different IP versions, the client can also decode correctly. The modified version works in both IPv4 and IPv6 network environments and is compatible with previous versions.

## To what degree should we solve the problem?
(1)Modify system flag;    
(2)Modify storage process;    
(3)Modify message encoding and decoding  process.    

# Non-Goals
## What problem is this proposal NOT designed to solve?
In this phase, the project will not support transactional message and message trace function. We will improve these functions in our subsequent work.

# Changes
## Enahncement 1: Sub-project: storage module
The storage structure of the message contains the IP. The storage module performs different storage processes depending on whether the message is from an IPv4 or IPv6 client.

## Enahncement 2: Sub-project: Message encoding and decoding module
When it comes to message encoding with IP information, for example, msgID, RocketMQ will call different encoding methods depending on the IP version. At the same time, the client will call different decoding methods for each message according to the system flag when decoding.

## new features
RocketMQ can be deployed in an IPv6 network environment.
