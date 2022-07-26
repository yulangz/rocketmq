# Overview

Apache RocketMQ is a distributed messaging and streaming platform with low latency, high performance and reliability, trillion-level capacity, and flexible scalability. It consists of four parts: name servers, brokers, producers, and consumers. Each of them can be horizontally scaled without a Single Point of Failure.



**NameServer Cluster**

Name Servers provide lightweight service discovery and routing. Each Name Server node maintains a complete routing table, supports registration and queries of service entries and makes clusters scale instantly in the end.

**Broker Cluster**

Brokers are in charge of message storage and retrieval. Data on brokers are organized logically by TOPIC and QUEUE. They support the Push and Pull model, contains fault tolerance mechanism (2 copies or 3 copies), and provides strong padding of peaks and capacity of accumulating hundreds of billion messages in their original time order. In addition, Brokers provide disaster recovery, rich metrics statistics, and alert mechanisms, all of which are lacking in traditional messaging systems.

**Producer Cluster**

Producers support distributed deployment. Distributed Producers send messages to the Broker cluster through multiple load balancing modes. The sending processes support fast failure and have low latency.

**Consumer Cluster**

Consumers support distributed deployment in the Push and Pull model as well. It also supports cluster consumption and message broadcasting. It provides a real-time message subscription mechanism and can meet most consumer requirements. 
RocketMQ’s website provides a simple quick-start guide to interested users.

# NameServer

NameServer is a fully functional server, which mainly includes two features:

* Broker Management, **NameServer** accepts the register from Broker cluster and provides heartbeat mechanism to check whether a broker is alive.
* Routing Management, each NameServer will hold whole routing info about the broker cluster and the **queue** info for clients query.

As we know, RocketMQ clients(Producer/Consumer) will query the queue routing info from NameServer, but how do clients find NameServer address?

There are four methods to feed NameServer address list to clients:

* Programmatic Way, like `producer.setNamesrvAddr("ip:port")`.
* Java Options, use `rocketmq.namesrv.addr`.
* Environment Variable, use `NAMESRV_ADDR`.
* HTTP Endpoint.


# Broker Server

Broker server is responsible for message store and delivery, message query, HA guarantee, and so on.

As shown in the image below, Broker server has several important sub-modules:

* Remoting Module, the entry of broker, handles the requests from clients.
* Client Manager, manages the clients (Producer/Consumer) and maintains topic subscription of the consumer.
* Store Service, provides simple APIs to store or query message in the physical disk.
* HA Service, provides data sync feature between the master broker and slave broker.
* Index Service, builds an index for messages by specified key and provides quick message query.
