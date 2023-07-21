## CAP

#### 概念

C：Consistency 一致性，客户端的读操作，要么就最新数据，要么读取失败，强调数据正确性

A：Availability 可用性，任何请求都能得到响应，无论是不是最新的数据，强调不出错

P：Partition Tolerance 分区容错性，分布式系统在遇到某节点或网络分区故障的时候，仍然能够对外提供满足一致性或可用性的服务。强调系统不会挂掉

## Nacos



#### 比较

Zookeeper保证CP

## Zuul



## Ribbon

基于HTTP和TCP的客户端负载均衡工具



#### 常用负载均衡策略

![image-20220918234341604](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20220918234341604.png)



## Feign

在Ribbon基础上进一步改进，采用接口方式，将服务端方法定义成一模一样的抽象方法，不需要自己构建http请求

代码见feign-consumer

# Hystrix

### 降级



### 熔断



### 监控

![image-20220922095818744](https://raw.githubusercontent.com/zheyday/blog-picture-bed/main/image-20220922095818744.png)

### 依赖隔离





## 消息驱动