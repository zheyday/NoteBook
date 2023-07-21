# ES

### 基础

#### 什么是es

ES是基于 Lucene 的 Restful 的分布式实时全文搜索引擎，每个字段都被索引并可被搜索，可以快速存储、搜索、分析海量的数据。

shard 分片数量在建立索引时设置，设置后不能修改

#### 为什么不用B+树



#### 正排索引

文档和关键词的映射。如果找关键词，那么需要遍历所有文档，然后找出包含关键字的文档

#### 倒排索引

关键词和文档的映射

单词词典：记录单词信息和对应倒排列表的位置

倒排列表：记录出现过某个单词的所有文档列表及单词在文档中的位置

#### text 和 keyword类型的区别

两个类型的区别主要是分词：

keyword 类型不会分词，直接根据字符串内容建立倒排索引，所以keyword类型的字段只能通过精确值搜索到；

Text 类型在存入 Elasticsearch 的时候，会先分词，然后根据分词后的内容建立倒排索引

#### FST



#### 脑裂

如果集群中选举出多个Master节点，使得数据更新时出现不一致，这种现象称之为脑裂。

简而言之集群中不同的节点对于 Master的选择出现了分歧，出现了多个Master竞争。

一般而言脑裂问题可能有以下几个**原因**造成：

- **网络问题：**集群间的网络延迟导致一些节点访问不到Master，认为Master 挂掉了，而master其实并没有宕机，而选举出了新的Master，并对Master上的分片和副本标红，分配新的主分片。
- **节点负载：**主节点的角色既为Master又为Data，访问量较大时可能会导致 ES 停止响应（假死状态）造成大面积延迟，此时其他节点得不到主节点的响应认为主节点挂掉了，会重新选取主节点。
- **内存回收：**主节点的角色既为Master又为Data，当Data节点上的ES进程占用的内存较大，引发JVM的大规模内存回收，造成ES进程失去响应。

如何避免脑裂：我们可以基于上述原因，做出优化措施：

- 适当调大响应超时时间，减少误判。通过参数 discovery.zen.ping_timeout 设置节点ping超时时间，默认为 3s，可以适当调大。
- 选举触发，我们需要在候选节点的配置文件中设置参数 discovery.zen.minimum_master_nodes的值。这个参数表示在选举主节点时需要参与选举的候选主节点的节点数，默认值是 1，官方建议取值(master_eligibel_nodes/2)+1，其中 master_eligibel_nodes 为候选主节点的个数。这样做既能防止脑裂现象的发生，也能最大限度地提升集群的高可用性，因为只要不少于 discovery.zen.minimum_master_nodes个候选节点存活，选举工作就能正常进行。当小于这个值的时候，无法触发选举行为，集群无法使用，不会造成分片混乱的情况。
- 角色分离，即是上面我们提到的候选主节点和数据节点进行角色分离，这样可以减轻主节点的负担，防止主节点的假死状态发生，减少对主节点宕机的误判。

#### 选举

**什么时候发起**

1. 集群启动
2. Master探测到节点掉线
3. 非Master探测到Master掉线

**es7之前**

采用bully算法，

1. ping所有节点，获取fullPingResponses列表

2. 遍历fullPingResponses：将各节点认为的master加入到activeMasters列表，不包括当前节点

   ```java
   List<DiscoveryNode> activeMasters = new ArrayList<>();
     for (ZenPing.PingResponse pingResponse : pingResponses) {
         // We can't include the local node in pingMasters list, otherwise we may up electing ourselves without
         // any check / verifications from other nodes in ZenDiscover#innerJoinCluster()
         if (pingResponse.master() != null && !localNode.equals(pingResponse.master())) {
             activeMasters.add(pingResponse.master());
         }
     }
   ```

3. 遍历fullPingResponses：获取有master资格的节点，加入masterCandidates

4. 如果activeMasters非空，则排序选主

5. 如果activeMasters为空，则从masterCandidates中选主。首先判断节点数是否>=minimum_master_nodes，满足则排序选主
   排序：集群中每个节点对所有 master候选节点（node.master: true）根据 clusterState（从大到小）和nodeId（从小到大） 进行排序，然后选出第一个节点

6. 如果对某个节点的投票数达到阈值，并且该节点自己也选举自己，那这个节点就是master；否则重新选举一直到满足上述条件。

![img](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/v2-a5272b335bc7d2b9ee5cf60f4a4ce215_720w.webp)

缺点：discovery.zen.minimum_master_nodes这个很重要，但是动态扩展的时候有些时候可能会忘记设置

**es7之后**

Raft算法：

1. 初始都是follower
2. 发起者term+1，并给自己投票，然后给其他节点发送投票请求
3. 其他节点发现自己term小，就投票，然后更新term
4. 发起者收到大多数投票成为leader

类Raft算法

1. 初始化状态都是候选者
2. 给其他节点发送PRE_VOTE投票请求，不投给自己
3. 其他节点对请求进行处理：如果term比自己大，退化成follower
4. 如果收到足够多的投票，就发起startJoin请求，在请求中会将term+1，自己不更新。
5. 首次收到join请求会模拟一个join给自己，term+1。如果收到大多数join请求，即可成为leader。
6. 如果又收到了其他节点的拉票请求，就退出leader状态并投票，转为Candidate

#### 分片



### 进阶

#### 索引文档过程



![image-20220611183624506](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20220611183624506.png)

1. 客户端向Node1发送写请求
2. 节点根据文档id确定属于分片P0，请求被转发到Node3。确定属于哪个分片：shard = hash(document_id) % (num_of_primary_shards)
3. Node3在主分片执行写操作，成功之后，会将请求并发转到副本分片上。所有副本分片报告成功后，Node3向协调节点报告成功，协调节点向客户端报告成功



#### 存储原理

![image-20220619224328716](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20220619224328716.png)

- 主分片先将数据写入Memory Buffer，然后定时（refresh_interval，默认1s）写入文件系统缓存，称为一个segment，这个过程叫refresh，每个 segment 文件实际上是一些倒排索引的集合，此时可被检索
- 写入内存的同时写入translog中（保存未持久化的操作日志）。也是先写文件系统缓存，每5s刷盘，如果机器挂了，会丢5s数据。也可以将 translog 设置成每次写操作必须是直接 fsync 到磁盘，但是性能会差很多。
- flush：首先将内存buffer中的数据refresh到文件系统缓存，清空buffer。然后将一个commit point写入磁盘，标识这个commit point对应的所有segment file。同时文件系统缓存的数据写入磁盘并清空translog。触发时机：每30min或translog达到512M



#### 搜索过程

搜索被执行成一个两阶段过程，即 Query Then Fetch：

Query

1. 客户端发送请求到协调节点
2. 协调节点将请求转发到所有分片，每个分片在本地执行搜索并构建一个匹配文档的大小为 from + size 的优先队列，返回docId和打分值给协调节点
3. 协调节点对所有的docId排序聚合

Fetch

1. 协调节点根据docId查询对应的分片获取文档内容，返回给客户端

#### 更新、删除过程

更新和删除都是写操作，但es的文档是不可变的。

磁盘的每个段都有一个.del文件：

删除：会在.del文件中标记为删除。文档依然能被查询到，但是会在结果中进行过滤。当段合并时，在.del文件中的文档不会被写入新段

更新：创建新文档，分配一个新版本号，旧版本在.del中被标记为删除，查询时会过滤

#### merge

每秒产生一个segment，会越来越多，每个segment会消耗文件句柄、内存和CPU，且每次search要扫描所有的segment，效率低

ES有一个后台进程定期merge segment，将多个小的segment合并，被标记为删除的文档不会写入新的segment。合并完成后flush写入磁盘，新段打开用于搜索，旧段被删除，并创建一个新的commit point

#### 读写一致性



#### 集群状态

- Green：所有主分片和从分片都准备就绪（分配成功），即使有一台机器挂了（假设一台机器一个实例），数据都不会丢失，但会变成 Yellow 状态。
- Yellow：所有主分片准备就绪，但存在至少一个主分片（假设是 A）对应的从分片没有就绪，此时集群属于警告状态，意味着集群高可用和容灾能力下降，如果刚好 A 所在的机器挂了，而从分片还处于未就绪状态，那么 A 的数据就会丢失（查询结果不完整），此时集群进入 Red 状态。
- Red：至少有一个主分片没有就绪（直接原因是找不到对应的从分片成为新的主分片），此时查询的结果会出现数据丢失（不完整）。

#### Scroll

非实时大量数据的场景，不能指定页

- 第一次请求：缓存所有符合条件的数据，生成快照，并返回scroll_id
- 后续请求：带上scroll_id，继续读取下一页

#### Search_after

实时，根据上一页的最后一条数据确定下一页的位置。为了找到每一页最后一条数据，每个文档必须有一个全局唯一值




### 实际应用

总共7个节点，4个数据节点，3个主节点。

索引按日期创建，每个索引有10个分片。每周新增百万级

#### 调优

1. 按日期创建索引
2. 采取bulk批量写入
3. 类型使用keyword精确查询，text会分词
4. 如果搜索结果不需要近实时性，可以把每个索引的 index.refresh_interval 改到30s
5. 如果是大批量导入，可以设置 index.number_of_replicas: 0 关闭副本，等数据导入完成之后再开启副本
6. 段和合并：Elasticsearch 默认值是 20 MB/s。但如果用的是 SSD，可以考虑提高到 100–200 MB/s。如果你在做批量导入，完全不在意搜索，你可以彻底关掉合并限流。
7. JVM参数调优

#### 慢查询

1. 深度分页效率慢，超过1w条报错：es采用from+size进行分页，先从各分片查询from+size条数据，协调节点排序后再去对应分片查询，最终取size条数据。
2. Filter 比 Query 效率好， Filter 查询子句不需要计算相关性的分值，并且查询结果可以缓存
3. 查询范围过大，涉及太多分片；分片不均匀、单个分片过大
5. keyword比Integer性能更好，因为keyword使用的是倒排索引，Integer是KV树
7. 段文件过多，需要定时merge
8. mapping设计不合理

