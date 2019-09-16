RIP11- Evolution of The Next Decade Architecture for RocketMQ
# Status
Current State: Proposed    
Authors: Heng Du     
Shepherds: vongosling,  dongeforever, duhengforever   
Mailing List Discussion: users@rocketmq.apache.org;dev@rocketmq.apache.org    
Pull Request:  
Released:    
   
# Background & Motivation
## What do we need to do
   With the emergence of IoT, AI, Blockchain and other scenarios around the world, traditional messaging middleware faces huge challenges, so in order to better serve in the next decade and make Apache RocketMQ as the data infrastructure in the cloud computing era.  We should think about the architecture evolution for RocketMQ.
    After countless investigations, We think that Apache RocketMQ should aim to become a unified messaging engine, lightweight data processing platform.  Besides, Apache RocketMQ should have the IOT native ability. and Financial level stability.
    
     
# Goals
##  A unified messaging engine, lightweight data processing platform
      Separation of storage computing and pluggable architecture.
      Improve computing power and response speed without concerned about the machine cost, operation, and - - maintenance cost brought by storage. 
      Powerful storage and indexing capabilities, support for multiple forms of queries and lightweight computing.
      Lightweight streaming processing ability.
      Support for OpenMessaging standard.

## IOT native and Finacial level stability
      Integrating TCP-based MQTT, UDP-based CoAP or other IOT protocol.

## New hardware acceleration
      In the modern era, the emergence of much new hardware has brought a lot of different thinking and design patterns to the development of software, and this is what we are pursuing.

# Non-Goals
## More usage scenarios cover
In the next decade, we want to make Apache RocketMQ be a data infrastructure, so RocketMQ should cover more usage scenarios to meet different types of needs. But this is obviously a huge challenge


# Changes
 Great requirements for abstract capabilities brought about by the new architecture.     
 This is a huge project, so the cooperation with the community will bring even greater challenges.    
 Forward compatible ability    
 Stability testing and improvement     


