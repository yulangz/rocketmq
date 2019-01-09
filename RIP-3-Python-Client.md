
# Status
Current State: implemented      
Authors: qingfeng(wlliqipeng),dinglei    
Shepherds: zhengdong   
Mailing List discussion: users@rocketmq.apache.org;dev@rocketmq.apache.org     
Pull Request:https://github.com/apache/rocketmq-client-python/pull/1    
Released: <4.3.2>  
# Background & Motivation
## What do we need to do
     Now, applications developed by python can not send or consume messages through RocketMQ. So we want to provide new client which can be used by  that applications.We will add a new module  called rocketmq-client-python.
# Goals
## What problem is this proposal designed to solve?
       We plan to provide the same functions as the Python client. For example, producers can be sent in two ways, synchronous and asynchronous. For consumers, Python clients also offer two modes of push and pull.We are also concerned about availability, reliability and delay.
## To what degree should we solve the problem?
 Support all rocketmq features，such as broadcast/cluster model, concurrency/orderly publish/subscribe, timed/delay msg, consumer status query and so on.   
Support across platform,and all features are supported on both windows and linux system.   
Low latency,publish latency < 2ms, subscribe latency < 10ms .  
Fault recovery capability.Based on nameServer snapshot and network disaster recovery strategy, no real-time impact on publish/subscribe when anyone of broker or nameSrv was broken.   
#  Non-Goals
## What problem is this proposal NOT designed to solve?
 Rocketmq-client-Python does not support other protocols except rocketmq  protocol, such as JMS and AMQP.
Are there any limits of this proposal?
Rocketmq-client-python needs to rely on boost python, rocketmq cpp client library(librocketmq.so) and so.
# Changes
We will provide a new module named rocketmq-client-python. Read below sections to get more details about the python client  for RocketMQ
# Architecture
     Rocketmq-client-python  is based on the encapsulation of the C interface , which is provided by librocketmq.so or rocketmq.a . so it is logically divided into two  layers: API layer encapsulated by boost python, and C library which is alse divided into C API layer ,message layer, protocol layer and transport layer.


Interface Design
## 1 Python Interface
```
BOOST_PYTHON_MODULE (librocketmqclientpython) {
/*
   class_<CMessage>("CMessage");
   class_<CMessageExt>("CMessageExt");
   class_<CProducer>("CProducer");
   class_<CPushConsumer>("CPushConsumer");
*/
   enum_<CStatus>("CStatus")
           .value("OK", OK)
           .value("NULL_POINTER", NULL_POINTER);

   enum_<CSendStatus>("CSendStatus")
           .value("E_SEND_OK", E_SEND_OK)
           .value("E_SEND_FLUSH_DISK_TIMEOUT", E_SEND_FLUSH_DISK_TIMEOUT)
           .value("E_SEND_FLUSH_SLAVE_TIMEOUT", E_SEND_FLUSH_SLAVE_TIMEOUT)
           .value("E_SEND_SLAVE_NOT_AVAILABLE", E_SEND_SLAVE_NOT_AVAILABLE);

   enum_<CConsumeStatus>("CConsumeStatus")
           .value("E_CONSUME_SUCCESS", E_CONSUME_SUCCESS)
           .value("E_RECONSUME_LATER", E_RECONSUME_LATER);

   class_<PySendResult>("SendResult")
           .def_readonly("offset", &PySendResult::offset, "offset")
                   //.def_readonly("msgId", &PySendResult::msgId, "msgId")
           .def_readonly("sendStatus", &PySendResult::sendStatus, "sendStatus")
           .def("GetMsgId", &PySendResult::GetMsgId);
   class_<PyMessageExt>("CMessageExt");

   //For Message
   def("CreateMessage", PyCreateMessage, return_value_policy<return_opaque_pointer>());
   def("DestroyMessage", PyDestroyMessage);
   def("SetMessageTopic", PySetMessageTopic);
   def("SetMessageTags", PySetMessageTags);
   def("SetMessageKeys", PySetMessageKeys);
   def("SetMessageBody", PySetMessageBody);
   def("SetByteMessageBody", PySetByteMessageBody);
   def("SetMessageProperty", PySetMessageProperty);

   //For MessageExt
   def("GetMessageTopic", PyGetMessageTopic);
   def("GetMessageTags", PyGetMessageTags);
   def("GetMessageKeys", PyGetMessageKeys);
   def("GetMessageBody", PyGetMessageBody);
   def("GetMessageProperty", PyGetMessageProperty);
   def("GetMessageId", PyGetMessageId);

   //For producer
   def("CreateProducer", PyCreateProducer, return_value_policy<return_opaque_pointer>());
   def("DestroyProducer", PyDestroyProducer);
   def("StartProducer", PyStartProducer);
   def("ShutdownProducer", PyShutdownProducer);
   def("SetProducerNameServerAddress", PySetProducerNameServerAddress);
   def("SendMessageSync", PySendMessageSync);

   //For Consumer
   def("CreatePushConsumer", PyCreatePushConsumer, return_value_policy<return_opaque_pointer>());
   def("DestroyPushConsumer", PyDestroyPushConsumer);
   def("StartPushConsumer", PyStartPushConsumer);
   def("ShutdownPushConsumer", PyShutdownPushConsumer);
   def("SetPushConsumerNameServerAddress", PySetPushConsumerNameServerAddress);
   def("SetPushConsumerThreadCount", PySetPushConsumerThreadCount);
   def("SetPushConsumerMessageBatchMaxSize", PySetPushConsumerMessageBatchMaxSize);
   def("Subscribe", PySubscribe);
   def("RegisterMessageCallback", PyRegisterMessageCallback);

   //For Version
   def("GetVersion", PyGetVersion);
}
```


 # Compatibility, Deprecation, and Migration Plan
Are backward and forward compatibility taken into consideration?
Yes，all  interfaces are backward compatible
Are there deprecated APIs?
           No, all APIs are new.
## How do we do migration?
All the interfaces are newly developed, and new access applications are directly used without migration problems. If your system is currently using other python clients, you need to modify the code according to the current specifications.
## Implementation Outline
We will implement the proposed changes by 2 phases.     
API layer, message layer, protocol layer and transport layer    
Phase Development of basic future    
Implementation of sending message      
Implementation of Consuming message by push model    
Phase Development of other futures    
Implementation of orderly message and transaction message and so.    


