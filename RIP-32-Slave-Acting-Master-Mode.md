# Status
- Current State: Proposed
- Authors: [RongtongJin](https://github.com/RongtongJin)
- Shepherds: duhengforever@apache.org
- Mailing List discussion: dev@rocketmq.apache.org
- Pull Request: 
- Released: <relased_version>
- Related Docs: [Slave Acting Master Mode](https://docs.google.com/document/d/1LCTkfem6wcV-PqWjzPAvaxgqFMCj3ymPakDrcodX9ic/edit?usp=sharing)

# Background & Motivation
## What do we need to do
- Will we add a new module?   
 
  No
- Will we add new APIs?    

  For the client, there are no new APIs. There will be new methods in the broker to enable slave acting master when the master is at fault. The broker and nameserver will have new APIs but ensure backward compatibility.
- Will we add a new feature? 
  
  Yes, slave acting master mode is a new feature.


## Why should we do that

- Are there any problems of our current project?

**Master-slave deployment**

![](https://s4.ax1x.com/2022/02/05/HnW3CQ.png)

The figure above shows the RocketMQ master-slave deployment. Under this deployment mode, even if one master is down, the producer can still send messages to other masters. For the consumer, if the slave read is enabled, the consumer will automatically reconnect to the slave without consumption stagnation. However, there are also the following problems:

1. Some operations limited to the master cannot be performed, including but not limited to:

- searchOffset
- maxOffset
- minOffset
- earliestMsgStoreTime
- endTransaction
- All lock operations, including lock, unlock, lockbatch, and unlockall

The specific impacts are:

- The client cannot obtain the lock of MQ in the replica group, so when the local lock expires, it will not be able to consume order messages of the broker group
- The client cannot actively end the transaction message in half state, and can only wait for the broker to check the transaction state
Operations such as querying offset and earliestMsgStoreTime in admin tools or control cannot take effect on this broker group


2. The consumption of special messages on the failed broker group will be interrupted. The characteristics of such messages depend on the thread on the master broker to scan the special topics on the commit log and deliver the messages that meet the requirements back to the commit log. If the master broker goes offline, the consumption of special messages will be delayed or lost. This will affect the consumption of delayed messages, transaction messages, and pop messages in the current version.

3. There is no reverse synchronization of metadata. After the master is manually pulled up again, it is easy to cause the fallback of metadata. For example, after the master goes online, it synchronizes the backward consumer offset to the slave, and the consumer offset of this group of brokers rollback, resulting in a large number of repeated consumption.


**DLedger deployment**

![](https://s4.ax1x.com/2022/02/05/HnWwUU.png)

The above figure shows the dledger (raft) deployment, which can avoid the above issues by selecting a new master. However, it can be seen that three or more replicas are required in one broker group under the dledger deployment.

RIP-32 put forward a new proposal, slave acting master mode, as an upgrade of master-slave deployment mode. Under the original master-slave deployment mode, the slave will play a more important role in case of failure of the same group master through slave acting master, lightweight heartbeat, replica group information acquisition, broker pre-online mechanism, special message escape, etc.


- What can we benefit from proposed changes?  

1. When the master goes offline, the slave with the smallest brokerId in the broker group will undertake the tasks of slave reading and some tasks that the client and control can access but can only be completed on the master node. 
2. After the master goes offline, the special message consumption on the failed broker group will not be interrupted. The slave with the smallest brokerId in the broker group will undertake the task, and the delayed messages, pop messages, transaction messages, etc. can still operate normally.
3. After the master goes offline, the slave acts as the master for a period of time. Then when the Master goes online again, through the pre-online mechanism, the master will automatically complete the reverse synchronization of metadata before going online. There will be no metadata roll back, resulting in a large number of repeated consumption of messages or a large number of replay of special messages.


# Goals
- What problem is this proposal designed to solve?  

In master-slave deployment mode, after the master goes offline, some operations that can only be performed on the Master cannot be performed, special message consumption is interrupted, and metadata is rollback after the master goes online again.

# Non-Goals.

- What problem is this proposal NOT designed to solve? 
 
This RIP is not intended to change the original RocketMQ architecture. It is not intended to replace the master-slave architecture or DLedger architecture. It is an enhancement of the master-slave architecture.

# Changes
## Architecture

### Slave Acting Master

![](https://s4.ax1x.com/2022/02/05/HnWgDx.png)

In RIP-32, when the master goes offline, the slave can consume normally, and the operation can only be performed by the master without modifying the client code. This is because nameserver supports the acting master mode. Acting master here means that nameserver will treat the living slave with the smallest broker id as an acting master when the broker group is no master, by replacing the broker id for that slave with 0 when broker permission is changed to 4 (read-only) in topic route so that the slave acts as the master in read-only mode in the client view.

```java
public void changeSpecialServiceStatus(boolean shouldStart) {
	……

    changeScheduleServiceStatus(shouldStart);

    changeTransactionCheckServiceStatus(shouldStart);

    if (this.ackMessageProcessor != null) {
        LOG.info("Set PopReviveService Status to {}", shouldStart);
        this.ackMessageProcessor.setPopReviveServiceStatus(shouldStart);
    }
}
```

In addition, when the Master goes offline, the Slave with the smallest broker id takes over the scanning and redelivery of special messages.

### Lightweight heartbeat

As mentioned above, the surviving Slave with the smallest broker id will act master after master failure and therefore requires a mechanism that ensures that:

1. Nameserver can detect brokers up and down in time and complete topic route replacement and topic route elimination from offline brokers.

2. Brokers are aware of the ups and downs of a group of brokers in time.

First, Nameserver already has an active mechanism, which periodically scans inactive brokers and takes them offline. The "heartbeat" between the original broker and Nameserver relies on registerBroker operation, which involves topic information reporting and is too heavy. RIP-32 adds BrokerHeartbeat requests between Nameserver and broker, which will send the lightweight heartbeat to nameserver at regular intervals. If a scheduled nameserver task detects that it has not received the heartbeat of the broker after the heartbeat timeout period, it will unregister the broker. The heartbeat timeout is set when registerBroker, and if a change to the minimum broker id is found in a broker group, all brokers in that group are notified. And replace the slave topic route of the smallest broker id to act as the master in read-only mode when the topic route is fetched.

Second, there are two mechanisms to sense when a group of brokers is on or off in a timely manner. First is when the nameserver detects a change in the smallest broker id in the group described above, it reversely notifies all brokers in the group. Secondly, RIP-32 adds a GetBrokerMemberGroup request to the nameserver to get the broker replica group.

```java
if (this.brokerConfig.isEnableSlaveActingMaster()) {
    scheduledFutures.add(this.brokerHeartbeatExecutorService.scheduleAtFixedRate(new AbstractBrokerRunnable(this.brokerConfig) {
        @Override
        public void run2() {
            if (isIsolated) {
                return;
            }
            try {
                BrokerController.this.sendHeartbeat();
            } catch (Exception e) {
                BrokerController.LOG.error("sendHeartbeat Exception", e);
            }


        }
    }, 1000, brokerConfig.getBrokerHeartbeatInterval(), TimeUnit.MILLISECONDS));


    scheduledFutures.add(this.syncBrokerExecutorService.scheduleAtFixedRate(new AbstractBrokerRunnable(this.brokerConfig) {
        @Override public void run2() {
            try {
                BrokerController.this.syncBrokerMemberGroup();
            } catch (Throwable e) {
                BrokerController.LOG.error("sync BrokerMemberGroup error. ", e);
            }
        }
    }, 1000, this.brokerConfig.getSyncBrokerMemberGroupPeriod(), TimeUnit.MILLISECONDS));
}
```

The slave broker that is the smallest broker id in the broker group will start acting mode. Once the master Broker comes back online, the slave broker will also be aware of this through the nameserver’s notification or its own scheduled task to synchronize information about broker replicas information, and the acting mode is automatically terminated.

### Special message escape

After the acting mode is enabled, the slave with the smallest broker id will undertake the scanning and redelivery of special messages.

We can scan the slave commit log, but if the scanned message is re-delivered to the commitlog, the unwritten semantics of the slave will be destroyed. Therefore, RIP-32 proposes an escape mechanism to deliver the special messages to the commit log of other masters remotely or locally.

- Remote Escape

![](https://s4.ax1x.com/2022/02/05/HnWWVK.png)

As shown in the figure above, assuming that region a fails, node 2 in region B will undertake the task of scanning specail messages, and send the final messages remotely to the surviving master in the current broker cluster through EscapeBridge.

- Local Escape

![](https://s4.ax1x.com/2022/02/05/HnWfUO.png)

Local escape needs to be operated in the BrokerContainer mode (see RIP-31 for details). If there is a surviving master in the BrokerContainer, it will give priority to escape to the master commit log of the same process to avoid remote RPC.

**Behavior changes of various special messages**

- Delayed Message

When the slave acts the master, the scheduleMessageService will start, and the delayed messages time up will escape to the local master first through EscapeBridge. If not, they will escape to the remote master. Messages stored on the broker that has not expired in time will be escaped to other surviving masters. In terms of data volume, if a large number of delayed messages on the broker have not expired, remote escape will cause large data flow within the cluster, but it is basically controllable.

- POP Message

1. Add the broker name attribute when CK/ACK spells a key. This allows each broker to cancel CK/ACK messages from other brokers while scanning revive topic.

2. The CK/ACK message on the slave will be escaped to another specified master-A (the same master is required; otherwise, CK/ACK cannot cancel and the message is duplicated). Master-A scans its commit log message and cancels it. Based on the information in the CK message, the messages are pulled to the slave (locally if present, remotely otherwise) and then dropped to a local retry topic

- Transaction message

### Pre-online mechanism

![](https://s4.ax1x.com/2022/02/05/HnW5Pe.png)

After the master broker goes offline, the slave broker will play the role of standby reading and proxy secondary messages. Therefore, some metadata in the slave broker, including consumer offset and delayed message progress, will be more advanced than the offline master broker. If the master broker goes online again, the slave broker metadata will be overwritten by the master broker, and this group of broker metadata will be rolled back, which may cause a large number of duplicate messages. Therefore, a pre-online mechanism is needed to complete the reverse synchronization of metadata.

The concept of the data version needs to be added to metadata such as consumer offset and delay offset. In order to prevent too frequent version number updates, the concept of update step size needs to be added. For example, for consumer offset, the version number is increased to the next version more than 500 update times.

As shown in the above figure, the master broker will be in pre-online progress before it is launched. Before it is in pre-online progress, it will not be visible to the client (the broker will have isIsolated flag, and when it is true, it will not register and send heartbeat to nameserver). Therefore, it will not provide services to the client, and the scanning process of special messages will not be started. The specific pre-online mechanism is as follows:

1. The master broker obtains the slave broker address from the nameserver but does not register.
2. Master broker sends its own status information and address to slave broker.
3. After the slave broker obtains the master broker address and status information, it establishes the HA connection, completes the handshake, and enters the transfer state.
4. After the master broker completes the handshake, it reversely obtains the metadata of the slave, including the consumer offset, delayed message progress, etc., and determines whether to update it according to the data version.

After completing steps 1-4 for all slave brokers in the broker group, the master broker officially goes online, registers with the nameserver, and officially provides services to the client.

### Lock Quorum

When the slave acts as the master, the external view is a "read-only" master, so order messages can still lock the queue. However, after the master is re-launched, it may cause multiple consumers to lock the same queue within a certain time. For example, a consumer still locks a slave broker’s queue, and a consumer locks the same queue of the master, resulting in disorder and repetition of order messages.

Therefore, during lock operation, RIP-32 requires that the quorum of the brokers in the replica group be locked successfully. However, the two replicas cannot reach the principle of Qurum, so RIP-32 provides the lockInStrictMode parameter to indicate whether the strict mode is used when the consumer locks the queue of order messages. Strict mode means that for a single queue, the lock is successful only when the quorum members of the replica group are locked successfully. Non-strict mode means that the lock is successful even if any replica is locked successfully. This parameter is false by default. When the ordering of messages is higher than the availability, please set this parameter to false.

## Interface Design/Change
- Method signature changes

Requests and API changes

New request codes are added

```java
public static final int GET_BROKER_MEMBER_GROUP = 901;

public static final int BROKER_HEARTBEAT = 904;

public static final int NOTIFY_MIN_BROKER_ID_CHANGE = 905;

public static final int EXCHANGE_BROKER_HA_INFO = 906;
```

BrokerContainer Interface Design

```java
/**
 * An interface for broker container to hold multiple master and slave brokers.
 */
public interface IBrokerContainer {

    /**
     * Start broker container
     */
    void start() throws Exception;

    /**
     * Shutdown broker container and all the brokers inside.
     */
    void shutdown();

    /**
     * Add a broker to this container with specific broker config.
     *
     * @param brokerConfig the specified broker config
     * @param storeConfig the specified store config
     * @return the added BrokerController or null if the broker already exists
     * @throws Exception when initialize broker
     */
    BrokerController addBroker(BrokerConfig brokerConfig, MessageStoreConfig storeConfig) throws Exception;

   /**
     * Remove the broker from this container associated with the specific broker identity
     *
     * @param brokerIdentity the specific broker identity
     * @return the removed BrokerController or null if the broker doesn't exists
     */
    BrokerController removeBroker(BrokerIdentity brokerIdentity) throws Exception;

    /**
     * Return the broker controller associated with the specific broker identity
     *
     * @param brokerIdentity the specific broker identity
     * @return the associated messaging broker or null
     */
    BrokerController getBroker(BrokerIdentity brokerIdentity);

    /**
     * Return all the master brokers belong to this container
     *
     * @return the master broker list
     */
    Collection<BrokerController> getMasterBrokers();

    /**
     * Return all the slave brokers belong to this container
     *
     * @return the slave broker list
     */
    Collection<InnerSalveBrokerController> getSlaveBrokers();

   /**
     * Return all broker controller in this container
     *
     * @return all broker controller
     */
    List<BrokerController> getBrokerControllers();

    /**
     * Return the address of broker container.
     *
     * @return broker container address.
     */
    String getBrokerContainerAddr();

    /**
     * Peek the first master broker in container.
     *
     * @return the first master broker in container
     */
    BrokerController peekMasterBroker();

    /**
     * Return the config of the broker container
     *
     * @return the broker container config
     */
    BrokerContainerConfig getBrokerContainerConfig();

    /**
     * Get netty server config.
     *
     * @return netty server config
     */
    NettyServerConfig getNettyServerConfig();

   /**
     * Get netty client config.
     *
     * @return netty client config
     */
    NettyClientConfig getNettyClientConfig();

    /**
     * Return the shared BrokerOuterAPI
     *
     * @return the shared BrokerOuterAPI
     */
    BrokerOuterAPI getBrokerOuterAPI();

    /**
     * Return the shared RemotingServer
     *
     * @return the shared RemotingServer
     */
    RemotingServer getRemotingServer();
}

```

Data structure in BrokerContainer

```java
public class BrokerContainer implements IBrokerContainer {
    private static final InternalLogger LOG = InternalLoggerFactory.getLogger(LoggerName.BROKER_LOGGER_NAME);
    private final ScheduledExecutorService scheduledExecutorService = new ScheduledThreadPoolExecutor(1,
        new BasicThreadFactory.Builder()
            .namingPattern("BrokerContainerScheduledThread")
            .daemon(true)
            .build());
    private final NettyServerConfig nettyServerConfig;
    private final NettyClientConfig nettyClientConfig;
    private final BrokerOuterAPI brokerOuterAPI;
    private final ContainerClientHouseKeepingService containerClientHouseKeepingService;


    private final ConcurrentMap<BrokerIdentity, InnerSalveBrokerController> slaveBrokerControllers = new ConcurrentHashMap<>();
    private final ConcurrentMap<BrokerIdentity, InnerBrokerController> masterBrokerControllers = new ConcurrentHashMap<>();
    private final List<BrokerBootHook> brokerBootHookList = new ArrayList<>();
    private final BrokerContainerProcessor brokerContainerProcessor;
    private final Configuration configuration;
    private final BrokerContainerConfig brokerContainerConfig;


    private RemotingServer remotingServer;
    private RemotingServer fastRemotingServer;
    private ExecutorService brokerContainerExecutor;
    ……
}
```

![](https://s4.ax1x.com/2022/01/26/7LQ3GD.png)

The BrokerContainer will manage multiple innerBrokerControllers and innerSlaveBrokercontrollers. InnerBrokerController inherits from the brokerController and has the ability to reuse the brokerController. InnerSlaveBrokerController inherits from the innerBrokerController and has the ability to reuse the innerBrokerController, InnerBrokerController and innerSlaveBrokerController in the same BrokerContainer share the transport layer.

 
The BrokerContainer is configured as follows
```java
public class BrokerContainerConfig {


    private String rocketmqHome = System.getProperty(MixAll.ROCKETMQ_HOME_PROPERTY, System.getenv(MixAll.ROCKETMQ_HOME_ENV));


    @ImportantField
    private String namesrvAddr = System.getProperty(MixAll.NAMESRV_ADDR_PROPERTY, System.getenv(MixAll.NAMESRV_ADDR_ENV));


    @ImportantField
    private boolean fetchNamesrvAddrByAddressServer = false;


    @ImportantField
    private String brokerContainerIP = RemotingUtil.getLocalAddress();


    private String brokerConfigPaths = null;
    
    ……
}
```

- CLI command changes

If the producer and consumer have not modified, the admin tool will add a new command to complete the add broker and remove broker operations on the BrokerContainer.

AddBrokerCommand
```
usage: mqadmin addBroker -b <arg> -c <arg> [-h] [-n <arg>]
 -b,--brokerConfigPath <arg>      Broker config path
 -c,--brokerContainerAddr <arg>   Broker container address
 -h,--help                        Print help
 -n,--namesrvAddr <arg>           Name server address list, eg: 192.168.0.1:9876;192.168.0.2:9876
```

RemoveBroker Command
```
usage: mqadmin removeBroker -b <arg> -c <arg> [-h] [-n <arg>]
 -b,--brokerIdentity <arg>        Information to identify a broker: clusterName:brokerName:brokerId
 -c,--brokerContainerAddr <arg>   Broker container address
 -h,--help                        Print help
 -n,--namesrvAddr <arg>           Name server address list, eg: 192.168.0.1:9876;192.168.0.2:9876
```

- Log format or content changes

In the BrokerContainer mode and after log separation is enabled, the default output path of the log will change, and the specific path of each broker log will change to

{user.home}/logs/{$brokerCanonicalName}_rocketmqlogs/
 
brokerCanonicalName is {BrokerClusterName_BrokerName_BrokerId}.


## Compatibility, Deprecation, and Migration Plan
- Are backward and forward compatibility taken into consideration?
  
  Yes, There is no change in the specific functions and APIs of the broker in the BrokerContainer. There is no change in the functions and APIs of the broker when the user starts the broker in the normal way (not in the BrokerContainer way), and there is no change in the interaction between the broker and the client and the nameserver.

- Are there deprecated APIs?
  
  Nothing specific.

- How do we do migration?
  
**Scenario 1: Normal upgrade**

![](https://s4.ax1x.com/2022/01/26/7LQGxH.png)

The goal is to change the original master-slave to the mutual master-slave in the figure above.

Take this broker group offline and upgrade to a new version of the RocketMQ package. Modify the broker container configure file, configure brokerConfigPaths according to the expected master-slave relationship, and complete the specific configuration of the configuration files of the two brokers.

Each broker of the same BrokerContainer needs to have a different port number. If the configuration of the original broker's storePathRootDir and storePathCommitlog changes, data migration is also required. Copy the files in the store directory to the new directory and start the BrokerContainer.

Note: if the original broker has an amount of data, after configuring quotaPercentForDiskPartition and logicalDiskSpaceCleanForciblyThreshold, it will quickly clean up to the logical threshold, which needs to be adjusted dynamically.

**Scenario 2: The node offline and cleanup, replace it to the new version** 

1. Set broker that needs to be upgraded write permission to false, waiting for the data to be completely consumed
2. Take the broker in step 1 offline and clean up relevant data
3. Start and join the cluster in BrokerContainer mode.

## Implementation Outline

We will implement the proposed changes by 3 phases.
### Phase 1 

Support master-slave mode. 

Phase 1 can be divided into the following steps:
1. Implementation of transport layer sharing and subRemotingServer
2. Implement the BrokerContainer module and the admin command
3. Implement log separation mechanism
4. Implement clean logic for BrokerContainer
Phase 1 will be supported in the 5.0 branch first.

### Phase 2
Support dledger mode.

### Phase 3
According to the isolation classification of thread pools, make thread pools shared.

# Rejected Alternatives

## How does alternatives solve the issue you proposed?

Deploy multiple broker processes directly on a single node

## Pros and Cons of alternatives

Pros:

Better isolation between brokers

Cons:

1. When deploying multiple broker processes on a single node, the resources are managed by the operating system, while the brokers in BrokerContainer are managed by the RocketMQ process. RocketMQ can better control each broker, and some resources can also be fully reused such as transport layer, thread pool, etc.
2. When deploying multiple broker processes on a single node, it cannot provide a unified view for RocketMQ, but the BrokerContainer can provide a unified view to complete the centralized control of brokers. For example, the BrokerContainer can monitor the capacity occupation of every broker and handle them according to the situation.
3. In order to isolate resources, deploying multiple broker processes on a single node may need to be through docker, but the broker container does not need to install docker. RocketMQ provides the ability of container natively, and the deployment and installation cost is lower.
4. From the perspective of single process RocketMQ, the resource utilization is improved.

## Why should we reject above alternatives

Based on the comparison of Pros and Cons above, RIP-31 provides BrokerContainer capability. 
