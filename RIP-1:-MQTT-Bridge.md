# Status   
Current State: Proposed   
Authors: Zongyang   
Shepherds: Zhendong   
Mailing List Discussion: <apache mailing list archive>    
Pull Request: #PR_NUMBER    
Released: <released_version>    
# Background & Motivation   
## What do we need to do   
We are providing a new solution about for reliable and real-time message service for IoT devices and users. We are addressing the issues about reliability, latency and availability.

About the Apache RocketMQ project, we will build a new module named mqtt-bridge which enables the IoT devices leveraging the Apache RocketMQ abiliities and features through MQTT.

We are solving the connection  problems in the IoT field and we will support multiple IoT protocols including CoAP, LoRa and NB-IoT in the long term.
## Goals
### What problem is this proposal designed to solve?
Since the functional capabilities are described by the MQTT protocol specification(see here for more details about the specification), we are addressing issues about
* Reliability
* Latency
* Availability
In more details, the reliability comes first, which means our system should make ensure a message won’t be lost once the producer publish it to the bridge successfully. Then, we should reduce the latency of the message delivery. Finally, we should make our system highly available, that is to say, if any single node of the system broke down wouldn’t cause the system unavailable.
About architecture, the MessageStore, SessionStore modules are all plugable.
### To what degree should we solve the problem?
We support the MQTT protocol first, and other protocols later. The IoT applications which want to apply our solution should work based on the message. And their applications should implement supported protocols.
## Non-Goals
### What problem is this proposal NOT designed to solve?
* IoT applications implement unsupported protocol.
* IoT applications work by model except message.
### Are there any limits of this proposal?
The proposed solution requires a independently deployed process rather than threads in broker process. The IoT Bridge may require other dependencies like Redis which depends on the specific implementation.

# Changes
We will provide a new module named iot-bridge whose artifact id is rocketmq-iot-bridge. Read below sections to get more details about the IoT Bridge for RocketMQ.
# Architecture
We plan to design the bridge to support pluggable Message Storage, Subscription Storage and Upstream analysistic systems. So the Apache RocketMQ may be one option of Message Store and Subscription Store which depends only on the implementation.
 
As shown in above diagram, the Client communicate with the Bridge to send or receive messages to implement their business.
In more details, the responsibilities of the bridge internal modules are:
* The Connection Manager manages the connection between the Client and the Bridge.
* The Protocol Converter converts the MQTT message to the RocketMQ message, and vice versa.
* The Downstream module
* handles the SUBSCRIBE requests from the Clients
* handle the PUBLISH messages from the Clients and persist them to Message Store if needed.
* send messages to the Clients which subscribes the topic of the message
* forward a message to other Bridge if all the Clients curious about the message connected to this Bridge.
* The Upstream module receives message published by the Clients and upload the messages to external systems like Refid, Kafka, Spark, etc.
* The Message Store persists all the messages produced by all the Clients in our system and providing interfaces for the Bridge to 
* query specific messages
* mark a message to be acknowledged by a Client
* put a message with the clients which are curious about it.
* The Subscription Store persists global information about the subscription relationship which is a mapping from topic to the list of Clients and its qos level

A Client, whatever it is a Producer or Consumer, should establish a connection with the any Bridge instance.

When a Producer sends a message to a Bridge instance, the message should be processed by the Message Receiver first which may include authorization, message validation and persistent to the Message Store. Before the message persisted to the Message Store, the receiver should push the message to the Message Cache.

When a Consumer subscribes to a series of topics by it sending a SUBSCRIBE message to the Bridge instance to which it connects. The Bridge should maintain all the subscription relationship which is a mapping from topic to client and fetch message from Message Cache or Message Store for the Consumer.

When the Bridge fetch a message, it should dispatch the message to the Clients which subscribes to the topic of the message and wait the acknowledgement from the Clients. When the bridge receives the ACK from a client, the Bridge should mark the client already acknowledged the message in the Message Store.

When a Consumer sends a UNSUBSCRIBE message to unsubscribe from the topics, the Bridge should update the topic-client mapping and destroy the subscription relationship of the Client.
# Interface
The data used by the interfaces are 
public class Client {
    private String id;
    private transient ChannelHandlerContext ctx;
    private transient String token;
    private long maxInFlightMessageNum;
    private long inFlightMessageNum;
    private List<Subscription> subscriptions;
}

public class Subscription {
    private String topic;
    private Client client;
    private int qos;
}


public class Message {
private String id;
    private Map<String, String> headers;
    private Type type;
    private Object payload;
    private Client client;
    private String topic;

}

public class Bridge {
    private String id;
    private Channel channel;
}

The interfaces of above modules are listed.
# Subscription Store
public interface SubscriptionStore {
    /**
     * get the id list of the subscriptions which subscribe to the topic
     * @param topic
     * @return id list of the subscriptions which subscribe to the topic
     */
    List<Subscription> get(String topic);

    /**
     * check if a topic exists or not
     * @param topic
     * @return
     */
    boolean hasTopic(String topic);

    /**
     * add a new topic to existing subscriptions
     * @param topic
     */
    void addTopic(String topic);

    /**
     * append the client to the topic
     * @param topic the topic to which the client subscribes
     * @param subscription the subscription of the client
     * @return the subscription list of the client
     */
    void append(String topic, Subscription subscription);

    /**
     * remove the subscription of a client from the topic
     */
    void remove(String topic, Client client);

    /**
     * get the topics which match the filter
     * @param filter the topic filter which contains wildcards ('+' and '#')
     * @return matched topics
     */
    List<String> getTopics(String filter);

    /**
     * start the Session Store
     */
    void start();
    /**
     * shutdown the Session Store
     */
    void shutdown();
}

# Message Store
public interface MessageStore {

    /**
     * get message from the topic by <code>id</code>, the method will return
     * <b>null</b> if the topic doesn't exist or the message doesn't exist
     * @param id identifier of the message
     * @return
     */
    Message get(String id);

    /**
     * put message to MessageStore
     * @param message the message to put to the Message Store
     * @return identidier of saved message
     */
    String put(Message message);

    /**
     * prepare to push message to the clients by marking the receiving id list of the clients
     * @param messageId the identifier of the message to be prepared
     * @param clientIds the identifier list of the clients which will receive the message
     * @param qos the QoS level of the message to be prepared
     */
    void prepare(String messageId, List<String> clientIds, int qos);

    /**
     * acknowledge the message for the specific client
     * @param message the message to be acknowledged
     * @param client the client which acknowledge the message
     */
    void ack(Message message, Client client);

    /**
     * acknowledge the message for the specific list of clients
     * @param message the message to be acknowledged
     * @param clients the list of clients which acknowledge the message
     */
    void ack(Message message, List<Client> clients);

    /**
     * expire the message with specific identifier
     * @param id identifier of the message to be expired
     */
    void expire(String id);

    /**
     * start the MessageStore
     */
    void start();

    /**
     * get offline messages of the client
     * @param client
     * @return the offline messages of the client
     */
    List<Message> getOfflineMessages(Client client);

    /**
     * shutdown the MessageStore
     */
    void shutdown();
}

# MessageHandler
public interface MessageHandler {
    /**
     * handle message from client
     *
     * @param message
     * @return whether the message is handled successfully
     */
    boolean handleMessage(Message message);
}

# Client Manager
The Client Manager create or retrieve Client when a Channel is active which means a packet from client is arrived.
public interface ClientManager {

    /**
     * get the Client by its channel
     * @param channel
     * @return
     */
    Client get(Channel channel);

    /**
     * put the Client by its channel
     * @param channel the channel by which the Client connects to the server
     * @param client
     */
    void put(Channel channel, Client client);

    /**
     * remove the Client by its channel usually the channel is disconnected
     * @param channel
     */
    void remove(Channel channel);
}

# Compatibility, Deprecation, and Migration Plan
Are backward and forward compatibility taken into consideration?
All the Clients want to leverage the solution should apply the MQTT protocol version 3.1.1.
Are there deprecated APIs?
No, all APIs are new.
How do we do migration?
The Client applies the MQTT protocol doesn’t need to modify any code and just configure to use the endpoint of the Bridge cluster.

# Proposed Implementation Outline
We will implement the proposed changes by 3 phases. 
## Phase 1 Support QoS 0 and Clean Session
Implement interfaces of Message Sender, Message Receiver and Subscription Manager
Don’t take the client re-connecting into consideration with manage session (Clean Session)
Don’t take the offline message persistence and delivery into consideration
## Phase 2 Support QoS 1 and Sticky Session
Support Sticky Session
Support QoS 1 by the acknowledgement mechanism
Solve offline messages
## Phase 3 Support Message Forwarding
Forward messages among Bridges
# Rejected Alternatives 
Extends the broker to support multiple Protocols and adaptes model of other protocols to RocketMQ model.
How does alternatives solve the issue you proposed?       
All the Clients connection to the RocketMQ broker, the pressure of the broker is too heavy to undertake. And millions of requests will break the broker down.       
## Pros and Cons of alternatives  
### Pros    
The Bridge undertakes the pressure of the broker and manage the model mapping including message mapping and topic mapping.    
### Cons  
The latency for message delivery is increased because we add one more network connection.      
Why should we reject above alternatives        
We should handles the connections and requests of the Clients in front of the broker by Bridge, which also gets rid of causing side effects of co-existing of multiple protocols.         