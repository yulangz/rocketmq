# Status
Current State: Proposed    
Authors: Kevin Wang     
Shepherds: Vongosling     
Mailing List Discussion: <apache mailing list archive>     
Pull Request: #PR_NUMBER      
Released: <released_version>     

# Background & Motivation
## What do we need to do
Apache RocketMQ is the top-level project of the Apache Software Foundation, which is used widely both in both China and the rest of the world  by more and more developers, it provides convenient and multi-language APIs. 
With the popular of Spring Cloud and Spring Boot among developers, it is required to wrap RocketMQ API with Spring style and work with Spring to allow developers easily integration with Spring cloud/boot with less code and less configuration.     
There has been provided a tentative project rocketmq-spring-boot-starter in RocketMQ community, it provides flexible annotations and POJOs to work with Spring, we intend to enhance the project with more features, documents and usage demos in the near future and contribute this to Spring community as a standard messaging microservice besides RabbitMQ and Kafka.      

# Goals
## What problem is this proposal designed to solve?
1. Merge latest features of RocketMQ into spring-rocketmq     
2. Attract more developers join the project     
3. Push the spring-rocketmq to Spring community     

## To what degree should we solve the problem?
We hope with the spring-rocketmq, the developers and cloud users can use the RocketMQ easy and smooth in the Spring Cloud world.

# Non-Goals
## What problem is this proposal NOT designed to solve?
In this phase, the project is designed to only support Spring Boot, not support the Spring Integration or Spring Cloud Stream.     
The project will only support in Java languate and Spring framework.     

## Are there any limits of this proposal?
Pushing the project to Spring needs approval of goalkeeper of Pivotal.     

# Changes
## Enahncement 1:  Sub-project: rocketmq-spring-test
This is an accessional project to generate an embedded RocketMQ Broker in a standalone jar, it is majorly used in spring-rocketmq test scenarios instead of using it in production environment. 
## Enahncement 2:  Support sending transactonal message
Distributed transactional message is an important new feature in RocketMQ 4.3.0 release, we need this function in spring-rocketmq producer API.
Currently the project has supported the following features:
RocketMQTemplate: It provides a "template" as a high-level abstraction for sending messages;

```
@RocktMQMessageListener: The annotation works with Message-driven POJOs and implements callback method to consume messages.

@RocketMQMessageListener(topic = "${spring.rocketmq.topic}", consumerGroup = "string_consumer")
public class StringConsumer implements RocketMQListener<String> {
   @Override
   public void onMessage(String message) {
       log.info("------- StringConsumer received: {}", message);
   }
}
```
The enhancement will add the below changes on producer side:
1. Add new annotation @RocketMQTransactionListener to define Executor and Checker implementations
```
@RocketMQTransactionListener(transName = TRANS_NAME)
class TransactionListenerImpl implements TransactionListener {
   @Override
   public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
       ..
   }
 
   @Override
   public LocalTransactionState checkLocalTransaction(MessageExt msg) {
      ...
   }
}
```
2. Add new methods to send transactional message in RocketMQTemplate class:
```
public TransactionSendResult sendMessageInTransaction(String txProducerName, org.apache.rocketmq.common.message.Message rocketMsg, Object arg) throws MQClientException {
   ...
}
 
public TransactionSendResult sendMessageInTransaction(String txProducerName, String destination, org.springframework.message.Message<?>msg, Object arg) throws MQClientException {
   ...
}
```
## Enhancement 3:  Enrich code comments and reformat     
In order to pull request (or become a new) Spring project, we’d like to refactor the source code to add more comments and format with Spring concepts.    
Reference the below section:      
None of these is essential for a pull request, but they will all help. They can also be added after the original pull request but before a merge.      
Use the Spring Framework code format conventions (import eclipse-code-formatter.xml from the root of the project if you are using Eclipse).     
Make sure all new .java files to have a simple Javadoc class comment with at least an @author tag identifying you, and preferably at least a paragraph on what the class is for.    
Add the ASF license header comment to all new .java files (copy from existing files in the project)
Add yourself as an @author to the .java files that you modify substantially (more than cosmetic changes).    
Add some Javadocs and, if you change the namespace, some XSD doc elements.   
A few unit tests would help a lot as well - someone has to do it.    
If no-one else is using your branch, please rebase it against the current master (or other target branch in the main project).     
It might be required to change the package name of each classes before doing the final merge.     
## Enhancement 4:  Demo and Tutorial docment
For users quickly start to use the spring-rocketmq, we will provide demo and usage document.    
The orginal demo project is in https://github.com/aqlu/rocketmq-demo.git. The enhancement will move it into the RocketMQ samples project and with completed documentation.   

