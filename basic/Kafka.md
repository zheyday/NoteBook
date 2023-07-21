## 基础

### 基本概念

|           |                                                           |
| --------- | --------------------------------------------------------- |
| Broker    | 服务器，一个Kafka集群由一个或多个Broker组成               |
| Topic     | 消息必须指定Topic。topic是一个逻辑概念，包含多个Partition |
| Partition | 最小存储单元，对应一个文件夹，存储数据和索引文件          |

如果一个topic包含多个partition，那么从topic角度来看，消息是无序的，partition内消息是有序的

#### 优点

高性能（单机QPS100w）、高可用（多副本）、低延时（毫秒级）

#### 缺点

- 主读主写，无法通过扩容解决单个broker压力问题
- 新增broker只会处理新topic，如果要分担老topic压力，需要手动迁移partition
- 消费者加入和退出会导致rebalance

### 为什么快

#### 写数据

1、批量压缩、批量发送

2、顺序写，每个partition是一个文件，kafka会把消息追加到文件末尾，顺序读写磁盘速度远大于随机

3、Memory Mapped Files内存映射文件：原理是利用操作系统的PageCache实现文件到物理内存的直接映射，对物理内存的修改会同步到磁盘（非实时），从而实现内核缓冲区和用户缓冲区的共享，省去从内核缓冲区拷贝到用户缓冲区的过程。很明显有个缺点：不可靠。https://juejin.cn/post/6956031662916534279

![image-20220830223333027](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20220830223333027.png)

#### 读数据

零拷贝，详见Netty篇

### 高可用

![在这里插入图片描述](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/20201206155011552.png)



![preview](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/view)



#### 多副本

一个Partition分区有Leader副本和Follower副本，副本的数据是相同的。生产者和消费者只会和Leader交互，Follower只负责消息的同步。

主写主读：避免了主写从读延迟导致的数据一致性问题

读写分离是为了解决负载均衡，而Kafka本身能够很好的做到负载均衡

##### leader和follower同步

AR(Assigned Replicas)：一个分区中的所有副本统称为 AR；
ISR(In-Sync Replicas)：Leader 副本和所有保持一定程度同步的 Follower 副本（包括 Leader 本身）组成 ISR；
OSR(Out-of-Sync Raplicas)：与 ISR 相反，没有与 Leader 副本保持一定程度同步的所有Follower 副本组成OSR

满足以下条件才能被认为是同步的：

- 与zookeeper之间有一个活跃的会话，也就是说，它在过去的6s（可配置）内向zookeeper发送过心跳。
- 在过去的10s（可配置）内从首领那里获取过最新的数据。

#### 故障恢复

```
LEO(log end offset)：每个副本的最后一个offset
HW(high watermark)：高水位，消费者可见的offset，是ISR中最小的LEO
```

**leader故障**

当宕机后会从所有副本中顺序查找，如果查找到的副本在ISR列表中，则当选为Leader。另外还要保证前任Leader已经是退位状态了，否则会出现脑裂情况（有两个Leader）。怎么保证？Kafka通过设置了一个controller来保证只有一个Leader。

为保证多副本的数据一致性，其他follower要将log文件中高于HW的数据截掉，然后从新的leader同步

**follower故障**

被踢出ISR，待恢复后读取本次磁盘记录的HW，并将log文件高于HW的部分截掉，开始同步leader。等到follower的LEO等于partition的HW就加入ISR

![image-20230223231556086](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20230223231556086.png)



#### ACK应答机制

request.required.asks 参数：

-  **0：** producer不等待broker的ack，这一操作提供了一个最低的延迟，broker一接收到还没写入磁盘就已经返回，当broker故障时可能丢失数据；
-  **1：** producer等待leader的ack，partition的leader落盘成功后返回ack，如果在follower同步成功之前leader故障，那么将会丢失数据；
-  **-1（all）：**producer等待broker的ack，partition的leader和ISR里的follower全部落盘成功后才返回ack。但是如果在follower同步完成后，broker发送ack之前，leader发生故障，那么会造成重复数据。（极端情况下也有可能丢数据：ISR中只有一个Leader时，相当于1的情况）。

#### 数据不丢失

1. 生产者：消息应答机制
2. broker：副本
3. 消费者：自动提交是上次消费的消息；通过offset记录消费位置，下次继续消费

#### 不重复消费

consumer在故障恢复后，需要从故障前的位置继续消费，所以需要实时记录消费的位置。在0.9版本之前，offset保存在zk中，0.9开始保存在Kafka一个内置的名字叫_consumeroffsets的topic中



#### 主读主写

生产者和消费者都是和leader副本交互的，没有采用主写从读是因为：

- 延时：从主节点同步到从节点需要经过 网络->主节点内存->主节点磁盘->网络->从节点内存->从节点磁盘
- 数据一致性：

## 常见问题

### 消息丢失

#### 生产丢失

kafka采用异步发送消息，发送结果封装在Future中。如果失败：

1. 手动重试再调用send。还失败就建信息异常表，通过定时任务重试
2. kafka支持自动重试，超过重试次数还会失败，一般不推荐

#### broker持久化丢失

异步刷盘，设置刷盘频率降低丢失概率

### 重复消费

1. 在消费之后、提交offset之前消费者宕机，会触发rebalance。再次消费的时候offset是之前提交的，所以会导致重复消费
2. 消息处理耗时，超过max.poll.interval.ms，导致认为消费者宕机，触发rebalance

**解决**

实现消费幂等：唯一索引、消息表、缓存

### 积压

生产变快或者消费跟不上
 1. 消费能力不符合预期：是否有异常，可能是某条消费失败导致反复消费
 2. 消费能力符合预期，但是跟不上生产：增加每批拉取的数量；增加消费线程；增加partition同时增加消费者数量；

### 顺序消费

全局有序：一个topic下的所有消息都是有序的，需要1个Topic只能对应1个Partition

局部有序：一个topic下，同一业务字段的消息有序。消费者接收到消息后，对消息再次hash路由到不同的队列，然后多个线程消费对应的队列

消息重试的影响：生产者发送两条消息A、B，A发送失败，B发送成功，A通过重试也成功了，这时候消息顺序就成了BA。针对这种情况，需要配置max.in.flight.requests.per.connection参数，该参数指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，同时也会提升吞吐量。把它设为1就可以保证消息是按照发送的顺序写入服务器的。

![img](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/2795758f78f0e1f7278f151b5200e438.png)

## 其他

### 分区分配策略

**RangeAssigor**

按单个topic进行分区分配，

**RoundRobinAssignor**



**StickyAssignor**

在上一次分配的结果的基础上，尽量少的调整分区分配的变动，节省因分区分配变化带来的开销



## 基本操作

https://spring.io/projects/spring-kafka#overview

```cmd
#启动
kafka-server-start.bat ..\..\config\server.properties
#创建topic
kafka-topics.bat --bootstrap-server 192.168.31.23:9092 --create --replication-factor 1 --partitions 1 --topic zcs-kafka

kafka-topics.bat --bootstrap-server 192.168.31.23:9092 --list
```



