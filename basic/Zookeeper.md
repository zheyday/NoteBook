# 1、分布式架构

## 1.2.3、CAP理论

**一致性Consistency**

更新操作成功并返回客户端后，所有节点在同一时间的数据完全一致。对于客户端来说，一致性指的是并发访问时更新过的数据如何获取的问题。从服务端来看，则是更新如何复制分布到整个系统，以保证数据最终一致。

**可用性Availability**

请求都能得到响应数据，不会出现响应错误。换句话说，可用性是站在分布式系统的角度，对访问本系统的客户的另一种承诺：我一定会给您返回数据，但不保证数据最新，强调的是不出错。

**分区容错性Partition tolerance**

在分布式系统中，节点应该是连通的。但是因为故障使得一些节点不连通了，从而导致整个网络分为了若干个区域，这就叫网络分区。

分区容错性要求分布式系统在遇到网络分区故障时，仍能对外提供满足CA的服务。

对于一个分布式系统而言，P是前提，必须保证