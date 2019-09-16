# Status

Current State: develop    
Authors: huzongtang,feichao,liujiangang,huangdian    
Shepherds: liqipeng     
Mailing List discussion: users@rocketmq.apache.org;dev@rocketmq.apache.org     
Released:       

# Background & Motivation

## What do we need to do

For RocketMQ project, there are some issues in the console project, such as unfriendly UI pages and exception codes, some bugs and poor scalability and maintainability. So, we're going to develop a new industrial level console project for RocketMQ which has a good UI page and higher scalability.

# Goals
## What problem is this proposal designed to solve?

We plan to design front and back end module separation architecture for RocketMQ console project. So, users can customize their UI Pages and css by themselves. And they also can expand the open source console project for their business processes very easily.At the same time, we will add some new features and optimize codes for RocketMQ console project.  

## To what degree should we solve the problem?
(1)More user-friendly UI;    
(2)Add some new features for console project;    
(3)Better scalability     


# Non-Goals
## What problem is this proposal NOT designed to solve?

In this phase, the project will not support api-gateway.If users need to authenticate bettween front and backend module,they will provide the api-gateway project by themselves.

## Are there any limits to this proposal?

Front and back end module separation architecture for console will bring a little extra work for deploying console to user.User need to deploy front module to a static server, such as nginx.And then,they startup backend project which is based on springboot as before.

# Changes

## Enahncement 1: Sub-project: front-end module
The fron-end module will provide much more friendly UI pages for users.The UI pages can reference prometheus visualization UI pages.


## Enhancement 2: Sub-project: backend module
The backend module will provide encapsulated standard restful api interface for users.Firstly, we will optimize original api in the original console project.Secondly, we will add some new features api, such as DLQ information export, msg trace data export, more friendly exception handling and so on. 

## new features:
(1)users can login on the console.    
(2)add DLQ and retry MessageQueue detail.    
(3)more friendly clustering showing.    
(4)export messages of DLQ.    
(5)export data of msg trace.        
(6)support acl in console.     

