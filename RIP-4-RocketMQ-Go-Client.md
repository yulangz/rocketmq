# Status
Current State: implemented    
Authors: dinglei(ShannonDing)    
Shepherds: zhengdong    
Mailing List discussion: users@rocketmq.apache.org;dev@rocketmq.apache.org   
Pull Request:https://github.com/apache/rocketmq-client-python/pull/1    
Released: <4.3.2>    
# Background & Motivation
## What do we need to do
     Now, applications developed by go can not send or consume messages through RocketMQ. So we want to provide new client which can be used by  that applications.We will add a new module  called rocketmq-client-go.
Goals
## What problem is this proposal designed to solve?
       We plan to provide the same functions as the Go client. For example, producers can be sent in two ways, synchronous and asynchronous. For consumers, Go clients also offer two modes of push and pull.We are also concerned about availability, reliability and delay.
## To what degree should we solve the problem?
 Support all rocketmq features，such as broadcast/cluster model, concurrency/orderly publish/subscribe, timed/delay msg, consumer status query and so on.      
Support across platform,and all features are supported on both windows and linux system.     
Low latency,publish latency < 2ms, subscribe latency < 10ms      
Fault recovery capability.Based on nameServer snapshot and network disaster recovery strategy, no real-time impact on publish/subscribe when anyone of broker or nameSrv was broken.     
# Non-Goals
## What problem is this proposal NOT designed to solve?
 Rocketmq-client-go does not support other protocols except rocketmq  protocol, such as JMS and AMQP.
Are there any limits of this proposal?
Rocketmq-client-python needs to rely on boost python, rocketmq cpp client library(librocketmq.so) and so.
# Changes
We will provide a new module named rocketmq-client-go. Read below sections to get more details about the python client  for RocketMQ
# Architecture
     Rocketmq-client-go  is based on the encapsulation of the C interface , which is provided by librocketmq.so or rocketmq.a . so it is logically divided into two  layers: API layer encapsulated by cgo, and C library which is also divided into C API layer ,message layer, protocol layer and transport layer.


## Interface Design
### 1 Go Interface
```
func Version() (version string){
   return GetVersion()
}

type Message interface {
}
type MessageExt interface {
}
type Producer interface {
}
type PushConsumer interface {
}

func CreateMessage(topic string)(msg Message){
}
func DestroyMessage(msg Message){
}
func SetMessageKeys(msg Message,keys string)(int){
}
func SetMessageBody(msg Message,body string)(int){
}
func CreateProducer(groupId string)(producer Producer){
}
func DestroyProducer(producer Producer){
}
func StartProducer(producer Producer)(int){
}
func ShutdownProducer(producer Producer)(int){
}
func SetProducerNameServerAddress(producer Producer, nameServer string)(int){
}
func SetProducerSessionCredentials(producer Producer, accessKey string, secretKey string, channel string) (int) {
}
func SendMessageSync(producer Producer, msg Message)(sendResult SendResult){
}


type Callback func(msg MessageExt) ConsumeStatus

func CreatePushConsumer(groupId string) (consumer PushConsumer) {
}
func DestroyPushConsumer(consumer PushConsumer) {
}
func StartPushConsumer(consumer PushConsumer) int {
}
func ShutdownPushConsumer(consumer PushConsumer) int {
}
func SetPushConsumerGroupID(consumer PushConsumer, groupId string) (int) {
}
func SetPushConsumerNameServerAddress(consumer PushConsumer, name string) (int) {
}
func SetPushConsumerThreadCount(consumer PushConsumer, count int) (int) {
}
func SetPushConsumerMessageBatchMaxSize(consumer PushConsumer, size int) (int) {
}
func SetPushConsumerInstanceName(consumer PushConsumer, name string) (int) {
}
func SetPushConsumerSessionCredentials(consumer PushConsumer, accessKey string, secretKey string, channel string) (int) {
}

func Subscribe(consumer PushConsumer, topic string, expression string) (int) {
}

func RegisterMessageCallback(consumer PushConsumer, callback Callback) (int) {
}
````


###  2. Other Features
# Compatibility, Deprecation, and Migration Plan   
## Are backward and forward compatibility taken into consideration?
Yes，all  interfaces are backward compatible
## Are there deprecated APIs?
No, all APIs are new.
## How do we do migration?
All the interfaces are newly developed, and new access applications are directly used without migration problems. If your system is currently using other python clients, you need to modify the code according to the current specifications.     
# Implementation Outline
We will implement the proposed changes by 2 phases.     
API layer, message layer, protocol layer and transport layer    
Phase Development of basic future    
Implementation of sending message     
Implementation of Consuming message by push model     
Phase Development of other features     
Implementation of pull consume model,  orderly message and transaction message and so.    


