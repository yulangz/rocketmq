# Status
Current State: Drafted   
Authors: ferrirw    
Shepherds:  duhenglucky    
Mailing List discussion: [dev@rocketmq.apache.org](mailto:dev@rocketmq.apache.org)    
Pull Request:   
Released:  
Related Docs:  

 
# Background & Motivation
## What do we need to do
Currently, RocketMQ Message has no Schema constraint, and the serialization and deserialization processes are entirely left to users. The following problems may occur:
1. Type safety: For a data flow scenario built around RocketMQ, producers and consumers may be completely different teams, and incompatible changes to the data format made upstream can cause downstream data to fail to process properly and recover quickly
2. Application extension: Serialization and deserialization processes are coupled with application logic, for example, for structured or semi-structured data ETL scenarios, because no schema definition exists, the data parsing logic is rewritten each time ETL is built, and serialization efficiency cannot be guaranteed; Similarly, the Connector and stream extensions face the same problem
Therefore, we considered building a Schema management center to enhance type security and application extensibility for structured and semi-structured data


目前 RocketMQ Message 并无 Schema 约束，序列化与反序列化过程完全交给用户，可能存在以下问题：
1. 类型安全：对围绕 RocketMQ 构建的数据流场景，生产者和消费者可能是完全不同的团队，当上游对数据格式进行了微小但不兼容的修改后，会导致下游无法正常处理数据，并且无法快速恢复 
2. 应用扩展：序列化和反序列化过程与应用逻辑耦合，例如对于结构化或半结构化的数据 ETL 场景，因为不存在 schema 定义，每次构建 ETL 时都会重写数据的解析逻辑，并且无法保证序列化效率；同样的，connector 和 stream 扩展也面临同样的问题 
因此我们考虑构建 Schema 管理中心，来增强结构化、半结构化数据的类型安全和应用可扩展性
# Goals
## What problem is this proposal designed to solve? 
When produce message with a fixed schema, you will not need to serialize messages into bytes, instead producer with schema do this job in the background. This solves the problem of type-safe and application extension.

当发送结构化消息时，用户不再需要将消息序列化为字节，而是由新的生产者在后台进行格式化校验和序列化工作，同时由新的消费者在读取时将字节反序列化为结构化数据，这解决了上述类型安全和应用程序扩展的问题。
# Non-Goals.
## What problem is this proposal NOT designed to solve? 
- Column storage 
- Data lineage 
- Schema field supports references

- 列存 
- 数据血缘 
- 支持引用
## Are there any limits of this proposal? 
- After topic bound to a schema, the old producer which without the Schema may not be able to sending message to broker

- 当 Topic 与 Schema 绑定后，旧的不带 Schema 的 Producer 可能无法继续写入数据 
# Changes
- Added Schema-Registry as RocketMQ unified schema management center (including connector / stream) 
- Added a client with built-in Schema format validation, serialization, and deserialization capabilities 
- Support avro, json, thrift, protobuf and other type extensions 
- Schema storage layer plug-in, support rocketmq compact topic, mysql and other storage expansion


- 新增 Schema-Registry 作为 RocketMQ 统一的 Schema 管理中心（包括 connector / stream） 
- 新增内置 Schema 格式化校验、序列化、反序列化能力的客户端 
- 支持 avro，json，thrift，protobuf 等类型拓展 
- Schema 存储层插件化，支持 rocketmq compact topic、mysql 等存储拓展 
# Architecture
## Some key designs of Schema Registry:
- The deployment architecture is similar to namesrv 
- The storage layer is extensible, using RocketMQ Compaction Topic by default 
- Provide interfaces for creating, modifying, querying, and binding schemas, and restrict schema version evolution according to topic-level compatibility policies 
- Each Schema has a globally unique ID 
- The client has a built-in module to manage serialization/deserialization during message sending/reading
- Schema can be automatically created or updated on the Producer.
- Rocketmq-connector are compatible
- Flink and Metacat are compatible via NewApis

- 采用分布式架构，类似 namesrv 的横向扩展机制 
- 存储层可扩展，默认使用 RocketMQ Compaction Topic 
- 提供接口用于创建、修改、查询、绑定多种类型的 schema，并根据 Topic 级别的兼容性策略限制 Schema 的版本演化 
- 每个 Schema 具有全局唯一 ID，由 SchemaID 与 Version 组成，每次变更时 ID 保持不变仅 Version 单调递增 
- 客户端内置 Schema Wrapper 和 Serializer/Deserializer 模块，Schema Wrapper 用于解析消息中的 Schema 并进行格式化校验，Serializer/Deserializer 负责消息发送/读取过程中的序列化/反序列化 
- 支持 Producer 端自动创建/更新 Schema，需要参考兼容性-升级顺序，可通过配置关闭 
- 兼容 rocketmq-connector
- Flink 和 Metacat 可以通过 NewApis 兼容结构化数据 

## Data flow
![image](https://user-images.githubusercontent.com/16487356/183577841-8c8599fe-745a-44ff-a298-9fb7dc5d671f.png)

As shown in the figure above:
1. During the sending process, the Message Schema will be parsed and sent to the Schema Registry to check whether it conforms to the Topic Schema compatibility policy. If it passes, the SchemaMeta(Id) will be returned to serialize the data. If the verification fails, the sending will be rejected; 
2. The consumer will first parse the Message SchemaMeta(Id), then perform compatibility verification with the Topic Schema, and return the Schema for deserialization if the verification passes. 
3. What the user sees is always structured data similar to public class User {  String name;  int age;  }
 

如上图所示：
1. 发送过程中 Message Schema 会被解析出来交给 Schema Registry 校验是否符合 Topic Schema 的兼容性策略，如果通过则返回 SchemaMeta 用于序列化数据，校验失败则拒绝发送； 
2. 反之，消费者会先解析 Message SchemaMeta 然后与 Topic Schema 进行兼容性校验，校验通过则返回 Schema 进行反序列化 
3. 用户视角发送和接收到的都是类似 public class Order {  String user;  int id;  } 的结构化数据 

## Compatibility
### Compatible strategy
The default is BACKWARD, which means that each change to the Schema has no impact on the running programs.
Transitivity specifies the compatibility check scope for each change. All previous versions indicates that the compatibility check is performed on all versions of the Schema. Last version indicates that the compatibility check is performed only with the last version.

默认为 BACKWARD，表示 Schema 的每一次变更都不会对存量正在运行的程序造成影响
可传递性明确了每次变更需要进行兼容性校验的范围，可传递表示需要在该 Schema 的全部版本中进行兼容性检查，不可传递表示仅与上一个版本进行对比


Compatible strategy | Permitted changes | Transitivity | Upgrade order
-- | -- | -- | --
BACKWARD | Delete fieldsAdd <br> optional fields | Last version | Consumers
BACKWARD_TRANSITIVE | Delete fieldsAdd <br> optional fields | All previous versions | Consumers
FORWARD | Add fieldsDelete <br> optional fields | Last version | Producers
FORWARD_TRANSITIVE | Add fieldsDelete <br> optional fields | All previous versions | Producers
FULL | Modify optional fields | Last version | Any order
FULL_TRANSITIVE | Modify optional fields | All previous versions | Any order
NONE | All changes are accepted | Compatibility checking disabled | Depends

### relationships of Schema and Topic
To reduce the coupling between Schema Registry and RocketMQ, Subjects were introduced to represent resource collections, with each Subject composed of a set of topics and Schema associations

为了降低 Schema Registry 和 RocketMQ 的耦合度，引入 Subjects 用于表示资源集合，每个 Subject 都由一组 Topic 和 Schema 关联构成
![image](https://user-images.githubusercontent.com/16487356/183578707-812223dc-8ba9-4647-a6b6-e9478032bc05.png)

Subject can also be treated as a Table, for example in Flink SQL:
```
create table rocketmq_table (
  timestamp BIGINT,
  user_id VARCHAR,
  user_operation VARCHAR,
) with (
  'connector.topic'='topic',
  'update-mode'='append',
  'format.type'='thrift',
  'format.thrift-class'='com.xiaomi.xx',
  'format.thrift-package'='com.xiaomi.xx:0.0.1-SNAPSHOT',
);
```
### Compatible for RocketMQ connector
The current RocketMQ Connector stores Schema information in MessageExt Property.
![image](https://user-images.githubusercontent.com/16487356/183579337-3ad672ef-0004-427e-8599-795aca9f3cf5.png)

If we want to reduce the amount of data transferred:
- Schema information is hosted by SchemaRegistry instead of being passed through messaging
- SchemaRegistry converts ConnectRecord to other types through Converter

The expected effect is shown below:
![image](https://user-images.githubusercontent.com/16487356/183579411-67bc6d81-ecf6-45c5-a875-f7cc5b1ebc98.png)

### Support types
#### Primitive type 
BOOLEAN | 1 比特二进制数值
-- | --
INT8/16/32/64 | 8 / 16 / 32 / 64 位有符号整数
FLOATE | 单精度浮点数
DOUBLE | 双精度浮点数
BYTES | 字节序列
STRING | Unicode 字符集序列
TIMESTAMP | 时间戳，保存形式为 64 位有符号整数

#### Complex type 
Type | Desc | Scenario | Scenario Desc
-- | -- | -- | --
struct | AVRO, JSON, Thrift and Protobuf | specific | 已知要发送消息的数据类型、依赖
generic | 不知道要发送消息的数据类型
Binlog | Reserved for Binlog | mysql2rocketmq | 支持 mysql、tidb 等
Text | Reserved for one-line-string | log2es | 支持日志场景

#### Nested type 
Not supported now

## Interface Design/Change
### Storage
```
public interface Storage {
    // To created / updated schemas
    CompletableFuture<> update(String id, byte[] schemaData);

    // Fetch schemaData by schemaId from storage
    CompletableFuture<SchemaData> get(String id);
    
    // Fetch all schemaDatas from storage
    CompletableFuture<> list();

    // Deleted schemas 
    CompletableFuture<> delete(String id);
    
    // Create reference from topic to schema
    CompletableFuture<Subject> bind(String id, String topicname)
    
    // Delete reference from topic to schema
    CompletableFuture<> unbind(String id, String topicname)
    
    // TODO support Table API
}
```

### Compiler 
```
public interface Compiler {

  // Generate java file like thrift:/gen-java/rpc.java
  public CompletableFuture<Map<String, ByteBuffer>> compile(String schemaIdl, String SchemaType);

  // upload to package management center, like artifactory
  public CompletableFuture<> upload(Map<String, ByteBuffer> schemaClass, String context);
}
```

### NewApis: Producer / Consumer with Schema 
```
AvroSchema schemaWrapper = AvroSchema.of();
Producer<T> producer = new ProducerBuilder<T>()
    .config(producerConfig)
    .topicName(topicName)
    .schema(schemaWrapper)
    .build();
    
producer.send(new MessageBuilder<T>().message(messageStr).build())

Consumer<T> consumer = new ConsumerBuilder<T>()
    .topicName(topicName)
    .config(consumerConfig)
    .schema(schemaWrapper)
    .groupId("ConsumerA")
    .clientId("client-1")
    .build();

Record<T> record = consumer.consume();

if(recors.isDeserialized()) {
    T message = record.message();
} else {
    byte[] message = result.undeserializedMessage();
}
```

### REST Definition 
refer：https://github.com/openmessaging/openschema/blob/master/spec_cn.md

## Compatibility, Deprecation, and Migration Plan
### Are backward and forward compatibility taken into consideration? 
Nothing specific. 
### Are there deprecated APIs?
Nothing specific. 
### How do we do migration? 
Nothing specific. 
## Implementation Outline
We will implement the proposed changes by 2 phases.
### Phase 1
Finish the development of storage layer, compiler layer, client and compatibility verification tool, and supports avro and THRIFT types

### Phase 2
- Extends type support of protobuf and JSON
- Supports advanced features such as monitoring, rights management, and version comparison
- Support compatible with RocketMQ-Connector, Metacat, and Flink

