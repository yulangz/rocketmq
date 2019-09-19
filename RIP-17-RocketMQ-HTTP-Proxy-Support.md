# Status

Current State: Proposed    
Authors: qqeasonchen    
Shepherds: duhengforever    
Mailing List discussion: dev@rocketmq.apache.org; users@rocketmq.apache.org    
Pull Request: RIP-17   
Released: 4.7.0（rocketmq-external first）    
  
# Background & Motivation
## What do we need to do
 
RocketMQ may access clients written in different languages. In order to meet the needs of heterogeneous systems to access RocketMQ, this proposal solve this problem by adding HTTP proxy module. At the same time, some logic such as flow control/ACL on broker and some process logic on client can be moved to proxy, which makes the broker and client lighter.

# Goals
## What problem is this proposal designed to solve?

In the future, RocketMQ may access clients written in different languages. At present, due to the complexity of the client logic, it takes a lot of effort to rewrite the client in different languages. And the persistent connection in RocketMQ is based on TCP, and the ability to resist network jitter is bad. HTTP connection can better resist network jitter. In order to write and access clients in different languages better, this proposal provides an HTTP sending proxy, which simplifies the processing logic of the client and implements a more lightweight client.

# Non-Goals
## What problem is this proposal NOT designed to solve?

    Nothing specific

## Are there any limits of this proposal?

    User need deploy proxy before use it. It will add a little complexity for operation and maintenance.

## Changes
    Adding a proxy module to achieve message proxy.

    This RIP will be divided into two phases.
    * The first phase will not have any impact on the existing architecture, including the nameserver, depending on the service discovery configured to do the proxy.
    * The second phase will decide whether to do proxy service discovery based on community feedback or provide a lightweight http client.

In our proposal, there are 3 features in the proxy:

(1)   We realize group proxy in producer group/consumer group level.    
(2)   The proxy receives HTTP packets sent by HTTP producer, which realizes HTTP proxy and reduces the impact of network jitter.    
(3)   The proxy simplifies processing logic in client and achieves lighter client.     