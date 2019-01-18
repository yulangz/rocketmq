# Status
Current State: Implementing    
Authors: HuzongTang     
Shepherds: Zhendong     
Mailing List Discussion: <apache mailing list archive>    
Pull Request: #PR_NUMBER    
Released: <released_version>  
 
# Background & Motivation
## What do we need to do
Message track trace refers to the consumption processing of a message which is sent from the producer instance has arrived the broker,and then has been consumed by the consumer instance.In the whole process, the information of time, location and other related services(including Client and Broker) is aggregated into the complete link information of message communication.      
In MQ system, the complete link of a message contains three roles: producer, broker server and consumer.In the process of message communicaton, each role adds relevant information to the track trace link.By aggregating these information,it can provide some powerful support to user.      
About the Apache RocketMQ project,I will add the feature of message track trace which can help users query every complete link data of a message exactly.This funcation is important to RocketMQ,especially in some  financial application fields.     
    
# Goals
##  What problem is this proposal designed to solve?
Reliability and Availability is the most two important characters for each MQ system.Although RocketMQ does very well in the two fields,it need some other methods to make sure the complete link of a message is no problem.By some ways,we should view the complete link of a messag and find the root cause of failure in process messaging quickly.        
So,if RocketMQ support the feature of Message Track Trace,we can find and analyse the root cause of failure in process messaging easily.And we can query many parameter values,such as sending cost time,consumption cost time,store time in broker and so on.The architecture of Message Track Trace will be introduced in below chapter.

# Non-Goals
## What problem is this proposal NOT designed to solve?
Every users who build application by using RocketMQ may lack of a efficient way to  find and analyse the root cause of failure in process messaging easily and quickly.The users may spend much more time and enery on operation of RocketMQ in production enviroment.           
## Are there any limits of this proposal?  
In order to reduce the impact of message entity content storage in RocketMQ，we redefine a special broker which stores every message track trace.So,if user don't want to use this funcation,he can close it in client by setting a flag.       
# Changes
We need add some codes in client and broker component which include collecting producer and consumer instances' infomation and redefine a special broker node.Read below sections to get more details about the Message Track Trace for RocketMQ.    

# Architecture
We plan to design the function of message track trace including its store part and client instance collecting. The total architecture has two part below:     
## Client instance collects the infomation of message track trace
    
**The design of architecture above as follows:       **
    * "producer and consumer multi-thread module, and Blocking Queue"：here, in client,as a producer model,we can collect the infomation(such as,sending and consuming message cost time,broker store time,broker stored ip address and so on) and put this information into the Blocking Queue.And as s consumer model,use a thread called "MQ-AsyncArrayDispatcher" to take the message track trace infomation from Blocking Queue.Then this asynchronous thread( called "MQ-AsyncArrayDispatcher") pack the message track trace element as AsyncAppenderRequest task and submit to the thead pool.      
    Last,the main execution process of  AsyncAppenderRequest task is sending the message track trace infomation which is collected above from client side to a special broker node which is redefined in below chapter.       
    * define a new pair hook which is implementation of the“SendMessageHook/ConsumeMessageHook”,from this,we can collect message track trace data  before and after publishing and subscribing messages.      
   
## Redefine a special broker node to store message track trace data
    
    As shown in the above picture,we can define a specail broker service node which can store the message track trace data in a common RocketMQ cluster.Here,we can add a flag(such as autoTraceBrokerEnable) in broker.properties file and use this variable to define Whether this broker is a specail node for storing message track trace data.      
    * .autoTraceBrokerEnable is false.This indicates this broker is an ordinary node,then "Trace_Topic" will not be created in this node.And it will still follow the original processing flow without any change.
    * .autoTraceBrokerEnable is true.This indicates broker is an special node,which stores the message track trace data specailly.And The "Trace_Topic" is automatically created during the Broker's startup phase,this node automatically registers its owned set of topics in memory to nameserver(including Trace_Topic).Thus,in a RocketMQ cluster,there is only a specail broker node which stores message track trace datas.And clients(including publishing and subscribing messages) knows which broker node to send message trace data after they try fetch topic routing info from nameserver.

## How to solve the query message track trace data by original topic 
  For example,Topic which saves message track trace data is called "RMQ_SYS_TRACE_DATA_XXX" is instead of Topic of original data.But，message query(By messageId + topic,topic+ key or topic) which still uses original data on RocketMQ console side can't query the expected result for us.Here, we can fill in the keyset field of the message track trace data by using msgId (not offset MsgId) or Key of the original message content when send the message track trace data collected by the client, so the IndexFile index of the Broker end can be built according to the msgId or Key of the original message.          
    And at the broker side,we can invokes the queryMessage() method of the storage part through the QueryMessageProcessor Business processor.         

## Finishing querying messages track trace datas in console
Fisrtly, the console project is based on spring boot technology,so we can finish querying messages track track datas in this project.The basic design is using thread pool to send a RPC request to the redefined broker node to get the message track trace data.     
  
# Interface Design/Change
    Here I just list part interface desgin and change in Client Side.And the chang parts of Broker and Console side leave out.          
   * The data transfering asynchronously interface is as followers: 
```
    public interface AsyncDispatcher {

	void start() throws MQClientException;

	boolean append(Object ctx);

	void flush() throws IOException;

	void shutdown();	
    }
```
* Defining the two Hooks which is implemented of ConsumeMessageHook and SendMessageHook are as followers:
     a.ClientSendMessageTraceHookImpl        
     b.ClientConsumeMessageTraceHookImpl       

* We define and write some codes for common data model which are as followers:
     a.TraceBean      
     b.TraceConstants     
     c.TraceContext     
     d.TraceDataEncoder    
     e.TraceDispatcherType   
     f.TraceTransferBean   
     g.TuxeTraceType    
# Compatibility, Deprecation, and Migration Plan
##     （1）Are backward and forward compatibility taken into consideration?
     No issues;
##     （2）Are there deprecated APIs?
     No issues;
##  （3）How do we do migration?
     No issues;

# Implementation Outline
     We will implement the proposed changes by 1 phase. All of implementation will be finished in 1 phase.