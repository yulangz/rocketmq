In the last few years, we use the [Jira](https://issues.apache.org/jira/projects/ROCKETMQ)(migrated to Issues) to trace issues or collect the new idea from the community. It’s a nice way for a feature request but not very formal and easy to trace and manage. So We introduce the RIP - RocketMQ Improvement Proposal mechanism to replace the current Feature Request Process. 

RIPs should be used for significant user-facing or cross-cutting changes, not small incremental improvements. When in doubt, if a committer thinks a change needs a RIP, it does. 

It is easy to start a RIP from sending proposals to dev mail lists. And the whole procedure of a RIP may follow the process:

1. Create a proposal as the [template](https://docs.google.com/document/d/19JssoEGnNp1x9MoXVMoeGCWSBnBdyv97FuGcIH1fV1g/edit). Take the next available RIP number and give your proposal a descriptive heading. 

2. Start a [DISCUSS] thread on the Apache RocketMQ dev mailing list. Please ensure that the subject of the thread is of the format [DISCUSS] RIP-{your RIP number} {your RIP heading} The discussion should happen on the mailing list not on the wiki since the wiki comment system doesn't work well for larger discussions. In the process of the discussion, you may update the proposal. You should let people know the changes you are making. When you feel you have a finalized proposal.

3. Once the proposal is finalized and some committers(Shepherd) are willing to support, please call a [VOTE] to have the proposal adopted. These proposals are more serious than code changes and more serious even than release votes. The criteria for acceptance is the lazy majority. The vote should remain open for 72 hours.

4. Please update the RIP wiki page, and the index below, to reflect the current stage of the RIP after a vote. This acts as the permanent record indicating the result of the RIP (e.g., Accepted or Rejected). Also, report the result of the RIP vote to the voting thread on the mailing list so the conclusion is clear.

### RIP List

| RIP Number |Description| Accepted | Activity |Release|
| ------ | ------ | ------ |------ |------ |
| RIP-1|[RocketMQ MQTT Bridge](https://github.com/apache/rocketmq/wiki/RIP-1-MQTT-Bridge)| yes| inactive | no |
| RIP-2 |[RocketMQ Spring](https://github.com/apache/rocketmq/wiki/RIP-2-RocketMQ-Spring) | yes|active|yes|
| RIP-3 |[RocketMQ Python Client](https://github.com/apache/rocketmq/wiki/RIP-3-RocketMQ-Python-Client) | yes|active|yes|
| RIP-4 | [RocketMQ Go Client](https://github.com/apache/rocketmq/wiki/RIP-4-RocketMQ-Go-Client)|yes |active|yes|
| RIP-5 | [RocketMQ Message Tracing](https://github.com/apache/rocketmq/wiki/RIP-6-Message-Trace)|yes|active|yes|
| RIP-6 | [RocketMQ ACL](https://github.com/apache/rocketmq/wiki/RIP-5-RocketMQ-ACL)| yes|active|yes|
| RIP-7 |[Multiple Directories Storage Support](https://github.com/apache/rocketmq/wiki/RIP-7-Multiple-Directories-Storage-Support) |yes|active|yes|
| RIP-8 |[Consumer RACK Support](https://github.com/apache/rocketmq/wiki/RIP-8-Consumer-RACK-Support) |yes |inactive|no|
| RIP-9 |[RocketMQ Develop Guide](https://github.com/apache/rocketmq/wiki/RIP-9-RocketMQ-Developer-Guide) |yes |active|yes|
| RIP-10 |[RocketMQ Unit Test](https://github.com/apache/rocketmq/wiki/RIP-10-RocketMQ-Unit-Test) |yes |active|yes|
| RIP-11 |[Evolution of The Next Decade Architecture for RocketMQ](https://github.com/apache/rocketmq/wiki/RIP-11-Evolution-of-The-Next-Decade-Architecture-for-RocketMQ) |yes |active|no|
| RIP-12 |[RocketMQ Replicator](https://github.com/apache/rocketmq/wiki/RIP-12-Message-Connector) |yes |active|yes|
| RIP-13 |[RocketMQ Console Project](https://github.com/apache/rocketmq/wiki/RIP-13-RocketMQ-Console-Project) |yes |active|yes|
| RIP-14 |[RocketMQ Community Operation Conventions](https://github.com/apache/rocketmq/wiki/RIP-14-RocketMQ-Community-Operation-Conventions) |yes |active|yes|
| RIP-15 |[RocketMQ IPv6 Support Project](https://github.com/apache/rocketmq/wiki/RIP-15-RocketMQ-IPv6-Support-Project) |yes |active|yes|
| RIP-16 |[RocketMQ RPC(Request-Response model) Support](https://github.com/apache/rocketmq/wiki/RIP-16-RocketMQ-RPC(Request-Response-model)-Support) |yes |active|yes|
| RIP-17 |[RocketMQ HTTP Proxy Support](https://github.com/apache/rocketmq/wiki/RIP-17-RocketMQ-HTTP-Proxy-Support) |yes |inactive|no|
| RIP-18 |[Metadata management architecture upgrade](https://github.com/apache/rocketmq/wiki/RIP-18-Metadata-management-architecture-upgrade) |yes|inactive|no|
| RIP-19 |[Server-side rebalance,  lightweight consumer client](https://github.com/apache/rocketmq/wiki/%5BRIP-19%5D-Server-side-rebalance,--lightweight-consumer-client-support) |yes|active|yes|
| RIP-20 |[Streaming Tiered Storage Optimize](https://github.com/apache/rocketmq/wiki/RIP-20-Streaming-Tiered-Storage-Optimize) |yes|active|no|
| RIP-21 |[Logical Queue Abstraction for Static Topic and Fast Scale-out](https://github.com/apache/rocketmq/wiki/RIP-21-logical-queue-abstraction-for-static-topic-and-fast-scale-out) |yes|active|yes|
| RIP-26 |[Improve Batch Message Processing Throughput](https://github.com/apache/rocketmq/wiki/RIP-26-Improve-Batch-Message-Processing-Throughput) |yes|active|yes|
| RIP-27 |[Auto batching in producer](https://github.com/apache/rocketmq/wiki/RIP-27-Auto-batching-in-producer) |yes|active|no|
| RIP-28 |[Light message queue (LMQ)](https://github.com/apache/rocketmq/wiki/RIP-28-Light-message-queue-(LMQ)) |yes|active|yes|
| RIP-29 |[Optimize RocketMQ NameServer](https://github.com/apache/rocketmq/wiki/RIP-29-Optimize-RocketMQ-NameServer) |yes|active|no|
| RIP-30 |[Support Compaction topic](https://github.com/apache/rocketmq/wiki/RIP-30-Support-Compaction-topic) |yes|active|no|
| RIP-31 |[Support RocketMQ BrokerContainer](https://github.com/apache/rocketmq/wiki/RIP-31-Support-RocketMQ-BrokerContainer) |yes|active|no|
| RIP-32 |[Slave Acting Master Mode](https://github.com/apache/rocketmq/wiki/RIP-32-Slave-Acting-Master-Mode) |yes|active|no|
| RIP-33 |[RocketMQ MQTT](https://github.com/apache/rocketmq/wiki/RIP-33-RocketMQ-MQTT) |yes|active|no|
| RIP-34 |[Support quorum write and adaptive degradation in master slave architecture](https://github.com/apache/rocketmq/wiki/RIP-34-Support-quorum-write-and-adaptive-degradation-in-master-slave-architecture) |yes|active|no|
| RIP-37 |[New and Unified APIs](https://shimo.im/docs/m5kv92OeRRU8olqX) |yes|active|no|
| RIP-38 |[RocketMQ EventBridge](https://docs.google.com/document/d/1RWPeORHY_-ukq8qs1a1lH80fH8vSQ44U1R9xbxgEX_c/) |yes|active|no|
| RIP-39 |[Support gRPC protocol](https://shimo.im/docs/gXqmeEPYgdUw5bqo) |yes|active|no|




