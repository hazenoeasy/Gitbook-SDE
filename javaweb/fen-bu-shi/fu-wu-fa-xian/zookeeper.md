---
description: 文件系统·+通知机制
---

# Zookeeper



CAP 理论指出对于一个分布式计算系统来说，不可能同时满足以下三点：

* **一致性**：在分布式环境中，一致性是指数据在多个副本之间是否能够保持一致的特性，等同于所有节点访问同一份最新的数据副本。在一致性的需求下，当一个系统在数据一致的状态下执行更新操作后，应该保证系统的数据仍然处于一致的状态。
* **可用性：**每次请求都能获取到正确的响应，但是不保证获取的数据为最新数据。
* **分区容错性：**分布式系统在遇到任何网络分区故障的时候，仍然需要能够保证对外提供满足一致性和可用性的服务，除非是整个网络环境都发生了故障。



基于观察者模式设计的分布式服务管理框架，它负责存储和管理大家都关心的数据，然后接受观察者的注册，一旦这些数据的状态发生变化，Zookeeper就将负责通知已经在Zookeeper上注册的那些观察者作出相应的反应。

![](<../../../.gitbook/assets/Screen Shot 2022-02-27 at 10.46.01 PM.png>)

1. 一个leader 多个 follower 组成的集群， leader负责写数据，follower负责读数据
2. 只要有半数以上节点存活，cluster就能正常服务。
3. 全局数据一致，每个server保存一份相同的数据副本
4. 更新请求顺序执行，来自同一个client的请求按发送顺序执行
5. 数据更新原子性，一次更新要么成功，要么失败
6. 实时性，在一定时间范围内，client能读到最新数据

#### 数据结构

每个节点成为ZNode，每个Znode默认能够存储1MB的数据，每个ZNode都可以通过其路径唯一标识

![](<../../../.gitbook/assets/Screen Shot 2022-02-27 at 10.52.09 PM.png>)

不适合存储大数据，只适合存储配置信息。

#### 应用场景

统一命名服务，统一配置管理，统一集群管理，服务器结点动态上下线，软负载均衡

* 统一命名服务： 服务器 IP 和域名的关系
* 统一配置管理： 将配置信息写入Zookeeper的一个ZNode，各个服务端服务器监听这个ZNode
* 统一集群管理：分布式环境中，实时掌握每个节点的状态是必要的，Zookeeper可以将节点信息写入ZNode，并且监控ZNode获得它的实时状态变化。
* 服务器结点动态上下线
* 软负载均衡

#### 参数说明

```
# The number of milliseconds of each tick  心跳时间 服务端和客户端心跳时间
tickTime=2000 
# The number of ticks that the initial 
# synchronization phase can take leader和follower初始化，十次心跳， 十次失败，就真的失败了
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement leader和follower通信，五次心跳， 五次失败，就真的失败了
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes. 默认tmp 会被系统定期删除，所以一般不用tmp
dataDir=/opt/homebrew/var/run/zookeeper/data
# the port at which the clients will connect 默认端口号
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1

## Metrics Providers
#
# https://prometheus.io Metrics Exporter
#metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpPort=7000
#metricsProvider.exportJvmInfo=true

```

#### 选举机制&#x20;

* 第一次启动  比较Myid

![](<../../../.gitbook/assets/Screen Shot 2022-02-27 at 11.30.25 PM.png>)

*   SID：服务器ID。用来唯一标识一台 ZooKeeper集群中的机器，每台机器不能重复，和 `myid` 一致。

    编号越大在选举算法中的权重越大。
* epoch：epoch可以认为是Leader编号，每一次重新选举出一个新Leader时，都会为该Leader分配一个epoch，该值也是一个递增的，可以防止旧Leader活过来后继续广播之前旧提议造成状态不一致问题，只有当前Leader的提议才会被Follower处理。Leader没有进行选举期间，epoch是一致不会变化的。
* counter：ZooKeeper状态的每一次改变, counter就会递增加1.
* zxid=epoch+counter，其中epoch不会改变，counter每次递增1，,这样zxid就具有递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生。
