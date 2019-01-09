In the last few years, we use the [Jira](https://issues.apache.org/jira/projects/ROCKETMQ)(migrated to Issues) to trace issue or collect new idea from the community. It’s a nice way for a feature request but not very formal and easy to trace and manage. So We introduce the RIP - RocketMQ Improvement Proposal mechanism to replace the current Feature Request Process. 

RIPs should be used for significant user-facing or cross-cutting changes, not small incremental improvements. When in doubt, if a committer thinks a change needs a RIP, it does. 

It is easy to start a RIP from sending proposals to dev mail lists. And the whole procedure of a RIP may follow the process:

Create a proposal as the [template](). Take the next available RIP number and give your proposal a descriptive heading. 

Start a [DISCUSS] thread on the Apache RocketMQ dev mailing list. Please ensure that the subject of the thread is of the format [DISCUSS] RIP-{your RIP number} {your RIP heading} The discussion should happen on the mailing list not on the wiki since the wiki comment system doesn't work well for larger discussions. In the process of the discussion, you may update the proposal. You should let people know the changes you are making. When you feel you have a finalized proposal.

Once the proposal is finalized and some committers(Shepherd) are willing to support, please call a [VOTE] to have the proposal adopted. These proposals are more serious than code changes and more serious even than release votes. The criteria for acceptance is lazy majority. The vote should remain open for 72 hours.

Please update the RIP wiki page, and the index below, to reflect the current stage of the RIP after a vote. This acts as the permanent record indicating the result of the RIP (e.g., Accepted or Rejected). Also report the result of the RIP vote to the voting thread on the mailing list so the conclusion is clear.

### EN
* RIP-1 Accepted Inactive [RocketMQ MQTT Bridge](https://github.com/apache/rocketmq/wiki/RIP-1-MQTT-Bridge)
* RIP-2 Accepted Release [RocketMQ Spring](https://github.com/apache/rocketmq/wiki/RIP-2-RocketMQ-Spring)
* RIP-3 Accepted Release [RocketMQ Python Client](https://github.com/apache/rocketmq/wiki/RIP-3-RocketMQ-Python-Client)
* RIP-4 Accepted Release [RocketMQ Go Client](https://github.com/apache/rocketmq/wiki/RIP-4-RocketMQ-Go-Client)
* RIP-5 Accepted Release [RocketMQ Message Track](https://github.com/apache/rocketmq/wiki/RIP-6-Message-Track-Trace)

### CN
* RIP-6 Accepted [RocketMQ ACL](https://github.com/apache/rocketmq/wiki/RIP-5-RocketMQ-ACL)
