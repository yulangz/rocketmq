
## Status
* Current State: Development
* Authors: [Li Zhanhui](https://github.com/lizhanhui)
* Shepherds: -
* Mailing List discussion: dev@rocketmq.apache.org
* Pull Request:
* Released: no
## Background & Motivation
### What do we need to do
* Will we add a new module? -- No.
* Will we add new APIs? -- No.
* Will we add new features? -- No.
### Why should we do that

#### Are there any problems with our current project?
Yes.

Clients prior to 5.x shares the exactly same data layout with store module on the server side, strictly blocking them evolving independently. Aka, brokers cannot make any changes to storage data layout unless all consumers are upgraded first, as is virtually impossible in practice. This imposes a series of blocking challenges to development of RocketMQ.

There has been a few known limitations

![Current Data Layout](https://raw.githubusercontent.com/wiki/apache/rocketmq/assets/DataLayout_overall-v1.drawio.png)

1. Length of topic can be up to 128 bytes only; 
2. Occasional properties length run-over after more system properties are appended.
3. Deleted topic re-appear after system crash recovery; ABA issue is impossible to resolve under current data layout paradigm. For example, create a topic T, send a few messages, delete T, then re-create topic T and send another batch of messages; Once system crash and recover, it's hard to maintain data integrity.
4. Current data layout assumes IPv4 in terms of born host and store host. It is awkward to make it IPv6 compatible. 
5. System properties and user defined key-value pairs are mangled together. Newly added system properties have the risk of conflicting with existing user code.

#### What can we benefit from proposed changes?

* All issues listed in the previous section will be fundamentally resolved.

## Goals
### What problem is this proposal designed to solve?
* See the previous section, problems stated will be solved.
## Non-Goals
### What problem is this proposal NOT designed to solve?
N/A

### Are there any limits of this proposal?
Nothing specific.
## Changes
### Architecture

1. Current data layout is named v1. Brokers will, by default, assumes that client SDKs support v1.
2. When messages stored in v2 format are to deliver to SDKs with v1 capability, they would be reflowed in v1 format.
3. On connection, new SDKs would sync its capabilities to brokers, such that brokers may deliver messages in v2 format directly.
4. Serialization and deserialization of v2 are supposed to be almost zero overhead. Overall, its format are as follows. ![V2 Format](https://raw.githubusercontent.com/wiki/apache/rocketmq/assets/DataLayout_overall-v2.drawio.png)
5. [FlatBuffers](https://google.github.io/flatbuffers/) is targeted for message header serialization and deserialization because 1) it has well supports among popular programming languages; 2) its IDL is backward and forward compatible when adding/removing fields; 3) its extremely [good performance](https://google.github.io/flatbuffers/flatbuffers_benchmarks.html)


### Interface Design/Change

There is no API/interface change involved.

### Compatibility, Deprecation, and Migration Plan

Compatibility is kept in mind by design. Future new SDK would enjoy all benefits while released client SDKs would still be supported as before.

### Implementation Outline
We split this proposal into several tasks:
* Task1: Implement client capability sync on connection.
* Task2: Discuss v2 data layout, message header IDL.
* Task3: Store message in v2. Deliver messages according to client capabilities;
* Task4: Release remoting-based client SDK that support v2 data layout.

## Rejected Alternatives

Keep the status quo, and continue with mentioned blockers.

### How do alternatives solve the issue you proposed?
