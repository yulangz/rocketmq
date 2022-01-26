# Status
- Current State: Accept
- Authors: [RongtongJin](https://github.com/RongtongJin)
- Shepherds: duhengforever@apache.org, dinglei@apache.org
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: 
- Released: <relased_version>
- Related Docs: [Optimize RocketMQ NameServer](https://shimo.im/docs/pXgKrCwxhCcTwPkx/)

# Background & Motivation
What do we need to do
- Will we add a new module?    
  No.
- Will we add new APIs?    
  No.
- Will we add a new feature?   
  Yes.


Why should we do that

- Are there any problems of our current project?

Nameserver is a very important component in RocketMQ cluster, which is used for route discovery. At present, the nameserver is stateless and lightweight, but it still bears a certain amount of pressure especially when the cluster reaches a certain scale. RIP-29 will optimize RocketMQ nameserver in rocketmq 5.0 from many aspects.

- What can we benefit from proposed changes?  

 1. By separating the broker registration thread pool and the topic route info acquisition thread pool, we can ensure that different types of requests will not affect each other.
 2. Optimize topic routing cache to speed up topic routing acquisition, reduce nameserver CPU pressure.
 3. Unregister brokers in batches  to speed up the broker offline.


# Goals
- What problem is this proposal designed to solve?  
  Optimize nameserver through various aspects.
# Non-Goals.
- What problem is this proposal NOT designed to solve?  
 1. Add new features to nameserver.
 2. Affect compatibility
- Are there any limits of this proposal?  
  Only newer clients with changes in this proposal will benefit.

# Changes
## Architecture

### Current "Consume Queue index":

- Thread pool separation

![](https://s4.ax1x.com/2022/01/24/7ohj6s.png)

Currently, nameserver uses the same thread pool and queue to process all client routing requests, broker registration requests, and the queue size and a number of threads are not configurable. If one of the types of requests bursts the thread pool, it will affect all requests. RIP-29  isolates the most important client routing requests separately. The size of the queue and the number of threads are configurable. The request processing between the thread pools is isolated from each other and is not affected

- Topic route cache optimization

At present, when the client sends a routing request, the nameserver will use topicQueueTable and brokerAddrTable to construct the final TopicRouteData. This involves traversing the broker in the read lock, which has a certain CPU consumption. RIP-29 directly returns the topic route when requested by the client by constructing the cached of the topic route(topicRouteDataMap). The additional cost is to modify the topicRouteDataMap when the broker register, the broker offline  deletes topics, etc. For the nameserver, the read operation is more frequent than the write operation, so the cost is worth it.

- Batch unregister broker in nameserver

Add BatchUnRegisterService to batch processing of broker offline and accelerate the broker offline.

## Interface Design/Change
- Method signature changes

  Nothing specific.

- Method behavior changes

  Nothing specific.

- CLI command changes

  Nothing specific.

- Log format or content changes

  Add new log to print nameserver's watermark

## Compatibility, Deprecation, and Migration Plan
- Are backward and forward compatibility taken into consideration?
  
  Yes, there are no nameserver's functional changes and compatibility issues

- Are there deprecated APIs?
  
  Nothing specific.

- How do we do migration?
  
  Upgrade normally, no additional migration required

## Implementation Outline

We will implement the proposed changes by 1 phases.
### Phase 1 

1. Complete the thread pool separation
2. Complete topic routing cache optimization
3. Batch unregister broker in NameServer


# Rejected Alternatives

There are no other alternatives.