Below is Markdown text with some GFM syntax.

# Status
- Current State: Proposed
- Authors: [ayanamist](https://github.com/ayanamist)
- Shepherds: duhengforever@apache.org,vongosling@apache.org
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: #PR_NUMBER
- Released: <relased_version>
# Background & Motivation
What do we need to do
- Will we add a new module?
No.
- Will we add new APIs?
It will add a new constant to mock broker names for logical queues.
- Will we add a new feature?
Yes.

Why should we do that
- Are there any problems with our current project?
Currently, the MessageQueue of RocketMQ is coupled with broker name, which
results that the queue number will change if broker number increases or
decreases, which causes all queues to rebalance, which may cause service
disruption like flink job restarts in minutes.
- What can we benefit from proposed changes?
The number of logical queues is not related with the number of brokers: We
can increase broker number without changing logical queue number, moreover,
we can increase logical queue number without deploying a new broker.
# Goals
- What problem is this proposal designed to solve?
Provide an abstraction, logical queue, to decouple between queue number and
broker number.
- To what degree should we solve the problem?
We should not hurt availability or performance in the implementation.
# Non-Goals
- What problem is this proposal NOT designed to solve?
We will not improve the mechanism of queues rebalance.
- Are there any limits of this proposal?
Only newer clients with changes in this proposal will benefit.
# Changes
## Architecture

We use one or more MessageQueue to make one LogicalQueue; One LogicalQueue
is unique in one topic, and one MessageQueue belongs to one and only one
LogicalQueue.

| brokerName | MessageQueue | LogicalQueue |
|------------|--------------|--------------|
| broker1    | 0            | 0            |
| broker1    | 1            | 1            |
| broker2    | 0            | 2            |
| broker2    | 1            | 3            |

After one LogicalQueue migrated from broker1 to broker2, there will be two
MessageQueues for one LogicalQueue. We only migrate mapping but not actual
data, so broker1 is still serving for old data consuming but not data
producing.

| brokerName | MessageQueue | LogicalQueue | QueueStatus |
|------------|--------------|--------------|--------------------|
| **broker1**    | **0**            | **0(0-100)**         | **ReadOnly**
        |
| broker1    | 1            | 1            | Normal             |
| broker2    | 0            | 2            | Normal             |
| broker2    | 1            | 3            | Normal             |
| **broker2**    | **2**            | **0(101-)**         | **Normal**
        |

After broker1 cleans all data from the commit log and consume queue,
QueueStatus becomes Expired(neither readable nor writable).

| brokerName | MessageQueue | LogicalQueue | QueueStatus |
|------------|--------------|--------------|-------------|
| **broker1**    | **0**            | **0(-)**         | **Expired**     |
| broker1    | 1            | 1            | Normal      |
| broker2    | 0            | 2            | Normal      |
| broker2    | 1            | 3            | Normal      |
| broker2    | 2            | 0(101-)      | Normal      |

If this LogicalQueue is migrated back to broker1, it will reuse this
expired MessageQueue

| brokerName | MessageQueue | LogicalQueue | QueueStatus |
|------------|--------------|--------------|-------------|
| **broker1**    | **0**            | **0(201-)**      | **Normal**      |
| broker1    | 1            | 1            | Normal      |
| broker2    | 0            | 2            | Normal      |
| broker2    | 1            | 3            | Normal      |
| **broker2**    | **2**            | **0(101-200)**   | **ReadOnly**    |

If this LogicalQueue is migrated back to broker1 while MessageQueue not
expired, it will create a new MessageQueue

| brokerName | MessageQueue | LogicalQueue | QueueStatus |
|------------|--------------|--------------|-------------|
| **broker1**    | **0**            | **0(0-100)**     | **ReadOnly**    |
| broker1    | 1            | 1            | Normal      |
| **broker1**    | **2**            | **0(201-)**      | **Normal**      |
| broker2    | 0            | 2            | Normal      |
| broker2    | 1            | 3            | Normal      |
| **broker2**    | **2**            | **0(101-200)**   | **ReadOnly**    |

If broker2 is offlined, all LogicalQueue in this broker should be migrated
away.

| brokerName | MessageQueue | LogicalQueue | QueueStatus |
|------------|--------------|--------------|-------------|
| broker1    | 0            | 0            | Normal      |
| broker1    | 1            | 1            | Normal      |
| **broker1**    | **2**            | **2(101-)**      | **Normal**      |
| **broker1**    | **3**            | **3(101-)**      | **Normal**      |
| **broker2**    | **0**            | **2(0-100)**     | **ReadOnly**    |
| **broker2**    | **1**            | **3(0-100)**     | **ReadOnly**    |

When all data including commit log and consume queue in broker2 are
cleaned, broker2 can be removed.

| brokerName | MessageQueue | LogicalQueue | QueueStatus |
|------------|--------------|--------------|-------------|
| broker1    | 0            | 0            | Normal      |
| broker1    | 1            | 1            | Normal      |
| **broker1**    | **2**            | **2(101-)**     | **Normal**      |
| **broker1**    | **3**            | **3(101-)**      | **Normal**      |

When a new broker is deployed, we can migrate some LogicalQueues to this
broker to spare some producing traffic.

| brokerName | MessageQueue | LogicalQueue | QueueStatus |
|------------|--------------|--------------|-------------|
| broker1    | 0            | 0            | Normal      |
| broker1    | 1            | 1            | Normal      |
| **broker1**    | **2**            | **2(101-200)**   | **ReadOnly**    |
| **broker1**    | **3**            | **3(101-200)**   | **ReadOnly**    |
| **broker3**    | **0**            | **2(201-)**      | **Normal**      |
| **broker3**    | **1**            | **3(201-)**      | **Normal**      |

All mapping data are stored separately in brokers, and via registerBroker
to be gathered in namesrv; Each broker only stores its own logical queue
mapping but not other one's. All management operations should be invoked by
CLI or direct rpc command, automatic operation support is missing now, and
require [RIP-18](
https://github.com/apache/rocketmq/wiki/RIP-18-Metadata-management-architecture-upgrade)
to be implemented first.

## Interface Design/Change
- Method signature changes
No.
- Method behavior changes
When a topic is enabled LogicalQueue support, broker name of MessageQueue
result returned by some methods like `fetchSubscribeMessageQueues` or
`fetchPublishMessageQueues` will be a fake one, since LogicalQueue does not
have broker name concept.
- CLI command changes
Add some operation command for LogicalQueue, like
`UpdateLogicalQueueMapping` `DeleteLogicalQueueMapping`
`QueryLogicalQueueMapping` `UpdateLogicalQueueNum` `MigrateLogicalQueue`.
Also `updateTopic` adds a new parameter to enable LogicalQueue support
immediately after the topic is created.
- Log format or content changes
No.
## Compatibility, Deprecation, and Migration Plan
- Are backward and forward compatibility taken into consideration?
Yes.
- Everything will work well if no topic is enabled LogicalQueue support,
whether on old/new broker/namesrv/client.
- LogicalQueue support will work only under new broker+namesrv+client,
otherwise, everything will work like no LogicalQueue support.
- Are there deprecated APIs?
No.
- How do we do migration?
No need to migrate, this is a feature which needs to be enabled manually.
## Implementation Outline
We will implement the proposed changes by 2 phases.
### Phase 1 basic function
1. Implement LogicalQueue routes query and merge mechanism in namesrv.
2. Implement LogicalQueue mapping report from broker to namesrv.
3. Implement LogicalQueue support in the client.
4. Implement LogicalQueue migration in broker.
5. Implement LogicalQueue admin processor and CLI.
### Phase 2 compatibility
1. Implement proxy request+response for old client in broker.
2. Implement intermediate state(support is partly enabled in some brokers
but not all) protection in namesrv.
# Rejected Alternatives
- How does alternatives solve the issue you proposed?
Implement a whole new Logical Queue architecture from scratch, this
absolutely will solve the problem.
- Pros and Cons of alternatives
Pros: from-scratch way does not bring anything good.
Cons: from-scratch way will break existent concept, add much more
complexity to code and not user-friendly.
- Why should we reject the above alternatives
It does no good.