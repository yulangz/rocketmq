## Status
Current State: Proposed
Authors:[ aaron-ai](https://github.com/aaron-ai)
* Shepherds:[ lizhanhui](https://github.com/lizhanhui),[ zhouxinyu](https://github.com/zhouxinyu)
* Mailing List Discussion: dev@rocketmq.apache.org
* Pull Request: #PR_NUMBER
* Released: <released_version>
## Background & Motivation
### What do we need to do
* Will we add a new module? -- No.
* Will we add new APIs? -- No.
* Will we add a new feature? -- No.
### Why should we do that
#### Are there any problems with our current project?
 
Since gRPC/protobuf-based client has been released, more features will prefer to be developed on gRPC/protobuf-based client rather than the remoting-base client, but there are still many developers using the remoting-based client, which could not be ignored.
 
Alibaba Group has made many changes and optimizations to the client in its long-term production practices. Unfortunately, not all changes are merged into the upstream repository in time due to different release paces and strategies. 
 
In order to unify the code, while improving the stability of the client, backporting is essential.
	
#### What can we benefit from proposed changes?
 
More stable, unified remoting-based RocketMQ client.
## Goals
### What problem is this proposal designed to solve?
 
Improve the stability of the remoting-based client mainly.
## Non-Goals
### What problem is this proposal NOT designed to solve?
 
This RIP does not introduce new features to the remoting-based client.
## Changes
### Architecture
 
We will divide this RIP into multiple pull requests, which will:
* Fix an exponential duplication of message issue when messages were backed out to brokers on application process failure for future retry;
* Justify an offset management flaw;
* Fix several exceptions catch missing;
* Refactor XXX to follow single responsibility principle;
...
### Interface Design/Change
 
No interface changes involved.
### Compatibility, Deprecation, and Migration Plan
 
This RIP doesn't introduce any compatibility issues.
### Implementation Outline
 
This RIP is miscellaneous so that maybe it is not possible to include all specific topics in the proposal, you can see more details from pull requests soon.
## Rejected Alternatives
 
No other cadicate alternatives.