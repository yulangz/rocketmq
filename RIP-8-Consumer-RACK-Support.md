# Status
Current State: Proposed     
Authors: Jianfeng Wu     
Shepherds: Zhendong Liu     
Mailing List discussion: <apache mailing list archive>     
Pull Request: #PR_NUMBER     
Released: <relased_version>    
  
# Background & Motivation
## What do we need to do
## Will we add a new module? 
No
## Will we add new APIs?
No
## Will we add new feature?

# Why should we do that
## Are there any problems of our current project?
Yes, In our testing environment, we need test different feture in the same time, so we need deploy different consumer logic instance in the different test environment for test different feture, but message will be consumed not our expect consumer instance.
## What can we benefit proposed changes?
Improve consumer in same Consumer Group Name to support different RACK, producer can specify message RACK when send message.

# Goals
## What problem is this proposal designed to solve?
The producer can't specify who consumer instance can consumed when send message.
## To what degree should we solve the problem?

# Non-Goals
## What problem is this proposal NOT designed to solve?

## Are there any limits of this proposal?
 
# Changes
# Architecture

This solution I proposed consists of two operations: send message and consumer message strategy.    
## 1.  send message 

      Specify message for which RACK when producer send.
## 2.  consumer message 

      consumer instance can specify RACK or default RACK, if specify RACK, this consumer will only consume specify the same RACK message, if the message not have consumer specify same RACK, will consumed by default RACK consumer.

# Implementation Outline
  
