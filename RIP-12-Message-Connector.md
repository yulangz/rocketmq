# Status
Current State: Discuss    
Authors: HuZongTang, WangShaoJie     
Shepherds: duhengforever    
Mailing List discussion: users@rocketmq.apache.org;dev@rocketmq.apache.org    
Pull Request:    
Released:        

# Background & Motivation
## What do we need to do
In current version of rocketmq project, different cluster can't exchange messages each other. Although dLedger structure improves availability of rocketmq much more, it still can't implement backuping messages for different cluster.       
So, we need a mechanism to implement backuping messages for different cluster. It can improve availability of multiple rocketmq cluster. Here, we called it Message Connector or RocketMQ Replicator.   

# Goals
## What problem is this proposal designed to solve?
We plan to design a component call Message Connector for RocketMQ. User can define the topic message routing rule by configing file, such as source topic, targe topic, filter tag , source cluster and targe cluster info.    
    
## To what degree should we solve the problem?
      Define some rules in configing files;
      Better availability for messages in multiple cluster enviroment.
      Monitering some parameters in the process of message replication;

# Non-Goals
## What problem is this proposal NOT designed to solve?
     In this phase, this rip only support starting message replication from the latest message position currently in the message queue.      
## Are there any limits to this proposal?      
    If users want to use this feature, they will deploy Message Connector Component. It will add a little complexity for operation and maintenance.     

## Changes
    If users want to use this feature, they will deploy Message Connector Component. 

## Architecture

We plan to design the function of Message Connector as a new component. 


### Enahncement 1: Sub-project: Implement full messages synchronous replication
We will implement the basic feature for this rip. In this phase, Users can use the Message Connector Component for message replication basicly.


### Enhancement 2: Sub-project: Monitering some parameters in the process of message replication   
This phase we will collect some parameters in the process of message replication, such as message tps, delay, latest synchronous time and so on.User can show this parameters in their own monitering system.


### Enhancement 3: Sub-project: Implement exactly once replication 
This phase we will implement message idempotency to meet the exactly once demand.