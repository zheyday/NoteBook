---
title: redis学习
tags: redis
---

# 什么是redis



## 为什么快

- 单进程单线程（6.0之前）
- 基于内存
- 多路复用I/O

## 线程模型

基于Reactor实现的文件事件处理器，是单线程的，采用多路复用IO机制同时监听多个socket

## 数据类型

- String：key-value结构，一个键最大能存储512MB。微博数、粉丝数。
- list：双向链表。微博关注列表、粉丝列表、消息列表。
- hash：适合存储对象。存储用户信息。
- set：无序集合，集合成员是唯一的。共同关注、共同粉丝。
- zset：有序集合，比set多了权重参数score。
- bitmap
- hyperloglog
- Geo

## 编码

```sql
object encoding key
```

| 数据类型 | 编码                |
| -------- | ------------------- |
| String   | int、embstr、raw    |
| list     | ziplist、linkedlist |
| hash     | ziplist、字典       |
| set      | intset、字典        |
| zset     | ziplist、skiplist   |

- String：
  - int：整数值并且可以用long表示。当执行一些命令(append)使得这个值不再是整数值的时候会变为raw
  - embstr：长度小于等于32字节，一次内存分配。只读类型，修改时会转换为raw
  - raw：其他，两次内存分配
- list：当所有字符串元素的长度小于64字节、数量小于512时使用ziplist

## 数据结构

### SDS

用途：保存字符串值、AOF缓冲区、客户端状态中的输入缓冲区

sdshdr结构（Simple Dynamic Strings Header）表示SDS值，有sdshdr5、sdshdr8、sdshdr16、sdshdr32、sdshdr64

![image-20220114232045617](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20220114232045617.png)



![image-20200625103317894](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200625103317894.png)

redis使用SDS来代替C语言中的字符串，优点：

- 常数复杂度获取字符串长度
- 当SDS需要修改时，会先检查空间是否足够，防止缓冲区溢出
- 通过空间预分配和惰性空间释放来减少修改字符串带来的内存重分配次数
  - 空间预分配：当修改SDS并且需要空间扩展时，程序会分配额外的未使用空间。如果SDS修改之后，长度小于1MB，那么分配和len属性一样的free空间，实际长度为len+free+1byte(free=len，1为空字符)，否则分配1MB的free空间，实际长度为len+1MB+1byte
  - 惰性空间释放：当SDS需要缩短时，不立即进行内存重分配来回收多余的字节，而是使用free属性记录
- 二进制安全：c字符串通过空字符判断结束，会有问题
- 可以使用部分<string.h>库中的函数：比如strcasecmp、strcat

### 链表



### 压缩列表

将数据按照一定规则编码在一块连续的内存区域，以节省内存。

![image-20200702171149982](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200702171149982.png)

![image-20200702171211275](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200702171211275.png)

- previous_entry_length：前一个节点的长度，1个或5个字节。如果前一个节点的长度小于254，则为1个字节，存储的就是长度；如果大于等于254，则第一个字节的值为254，后四个字节表示长度
- encoding：表示content的内容类型和长度
- content：保存节点的内容

### 字典

Redis的数据库、hash键都是使用字典实现

Redis定义了dictEntry，dictType，dictht和dict四个结构体来实现字典结构

dictEntry：hash表节点，保存键值对

<img src="https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200702142116493.png" alt="image-20200702142116493" style="zoom:67%;" />

hash表由dict.h/dictht结构表示

<img src="https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200623133155966.png" alt="image-20200623133155966" style="zoom:67%;" />

字典由dict.h/dict结构表示，其中包含了两个hash表和rehashidx变量。一般情况下字典只使用ht[0]哈希表，ht[1]只会在对ht[0]进行rehash时使用

![image-20200623132716693](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200623132716693.png)

#### rehash

1. 扩展：将ht[1]的大小设为第一个大于等于`ht[0].used*2`的2^n^，
   收缩：将ht[1]的大小设为第一个大于等于`ht[0].used`的2^n^，
2. 将ht[0]的键值rehash到ht[1]
3. 释放ht[0]，将ht[1]设为ht[0]，然后为ht[1]分配一个空白哈希表

#### 渐进式rehash

如果哈希表键值对数量过多，一次性hash的话会导致服务器无响应，所以采用渐进式rehash

1. 为ht[1]分配空间，让字典同时持有ht[0]和ht[1]
2. 将rehashidx设为0，表示rehash开始
3. 在rehash进行期间，每次对字典执行CRUD操作时，服务器会顺带将ht[0]在rehashidx上的所有键值对rehash到 ht[1]，然后将rehashidx加1
4. 完成之后将rehashidx设为-1

#### 扩展与收缩

负载因子 = 哈希表已保存节点数量 / 哈希表大小

- 服务器没有执行BGSAVE或BGREWRITEAOF命令，且负载因子>=1，执行扩展
- 服务器在执行BGSAVE或BGREWRITEAOF命令，且负载因子>=5，执行扩展
- 负载因子<=0.1时，执行收缩

#### 为什么执行BGSAVE或BGREWRITEAOF命令时负载因子>=5

首先服务器进程在执行BGSAVE或者BGREWRITEAOF命令时，创建了新的子进程，子进程和父进程共享内存空间。此时如果我们扩展哈希表，那么那么相当于往父进程写入数据，同时会导致子进程进行复制操作，因为复制是按页来进行复制，所以，如果在执行上述命令时大量写入，则会出现大量页中断，导致大量复制操作，影响系统性能。



### 跳跃表

平均时间复杂度 O(logN)，最坏O(N)，是zset的底层实现之一

zskiplist结构保存相关信息，下图最左边的

- header：指向头节点，头节点是个什么都不存储的dummy节点，层数为32
- tail：指向尾结点
- level：最大层数
- length：长度

zskiplistNode结构表示跳跃表节点

- 层：每个层包括前进指针和跨度。前进指针指向下一个同层节点，跨度表示下一个节点和当前节点的距离，用来查询排名。层数是随机产生的
- BW：后退指针
- score：分值，可以相同
- obj：成员对象，指向一个字符串对象，字符串对象保存着一个SDS值，必须唯一。当score相同时，按照成员对象在字典序中的大小来排名

![image-20200702143701781](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200702143701781.png)

### zset实现

zset对象编码由ziplist和skiplist组成。当元素数量小于128并且所有member的长度都小于64字节时使用ziplist，其余情况使用skiplist+字典

skiplist的每个节点会随机一个层数。比如是3，那么就把它放入1-3层这三层链表中。字典存储了member-score的映射关系

### 对象



## 过期键删除策略

```yml
# 设置生存时间 秒/毫秒
expire/pexpire key seconds
# 过期时间 是unix时间戳
expireat/pexpireat key unix_time
# 显示剩余生存时间
ttl/pttl key

expire、pexpire、pexpireat
```

![1614867584437](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/1614867584437.png)

- 惰性删除：每次取键时检查，如果过期则删除。节省CPU资源，占用大量内存
- 定时删除：在设置键的过期时间的同时设置一个定时器，在键的过期时间到来时进行删除
- 定期删除：redis会将设置过期时间的key放到一个字典中，默认100ms随机抽取一些key，如果过期就删除。具体：从字典随机20个key，删除过期的key，如果比例超过1/4，则重复

Redis采用定期删除+惰性删除

## 淘汰策略

volatile-lru：从已设置过期的数据集中挑选最久未使用的淘汰

volatile-ttl：从已设置过期的数据集中挑选将要过期的数据淘汰

volatile-random：从已设置过期的数据集中任意挑选数据淘汰

allkeys-lru：从数据集中挑选最久未使用的数据淘汰

allkeys-random：从数据集中任意挑选数据淘汰

noeviction：禁止淘汰数据

## 数据库底层实现

```c++
struct redisServer {
    // 一个数组，保存着服务器中的所有数据库
    redisDb *db;
    // 数据库数量
    int dbnum;
};
typedef struct redisDb {
    // 数据库键空间，保存着数据库中所有键值对
    dict *dict;
    / ...
    // 过期字典，保存着键的过期时间
    dict *expires;
} redisDb;
```

![1614845760671](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/1614845760671.png)



## LRU

### HashMap和双向链表实现

HashMap的Key存储key，Value指向Node节点，保证save和get key的时间都是O(1)

首先预设LRU容量，如果满了，淘汰掉尾节点；每次新增和访问数据，把节点新增或移动到头部

<img src="https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/v2-09f037608b1b2de70b52d1312ef3b307_r.jpg" alt="preview" style="zoom:67%;" />

### Redis中LRU实现

上述实现需要额外空间存放双向链表中的next和pre指针，所以redis采用近似LRU算法

1. redis有一个全局时钟`lruclock`
2. 每个object有一个24bit的字段，用来记录最后一次操作的时间戳，在初始化和操作的时候会得到全局`lruclock`来更新自己的`lruclock`。
3. 如果设置了`maxmemory`，每次操作都要判断是否需要淘汰。随机选择5个(maxmemory-samples值)key，根据全局时钟和key 的内部时钟的差值由小到大排序，放入到**淘汰池**中，释放掉最后一个key

## 持久化

### RDB(默认)

SAVE、BGSAVE

SAVE：服务器会被阻塞

按照一定的时间间隔生成数据集的时间点快照，实际操作过程是fork一个子进程，先将数据集写入临时文件，写入成功后，再替换之前的文件，用二进制压缩存储。

优点：

1. 整个Redis数据库只有一个文件，非常适合用于备份和灾难恢复
2. 性能最大化。Redis服务在开始持久化时，只需要fork出一个子进程，然后子进程会处理持久化工作，父进程无需执行任何磁盘IO操作
4. 在恢复大数据集时比AOF快

缺点：

1. 在需要保证数据的高可用性，即尽量避免服务器故障时丢失数据的场合不适用。因为在定时持久化之前的数据都不能保存
2. Redis要fork出一个子进完成持久化，如果数据集过大，会阻塞服务器，写入磁盘的时候会影响性能

### AOF

将所有的命令行以 Redis 协议的格式来保存，新命令会被追加到文件的末尾。在服务器启动时，通过重新执行这些命令来还原数据集

优点：

1. 更高的数据安全性，即数据持久性。Redis提供了三种同步策略：每秒同步、每修改同步和不同步。默认是每秒同步，也是异步完成，效率非常高。
2. 通过追加模式写入文件，出现宕机也不会破坏日志已经存在的内容。如果写入了不完整的数据，`redis-check-aof` 工具也可以修复这种问题。
3. 如果文件过大，Redis会自动在后台启动**rewrite**机制，产生一个更小的AOF文件：在执行bgrewriteaof命令时，redis服务器会维护一个AOF重写缓冲区，该缓冲区会在子进程创建新文件期间，记录服务器执行的所有写命令。当子进程创建完新文件后，服务器会将缓冲区的内容追加到新文件末尾，然后替换旧文件

缺点：

1. AOF文件通常大于RDB文件，且恢复大数据集时的速度慢
2. 根据同步策略的不同，AOF在运行效率上往往会慢于RDB。每秒同步策略的效率是比较高的，**同步禁用**策略的效率和RDB一样高效。

## 事务

- 如果队列中存在语法性错误，则正确命令会被执行，错误命令抛出异常，不会回滚
- 如果队列中存在命令性错误，则所有命令都不执行

## 分布式锁的

### 坑

#### 1 非原子操作

以前经常看到使用setnx+expire命令加锁，del解锁，但是在加锁和解锁时都是有问题的：

- 加锁时：这两个命令是非原子性的，如果执行完setnx后redis宕机，那么锁就永远无法释放。
- 解锁时：直接使用del的话会导致任何客户端都可以解锁

通常有两种方法

```lua
1、-- 加锁：set key value ex sec(秒) px mill(毫秒) nx
set name unique_value px 3000 nx

2、-- 通常使用Lua脚本来实现 KEYS[1]：锁名，ARGV[1]：唯一值，ARGV[2]：超时时间
if redis.call('setnx',KEYS[1],ARGV[1]) == 1 then
   	redis.call('expire',KEYS[1],ARGV[2])
    return 1
else
    return 0
end


if (redis.call('exists', KEYS[1]) == 0) then
    --如果不存在，加锁并设置过期时间
    redis.call('hset', KEYS[1], ARGV[2], 1); 
    redis.call('pexpire', KEYS[1], ARGV[1]); 
 	return nil; 
end
if (redis.call('hexists', KEYS[1], ARGV[2]) == 1)
    --可重入锁
   	redis.call('hincrby', KEYS[1], ARGV[2], 1); 
   	redis.call('pexpire', KEYS[1], ARGV[1]); 
  	return nil; 
end; 
return redis.call('pttl', KEYS[1]);


```

#### 2 没释放锁

在try...finally中手动释放锁。如果在释放锁的时候系统宕机，但是到了锁超时时间，也会自动释放锁

#### 3 释放别人的锁

```lua
--解锁：通过UUID生成唯一值，解锁时判断是不是同一个值 KEYS[1]：锁名，ARGV[2]：过期时间，ARGV[3]：唯一值
if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) 
then 
  --锁不存在
  return nil
end
--重入锁-1
local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1);
if (counter > 0) 
then 
    --重入锁数量>0，重设过期时间
    redis.call('pexpire', KEYS[1], ARGV[2]); 
    return 0; 
 else 
   --重入锁数量=0，直接删除锁
   redis.call('del', KEYS[1]); 
   redis.call('publish', KEYS[2], ARGV[1]); 
   return 1; 
end; 
return nil
```



### RedLock

使用多个Redis实例来实现分布式锁，保证高可用

1. 获取当前时间戳
2. 从N个实例中获取锁
3. 计算获取锁后的时间减去第一步的时间，如果差小于锁过期时间并且有一半以上的实例，则获取锁成功
4. 成功后，锁的真正有效时间是锁过期时间减去第三步的时间差
5. 如果失败，则开始解锁

### Zookeeper

- 创建一个目录/lock
- 客户端获取锁时，在/lock下创建临时的有序子节点
- 获取/lock下的子节点列表，判断自己创建的是否为列表中最小的子节点，如果是则获得锁，否则监听前一个节点，得到通知后重复此步骤直到获取锁
- 业务完成后，删除对应的子节点

## 问题

### 热key

某一热点key的请求量特别大，超过服务器性能极限

解决：

- 本地缓存，缺点是过期时间问题
- 热key加上后缀（随机数），将一个热key变为多个key分布到不同的主机上

### 大key

定义：

1. key过大
2. list、hash、set、zset的value数量过多

影响：

1. 删除时会造成服务器阻塞

解决：

1. 拆分，比如对json通过mset进行分散
2. 定期清理过期数据
3. Redis4.0使用新特性UNLINK命令启用新线程来删除大key，小于4.0使用SCAN命令

### 缓存击穿

某个热点缓存过期，大量请求这个key导致数据库崩溃

**解决**

使用互斥锁

### 缓存雪崩

1. redis挂掉了
2. 大量缓存失效，新缓存未到，导致原本需要访问缓存的请求都去查询数据库了，数据库压力过大，严重的话会宕机，最后造成整个系统崩溃

**解决**

1. 实现redis的高可用；通过持久化机制保存的数据快速恢复缓存
2. 避免缓存在同一时间过期

### 缓存穿透

查询一个一定不存在的数据，缓存层不命中，又去查询存储层，也不命中。如果有大量这样的请求，会对存储层造成较大压力

**解决**

1. 参数校验
2. 缓存空对象：需要大量的缓存空间，可以设置一个较短的过期时间；数据不一致，可以利用消息系统删除
3. 布隆过滤器：本质是一个bit向量，通过多个Hash函数将值映射到不同的bit中。查询时将值进行同样Hash，如果存在bit为0，那么一定不存在该值，如果都是1，则可能存在

## 架构

### 主从复制

主服务器将数据同步给从服务器，降低主服务器的读压力。slave启动时会发送PSYNC命令给master，如果是第一次连接会触发全量复制。master生成RDB快照，并缓存新的写请求。slave收到快照后写进本地磁盘并加载进内存，然后master将缓存发送给slave

无法保证高可用，没有解决主服务器的写压力。

### 哨兵

监控主从数据库；master出现异常时进行故障迁移；多哨兵之间互相监控

![image-20200409222246529](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/img/image-20200409222246529.png)

哨兵启动后，会与要监控的master建立两条连接：
1、订阅master的\_sentinel_:hello频道用于获取其他监控该master的哨兵节点信息
2、定期向master发送INFO等命令获取master信息

建立连接后，哨兵会执行三个操作：
1、定期向master和slave发送INFO命令，用于新节点的自动发现
2、定期向master和slave的\_sentinel_:hello频道发送自己的信息
3、定期向master、slave和其他哨兵发送ping命令，如果没有回复则进行**主观下线**处理。如果主观下线的是主节点，会咨询其他哨兵节点，如果超过半数，则进行**客观下线**处理

哨兵认为master客观下线时，通过Raft算法选举出领头哨兵进行故障恢复：
1、发现master下线的哨兵节点(A)向每个哨兵发送命令，要求对方选自己为领头
2、如果其他哨兵节点没有选过其他节点，则同意
3、如果超过一半同意，则A当选
4、如果有多个哨兵节点参选，则有可能无法选出，此时参选的节点等待一个随机时间后再次发起参选请求，直至选出

选出领头哨兵后，开始故障恢复：
1、选择所有在线的slave中优先级最高的
2、如有多个，则选择复制偏移量最大的(即复制越完整)
3、否则选取id最小的

为了避免单个哨兵出问题，一般采用哨兵集群。

优缺点：保证高可用，监控各个节点，自动故障迁移，但是没有解决主服务器写的压力

### 集群

Redis Cluter 实现了完全去中心化、线性扩展的集群方案。集群由多个节点组成，数据分布在节点中。节点分为主节点和从节点。提出哈希槽概念，包含16384个哈希槽，使用CRC16(key)%16384来计算key属于哪个槽

选举机制：当 slave发现自己的master变为FAIL状态时，便尝试 发起选举 ，以期成为新的 master。由于挂掉的master可能会有 多个 slave，从而存在多个slave竞争成为master节点的过程， 其过程如下：

1.slave发现自己的master变为FAIL

2.将自己记录的集群currentEpoch(选举轮次标记)加1，并广播信息给集群中其他节点

3.其他节点收到该信息，只有master响应，判断请求者的合法性，并发送结果

4.尝试选举的slave收集master返回的结果，收到 超过半数master的同意后变成新Master

5.广播Pong消息通知其他集群节点。

如果这次选举不成功，比如三个小的主从 A,B,C组成的集群，A的master挂了，A的两个小弟发起选举，结果B的master投给A的小弟A1，C的master投给了A的小弟A2，这样就会发起第二次选举，选举轮次标记+1继续上面的流程。事实上从节点并不是在主节点一进入 FAIL 状态就马上尝试发起选举，而是有一定延迟，一定的延迟确保我们等待FAIL状态在集群中传播，slave如果立即尝试选举，其它masters或许尚未意识到FAIL状态，可能会拒绝投票。 同时下面公式里面的随机数，也可以有效避免slave同时发起选举，导致的平票情况。DELAY = 500ms + random(0 ~ 500ms) + SLAVE_RANK * 1000ms

 

## 常用命令

```yml
object encoding|refcount|idletime

set key val

get key

del key

exists key

hset key field value

hget key field

```









