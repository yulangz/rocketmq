# Status

- Current State: Proposed
- Authors: [drpmma](https://github.com/drpmma), [xdkxlk](https://github.com/xdkxlk)
- Shepherds: [zhouxinyu](https://github.com/zhouxinyu), [lizhanhui](https://github.com/lizhanhui), [lollipopjin](https://github.com/lollipopjin)
- Mailing List Discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)
- Pull Request: #PR_NUMBER
- Released: <released_version>

# Background & Motivation

## What do we need to do

- Will we add a new module? No.
- Will we add new APIs? No.
- Will we add a new feature? Yes.

## Why should we do that

**Are there any problems with our current project?**

The remoting protocol is not supported in proxy yet. And it should be put on the agenda. In this way, both the protocol and the architecture are aligned in a unified way.


**What can we benefit from proposed changes?**

With remoting protocol in proxy, the remoting client is able to connect to proxy in the unified architecture, which is stateless and has separate computing and storage. The proxy will focus on traffic management, connection management, and observability while the broker pays more attention to performance, storage, low latency, and high availability.


# Goals

**What problem is this proposal designed to solve?**

- Supporting RPC in remoting protocol related to messaging and connection.

# Non-Goals

- Do not support admin interface. The methods related to the admin interface are considered internal operations and should follow the way of connecting to namesrv.
- Do not support request-reply cause it's not widely used and hasn't reached maturity.

# Changes

## Architecture

![image](https://user-images.githubusercontent.com/20906038/196327732-aa0d0042-c8a1-49ba-b863-266f09ee0ba5.png)


## Interface Design/Change

The request codes are planned to implement.
```text
SEND_MESSAGE
PULL_MESSAGE
QUERY_MESSAGE
QUERY_CONSUMER_OFFSET
UPDATE_CONSUMER_OFFSET
SEARCH_OFFSET_BY_TIMESTAMP
GET_MAX_OFFSET
GET_MIN_OFFSET
GET_EARLIEST_MSG_STORETIME
HEART_BEAT
UNREGISTER_CLIENT
CONSUMER_SEND_MSG_BACK
END_TRANSACTION
GET_CONSUMER_LIST_BY_GROUP
CHECK_TRANSACTION_STATE
LOCK_BATCH_MQ
UNLOCK_BATCH_MQ
POP_MESSAGE
ACK_MESSAGE
PEEK_MESSAGE
CHANGE_MESSAGE_INVISIBLETIME
NOTIFICATION
POLLING_INFO
GET_ROUTEINFO_BY_TOPIC
CONSUME_MESSAGE_DIRECTLY
SEND_MESSAGE_V2
SEND_BATCH_MESSAGE
LITE_PULL_MESSAGE
```

- Method signature changes? Nothing specific.

- Method behavior changes? Nothing specific.

- CLI command changes? Nothing specific.

- Log format or content changes? Nothing specific.

## Compatibility, Deprecation, and Migration Plan

- Are backward and forward compatibility taken into consideration? No
- Are there deprecated APIs? No
- How do we do migration? No

## Implementation Outline

### Phase 1

The remoting server and protocol negotiation.

### Phase 2

Connection implementation.

### Phase 3

Messaging implementation