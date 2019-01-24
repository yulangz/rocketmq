# Status
Current State: Proposed  
Authors: chenhoudao, wlliqipeng  
Shepherds: dinglei(dinglei@apache.org)  
Mailing List discussion: users@rocketmq.apache.org;dev@rocketmq.apache.org  
Pull Request:<NULL>  
Released: <4.4.1>

# Background & Motivation
## What do we need to do
For RocketMQ project, Unit Test is very important. It can improve code quality by increase code coverage. So we're going to add a unit test for RocketMQ.

# Goals
## What problem is this proposal designed to solve?
We plan to use tools such as sonar to find out which modules, such as Store, Broker, Client, etc., are not covered by functions, and write unit test cases for these functions.

## To what degree should we solve the problem?
Increase RocketMQ  code coverage to 60%

# Non-Goals
## Are there any limits to this proposal?
Higher code coverage may be a bit more difficult

# Changes
We will add or modify some  test cases in test dir
