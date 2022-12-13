## Status

-  Current State: Proposed

- Authors: [Ni Ze](https://github.com/ni-ze)

- Shepherds: [ShannonDing](https://github.com/ShannonDing) [RongtongJin](https://github.com/RongtongJin)

- Mailing List Discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)

-  Pull Request: #PR_NUMBERe

- Released: <released_version>

## Background & Motivation

### What do we need to do

- Will we add a new module? Yes.

- Will we add new APIs? Yes.

- Will we add a new feature? Yes.

### Why should we do that

- Are there any problems with our current project?

There are several problems in current RocketMQ Streams:

1. ConfigurableComponent, which is a store and query component for streams task, appears to be a redundant component. Topology does not need to serialize and deserialize with it, just run the topology is better. ConfigurableComponent is more suitable for rsqldb to store streams task than RocketMQ Streams；This component need to removed in RocketMQ Streams.

2. Restricted data format for source topic.

In current RocketMQ Streams, custom serialization/deserialization is not support when consume/produce data from/to RocketMQ. Json is the default serialization and deserialization format， it is a heavy restrictions for user.

3. Not support for generics

 

- What can we benefit from proposed changes?

1. remove ConfigurableComponent for a more readable code.

2. custom serialization/deserialization for source topic and sink topic

3. support generics 

## Goals

What problem is this proposal designed to solve?

1. remove ConfigurableComponent

2. support custom serialization/deserialization

3. support generics

## Non-Goals

- What problem is this proposal NOT designed to solve?

## Changes

It is a big update for RocketQM Streams, almost rewote all module, hope this RIP can resolves the remaining problem of current version.

### Architecture
![image](https://user-images.githubusercontent.com/31175234/207049354-435e383c-3497-445a-b4e4-ab78eb5014c9.png)

#### typical usage
```java
public static void main(String[] args) {
    //wordCount为流计算任务唯一jobId
    StreamBuilder builder = new StreamBuilder("wordCount");
    //自定义反序列化
    builder.source("sourceTopic", new KeyValueDeserializer<Void/*key type*/, String/*value type*/>() {
          @Override
          public Pair<Void, String> deserialize(byte[] source) throws Throwable {
            String value = new String(source, StandardCharsets.UTF_8);
            return new Pair<>(null, value);
          }
       })
        .flatMapValues((ValueMapperAction<String, List<String>>) value -> {
          String[] splits = value.toLowerCase().split("\\W+");
          return Arrays.asList(splits);
        })
        .keyBy(value -> value)
        .count()
        .toRStream()
        .sink("sinkTopicName", new KeyValueSerializer<String/*key type*/, Integer/*value type*/>() {

         @Override
         public byte[] serialize(String key, Integer data) throws Throwable {

            //自定义序列化
           return new byte[0];
         }
       });

    TopologyBuilder topologyBuilder = builder.build();

    Properties properties = new Properties();

    //一个流处理任务只能从一个集群中读取数据

    properties.put(MixAll.NAMESRV_ADDR_PROPERTY, "127.0.0.1:9876");

    RocketMQStream rocketMQStream = new RocketMQStream(topologyBuilder, properties);

    rocketMQStream.start();
  }
```
#### domain model
![image](https://user-images.githubusercontent.com/31175234/207049690-fe48d502-73fa-46a6-ae83-6720e054cfed.png)

#### build process

When the user writes the above concatenation expression, the first construction occurs, that is, the addition of logical nodes. The preceding and following operators have a parent-child relationship. After construction, a logical node is formed, and multiple logical nodes form a linked list.

After the logical construction is completed, call the StreamBuilder#build() method for the second construction, and add multiple real nodes that may be included in the logical node to the topology to form a processing topology map.

After two builds, the processing topology is complete. However, in order to distinguish between different topics and use different topology processing, it is necessary to create real data processing nodes in the rebalancing stage before the data comes. This is the third construction.

This section discusses the first two builds, and the third build is discussed below in the data processing procedure.

##### Logical node build (first build)

The logical node itself does not include actual operations, but actual nodes can be constructed from logical nodes. A logical node may contain one actual node or multiple actual nodes. For example, the count logical operator not only includes the actual operation of accumulation, but also requires The data for the same key is routed to the same computing instance, so two actual nodes, sink and source, also need to be included, but these will only be reflected when the actual node is constructed, and will not be reflected in the stage of adding logical nodes.
 
![image](https://user-images.githubusercontent.com/31175234/207049758-6d1ac15f-1ba8-4a7b-bc0c-068be12a4935.png)
![img](./assets/RStream-pipeline.png)

**Shuffle node**

Count is a shuffle node, before count the number, data must be shuffle into rocketmq topic for that data with same key will be consumed by the same process.

1. Send the data back to RocketMQ, the topic name is jobId-ROCKETMQ-COUNT-${operator number}-shuffleTopic, and the number of partitions is set to 8 by default. The same key is sent to the same queue;

2. Subscription consumption shuffle topic;

3. Count the number of messages;

**JOIN node**

In addition to the special construction of the shuffle logical operator, the implementation of the Join operator is also special. Because the construction of all the above-mentioned operators in this article can be completed in one pipeline, but Join is an operation on two streams, so Join will involve two pipeline instances, the left stream and the right stream.

Design goals of the JOIN operator：

The same topology instance can process INNER JOIN and LEFT JOIN data at the same time;

For Inner join, key is equal to trigger;

For Left join, trigger the data in the left stream and the right stream whose key is equal to the left stream.

Inner Join:
| left  stream  |   right  stream  | result|
|  ---------- | ---------  | -------------|
| <k1:A>     |          |        |        
| <k2:B>     | <k2:b> | <k2:ValueJoinAction(K1,B,b)>|
|            |  <k3:c>|             |

Left Join:
| left  stream  |   right  stream  | result|
|  ---------- | ---------  | -------------|
| <k1:A>     |          |     <k1:ValueJoinAction(A,null)>   |        
| <k2:B>     | <k2:b> | <k2:ValueJoinAction(B,b)>|
|            |  <k3:c>|             |

JOIN operator logical topology:

![image](https://user-images.githubusercontent.com/31175234/207051191-8bb5321d-ee2c-4cee-8c33-e5258b2bdbdc.png)


##### Topology map construction (second construction)

After building the logical node, start traversing from the ROOT node, call the GraphNode logical node addRealNode method, and build the real node construction factory class.
![image](https://user-images.githubusercontent.com/31175234/207051259-8efb6e37-ac40-4f05-8b4d-1448fbf52e38.png)


The construction of the topology map just creates the RealProcessorFactory class and saves it. There is no topological graph that actually handles the data. Calling the RealProcessorFactory#build method to create real data processing nodes occurs during the rebalancing phase.

```java
public interface RealProcessorFactory<T> {
    String getName();

    //Processor为处理数据真实节点
    Processor<T> build();
}
```

#### Unify source topic and shuffle topic

Because there are operators such as group aggregation, use shuffle to route data with the same key to the same data processing instance. Therefore, a stream processing process can be regarded as two stages, one is the stage before shuffle, and the other is the stage after shuffle. They have different data sources, different data writing destinations, and different data processing topologies, but their essence is to read data from one topic, and after processing, write the result to another topic. So there is an opportunity to unify the two. From the perspective of data consistency, after unifying the two stages into one process, we only need to pay attention to the data consistency of one process, that is, we only need to concern the consistency of the process from data acquisition to data writing, and this process is consistent When the nature is realized, the overall consistency of stream processing is naturally realized.


![image](https://user-images.githubusercontent.com/31175234/207051386-7b1bb48a-ad92-46eb-82fe-2d1e65834db1.png)

In the schematic diagram, one litePullConsumer instance is used to subscribe to two topics, and the topology is selected according to the topic.

#### data processing

![image](https://user-images.githubusercontent.com/31175234/207051456-0814bf06-6d10-4284-9bcb-95476144d77d.png)


Because during the rebalancing process, the scheduling unit is Queue, a Queue is no longer consumed by a certain instance, and other surviving flow processing instances will continue to consume this Queue. Naturally, the state generated during the Queue consumption process should also be divided according to the Queue. In this way, the state can migrate with the migration of the source data Queue.

##### rebalance

###### **Build an operator instance (the third build)**

When the consumer has a Queue change, such as adding a new consumption Queue, the attributes of the Queue: brokerName, topic, and QueueId will be used to form a unique key, and a set of Processors will be built based on the construction results. This set of processors includes all the operators needed to process the data.

```java
public <T> Processor<T> build(String topicName) {
        SourceFactory<T> sourceFactory = (SourceFactory<T>) topic2SourceNodeFactory.get(topicName);
        Processor<T> sourceProcessor = sourceFactory.build();

        String sourceName = sourceFactory.getName();

        //集合中的顺序就是算子的父子顺序，前面的是后面的父节点
        List<String> groupNames = source2Group.get(sourceName);

        Processor<T> parent = sourceProcessor;
        for (String child : groupNames) {
            RealProcessorFactory<T> childProcessorFactory = (RealProcessorFactory<T>) realNodeFactory.get(child);

            //实际数据处理节点
            Processor<T> childProcessor = childProcessorFactory.build();

            //添加下一处理节点
            parent.addChild(childProcessor);
            parent = childProcessor;
        }

        return sourceProcessor;
    }
```

The next processing node of this node will be added during the build process. Returns a processor instance, which contains sub-processors.

```java
public interface Processor<T> extends AutoCloseable {
    void addChild(Processor<T> processor);

    //数据处理前调用，准备好数据处理上下文
    void preProcess(StreamContext<T> context) throws Throwable;

    //执行数据处理
    void process(T data) throws Throwable;
}
```
###### **Load the state**

Before performing data processing, it is also necessary to load the state of the stateful operator to the stream processing instance, so as to ensure that the stateful data gets the correct result.

Use the RocksDB as a locally storage, and the RocketMQ as a persistent storage. 

Because the data in the same queue will be processed by the same topology instance, the state generated by the same queue will be stored in the same state topic queue. When the state is restored, the queue is consumed from the beginning to the latest position, and the data with the same key takes the one with the largest queueOffset as the latest value. This process is called replay.

Save the latest obtained key-value value to the local storage RocksDB.

The entire state recovery process is an asynchronous process. When a stateful operator execute, check whether the corresponding state is restored. If there is no recovery, It need to wait.

The state storage implementation below will also discuss in more detail from the write-pull model.

##### data processing

###### **Get the data processor**

In the third construction process, the processor for data processing has been constructed. The mapping between queue and proessor is formed:

```java
//key= brokerName + topice + QueueId,即一个queue使用同一个processor
ConcurrentHashMap<String, Processor<?>> mq2Processor = new ConcurrentHashMap<>();
```
Use the queue to get the processor.

###### **processing data**

For the source node, need to define the watermark, obtain data time, and serialize the data

Data model passed between operators：

```java
public class Data<K, V> {
    private Properties header;
    private K key;
    private V value;
    private long timestamp;
}
```
The data processing logic is as follows：
```java
public void source(MessageExt messageExt) {
    //首个处理器
	SourceSupplier.SourceProcessor<K, V> processor = ...;
	//上下文
    StreamContextImpl<V> context = ...;

	//准备首个处理器的子节点，放入下文的childList
	processor.preProcess(context);

    //将messageExt转变为data
    Data<K, V> data = ;

    
    context.forward(data);
}

 public <K> void forward(Data<K, V> data) throws Throwable {
        ...

        List<Processor<V>> store = new ArrayList<>(childList);

        for (Processor<V> processor : childList) {

            try {
                processor.preProcess(this);
                processor.process(data.getValue());
            } finally {
                this.childList.clear();
                this.childList.addAll(store);
            }
        }
 }

public  void process(T data) throws Throwable {
    //实际处理数据
    boolean pass = filterAction.apply(data);
    if (pass) {
        Data<Object, T> result = new Data<>(this.context.getKey(), data, this.context.getDataTime(), this.context.getHeader());
        //传递个下游processor
        this.context.forward(result);
    }
}
```
#### state storage

RocketMQ Stream uses RocketMQ's Compact topic as persistent storage. In order to reduce the pressure of remote access, RocksDB is used as a local cache. When data is processed, only local storage is accessed, and the status data is persisted to the Compact topic before the consumption site is submitted.

In the main thread (workerThread) of the same stream computing instance, a state storage instance is shared; the state storage is a part of the data processing context StreamContext, which can be used at any stage of the topology map.

##### **create state topic**
![image](https://user-images.githubusercontent.com/31175234/207051854-4570b58c-39de-48ce-9f09-bb749675b8b0.png)
The queues of the source/shuffle topic have a one-to-one correspondence, when a queue of the source/shuffle topic is migrated due to rebalancing, they migrated together.

|      |     source  topic   |  state  topic  |
|  ----  |  ----  | ----  |
| topic  |MockTopicName  | MockTopicName-stateTopic |
| brokerName  |   broker1000   |broker1000    | 
| queueId  |   1   |1    | 

##### **store in locally**

```java
public interface StateStore {
	//流处理实例新增/减少消费某些queue
    void recover(Set<MessageQueue> addQueues, Set<MessageQueue> removeQueues) throws Throwable;

    //等待queue对应的状态加载
    void waitIfNotReady(MessageQueue messageQueue) throws Throwable;

	//按照key查询
    byte[] get(byte[] key) throws Throwable;

    //本地状态存储
    void put(MessageQueue stateTopicMessageQueue, byte[] key, byte[] value) throws Throwable;

    //搜索某个时刻之前的状态
    List<Pair<byte[], byte[]>> searchStateLessThanWatermark(String operatorName, long lessThanThisTime, ValueMapperAction<byte[], WindowKey> deserializer) throws Throwable;

    ...
```

##### **store in remotly**


###### **write state data**

1. Timing of state persistence: after batch data processing is completed and before offset submit;
2. Only the incremental state is persisted; the state that has not changed in this data processing will not be persisted;
3. When persisting the key-value, use a custom serialization method to save both the key and the value into body of the message;
4. Convert the key into a hexadecimal string and set it as the key of the persistent message, which is used for compact and local replay of the message;

```java
public void persist(Set<MessageQueue> messageQueues) throws Throwable {
    //由source/shuffle topic queue 转化成为state topice queue
    Set<MessageQueue> stateTopicQueues = convert(messageQueues);
    
    for (MessageQueue stateTopicQueue : stateTopicQueues) {

        //获取该queue在本轮数据处理中保存数据状态的key；
        //（保持状态时会将状态的key与state topic quue做映射）
        String stateTopicQueueKey = buildKey(stateTopicQueue);
        Set<byte[]> keySet = super.getInCalculating(stateTopicQueueKey);
 
            for (byte[] key : keySet) {
            	//状态在RocksDB中未byte array形式，不需要再次序列化
                byte[] valueBytes = this.rocksDBStore.get(key);

                byte[] body = this.protocol.merge(key, valueBytes);

                Message message = new Message(stateTopicQueue.getTopic(), body);
                message.setKeys(Utils.toHexString(key));

                //发送状态到rocketmq
                this.producer.send(message, stateTopicQueue);
            }
        }
    }
```



###### **recover**

1. State recovery timing: When rebalancing happen, a queue assgined to a stream instance, and the state corresponding to the queue needs to be loaded from the state topic to local storage.
2. Messages are grouped by queue, messages from the same queue are grouped by key, and the message with the largest queueOffset for the same key is the latest Value;
3. Store the key and the corresponding latest value Value in the local RocksDB;

```java
private void replayState(List<MessageExt> msgs) throws Throwable {
    //将msgs按照queueId进行分组
	Map<String, List<MessageExt>> groupByQueueId = ...;

    for (String uniqueQueue : groupByQueueId.keySet()) {

        //将同一queue中消息再次按照消息的key分组
        Map<String, List<MessageExt>> groupByKeyHashcode = ...;

        for (String keyHashcode : groupByKeyHashcode.keySet()) {
            
            List<MessageExt> exts = groupByKeyHashcode.get(keyHashcode);

            //对同一key的数据按照queueOffset排序
            List<MessageExt> sortedMessages = sortByQueueOffset(exts);

            //得到最新消息
            MessageExt result = sortedMessages.get(sortedMessages.size() - 1);

            //最新消息为EMPTY，表示该状态已经被删除
            String emptyBody = result.getUserProperty(Constant.EMPTY_BODY);
            if (Constant.TRUE.equals(emptyBody)) {
                    continue;
            }

            byte[] body = result.getBody();
            Pair<byte[], byte[]> pair = this.protocol.split(body);

            byte[] key = pair.getKey();
            byte[] value = pair.getValue();

            //放入rocksdb
      		this.rocksDBStore.put(key, value);
            }
        }
    }
```

###### **delete**

When a window is triggered, its corresponding state needs to be deleted from both local and remote. For example, for stream computing that includes a window operator, if the window has been triggered for a specific period of time from 10:00 to 10:05, and the data of this window is no longer processed, the state corresponding to this window needs to be deleted.

Deletion timing: after the window trigger data has been successfully transmitted to the downstream node and returned;

Use the queue-key mapping to find the status topic and queue information;

Construct a special message which the message body is empty_body (messages with an empty message body cannot be sent in RocketMQ), and send it. When the state is restored, it is found that the message body is empty, indicating that the message has been deleted.

#### Correctness

![image](https://user-images.githubusercontent.com/31175234/207052980-46638b00-8d12-4b57-8411-86e4c327e2b7.png)

Computational correctness is guaranteed by data integrity and engine consistency. If the flow computing process is regarded as a function mapping: output = f(input), in the above model, data integrity is a constraint on unbounded and unordered data sets, that is, the input in is clarified, and the engine data consistency clarifies the data Process (including output) f, so the output is deterministic.

 

##### Consistency

![image](https://user-images.githubusercontent.com/31175234/207053011-bdf3c57d-3a89-4b5d-bcbb-790c26d59a17.png)



```java
tx.begin();

try{
	process();
    sendOffset();
    sendState();
    sendSink();
}catch{
    tx.rollback();
}

tx.end();
```



At present, because producer transaction in RocketMQ is not support, this function has not yet been implemented. But it has the basis for implementation:

The process from source to sink is disassembled into two processes, source -> shuffle and shuffle -> sink, shuffle and sink are equal. The whole process can be simplified into a source -> sink process without shuffle;

stateStore and sink use the same producer;

The specific implementation requires RocketMQ to support producer transactions and will be discussed later.

##### Completeness
![image](https://user-images.githubusercontent.com/31175234/207053087-e4151262-77d3-4479-8b29-82f6d025a1ba.png)
As shown in the figure above, the width limit is 2, and the four data times are t=5, 5=7, and t=3 respectively. When passing through Operator2, the meaning of the expression is:
<div class="lake-content" typography="classic">

|data  time | Completeness | mean
|-- | -- | --|
|t=5 | Max(null, 5-2) = 3 |  The  data less than 3 are read to calculate  |
|t=7 | Max(3, 7-2) = 5 | The  data less than 5 are read to calculate  |
|t=6 | Max(5, 6-2) = 5 | The  data less than 5 are read to calculate  |
|t=3 | Max(5, 3-2) = 5 |   The  data less than 5 are read to calculate， t=3 is  the late record.  |

 In RocketMQ Stream, the data time and watermark are generated at the source, EVENT_TIME/PROCESSING_TIME determines the data time, and the data time minus the width limit time determines the watermark, and the watermark increase only, and the watermark will be transmitted downstream along with the data. For stateful operators, all states smaller than watermark need to be triggered. If the data time is less than watermark, it is considered as late data.


### Interface Design/Change

 

 

### Rejected Alternatives 

#### How does alternatives solve the issue you proposed?

#### Pros and Cons of alternatives

#### Why should we reject above alternatives