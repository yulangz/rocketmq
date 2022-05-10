
## Status
* Current State: development
* Authors: YuanchenZhang(Github:cryptoya)
* Shepherds: duhengforever@apache.org dinglei@apache.org
* Mailing List discussion: dev@rocketmq.apache.org
* Pull Request:
* Released: no
## Background & Motivation
### What do we need to do
* e2e is an essential part of the test environment, we need to add a rocketmq-e2e repository under Apache RocketMQ to write e2e use cases
* The process of adding GitHub Action under the Apache RocketMQ repository, triggering during Merge Request and Push, including but not limited to unit testing and e2e testing
### Why should we do that
* Supplement the problem that the current e2e test is not sound
* Resolve possible compatibility issues
* Expand E2E testing of subsequent clients in other languages
* Use GitHub Action to better integrate into GitHub's existing ecosystem
## Goals
### What problem is this proposal designed to solve?
The main purpose of end-to-end (E2E) testing is to verify the integration, data integrity, and compatibility of Broker, NameServer, and Client by simulating real user scenarios, and to test from the use of end users.
### To what degree should we solve the problem?
The JAVA client's externally exposed interface coverage rate is > 95%, which can cover compatibility tests of multiple major versions to ensure that the trunk branch iteration will not cause compatibility problems
## Non-Goals
### What problem is this proposal NOT designed to solve?
E2E testing is a supplement to the current unit testing and set testing, not a substitute for other testing methods.
### Are there any limits of this proposal?
Due to the use of different code bases, updates to major functions need to synchronously add test cases to the E2E code base to ensure continuous iteration of test cases. At the same time, the user's usage scenarios are complex, and it is difficult to combine coverage of all scenarios and high coverage of interfaces.
## Changes
We will provide a new project named rocketmq-e2e. Read below sections to get more details about the e2e test for RocketMQ.
### Architecture
![](https://s1.ax1x.com/2022/05/10/ONASvq.png)

### E2E-CICD solution 
At present, we use Travis for CICD. In order to better integrate the GitHub ecosystem and the continuous development of CICD, we will gradually convert to using GItHub Action for CICD.
When the code is submitted, it will trigger the workflow of the RocketMQ warehouse configuration. After the unit test is passed, the Linux operating system provided by GitHub is used to deploy the service (Broker, NameServer). After the deployment is completed, the e2e use case in the rocketmq-e2e project will be triggered to run, which has two advantages
1. Optimize some Commit's irregular behavior in RocketMQ testing. In the original way of use, part of the submission will modify the use case synchronously in order to make the modified server code pass the test, so that the use case seems to run successfully. In fact, the modified interface may appear incompatible with the published client and other unexpected behaviors. .
2. The compatibility of multiple versions of the client can be guaranteed. When the Commit is submitted, the latest version of the server built by the Commit is deployed, and multiple released important versions of the Client are used to test at the same time to ensure that there is no compatibility problem.
### RoceketMQ-E2E
This engineering architecture is based on the transformation and implementation of Alibaba Cloud's commercialized client, which is implemented using java language client + Junit5 framework. Some of the use case scenarios are from some internal use cases of basic use cases, and some are from the existing use cases of the community.
## Rejected Alternatives
No other alternatives
