# Status

- Current State: Discussing
- Authors: [lizhimins](https://github.com/lizhimins)
- Shepherds: [lizhanhui](https://github.com/lizhanhui), [zhouxinyu](https://github.com/zhouxinyu)
- Mailing List Discussion: dev@rocketmq.apache.org
- Pull Request: #PR_NUMBER
- Released: <released_version>

# Background & Motivation

## What do we need to do

- Will we add a new module? No.

- Will we add new APIs? No.

- Will we add a new feature? Yes.

## Why should we do that

- Are there any problems with our current project?

Remoting module is a very important component in RocketMQ, which is used for component communication.

We propose some solutions to optimize.

1. Server can't solve the back pressure situation well.
2. Server need a simple way to get request and response distribution for sloving probleam.
3. When thread abnormal exit, we can't find reason because we do not have log.
4. In some special network scenarios, the client needs to connect to the server by proxy

- What can we benefit from proposed changes?

The features of remoting improved the above problems.

# Goals

- What problem is this proposal designed to solve?

We will add some new features and optimize codes for RocketMQ remoting module.

# Non-Goals

- What problem is this proposal NOT designed to solve?

# Changes

## Architecture

We have some small modifications on this module, for example:

1. Logging rpc distribution can help developer quickly locate related requests with problems

![1665298653453-632bf3d5-d2cb-4b5a-acd6-dc7880c4c335](https://user-images.githubusercontent.com/22487634/194985670-3ef1d330-a3eb-4610-afe2-dfcd7842e5e7.png)

2. In some special network scenarios, the client needs to connect to the server by proxy
3. When thread abnormal exit, output thread exception information to the error log
4. To avoid the system from going OOM we have to override the method channelWritabilityChanged of the ChannelInboundHandler and do back pressure handling. The outbound data will be queued in an internal buffer maintained by netty. [Referral links](https://netty.io/4.1/api/io/netty/channel/Channel.html#isWritable--)

## Interface Design/Change

- Method signature changes? Nothing specific.
- Method behavior changes? Nothing specific.
- CLI command changes? Nothing specific.
- Log format or content changes? 

We add new log to print rpc distribution in remoting module.s

## Compatibility, Deprecation, and Migration Plan

- re backward and forward compatibility taken into consideration? No
- Are there deprecated APIs? No
- How do we do migration? No

## Implementation Outline

We will implement the proposed changes in pull requests. 

# Rejected Alternatives

## How do alternatives solve the issue you proposed?

## Pros and Cons of alternatives

## Why should we reject above alternatives

