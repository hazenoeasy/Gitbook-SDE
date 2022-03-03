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

### Zookeeper Session 基本原理

#### Session 的创建

**sessionID**: 会话ID，用来唯一标识一个会话，每次客户端创建会话的时候，zookeeper 都会为其分配一个全局唯一的 sessionID。

**Timeout**：会话超时时间。客户端在构造 Zookeeper 实例时候，向服务端发送配置的超时时间，server 端会根据自己的超时时间限制最终确认会话的超时时间。

**TickTime**：下次会话超时时间点，默认 2000 毫秒。可在 zoo.cfg 配置文件中配置，便于 server 端对 session 会话实行**分桶策略管理**。

**isClosing**：该属性标记一个会话是否已经被关闭，当 server 端检测到会话已经超时失效，该会话标记为"已关闭"，不再处理该会话的新请求。

#### Session 的状态

下面介绍几个重要的状态：

* **connecting**：连接中，session 一旦建立，状态就是 connecting 状态，时间很短。
* **connected**：已连接，连接成功之后的状态。
* **closed**：已关闭，发生在 session 过期，一般由于网络故障客户端重连失败，服务器宕机或者客户端主动断开。

![](<../../../.gitbook/assets/image (1).png>)

#### 会话超时管理（分桶策略+会话激活）

zookeeper 的 leader 服务器再运行期间定时进行会话超时检查，时间间隔是 ExpirationInterval，单位是毫秒，默认值是 tickTime，每隔 tickTime 进行一次会话超时检查。

在 zookeeper 运行过程中，客户端会在会话超时过期范围内向服务器发送请求（包括读和写）或者 ping 请求，俗称**心跳检测**完成会话激活，从而来保持会话的有效性。

### Zookeeper 权限控制 ACL

### Zookeeper watcher 事件机制原理剖析

zookeeper 的 watcher 机制，可以分为四个过程：

* 客户端注册 watcher。
* 服务端处理 watcher。
* 服务端触发 watcher 事件。
* 客户端回调 watcher。

![](<../../../.gitbook/assets/Screen Shot 2022-02-28 at 12.52.45 AM.png>)

### &#x20;Zookeeper 数据同步流程

在 Zookeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性。

ZAB 协议分为两部分：

* 消息广播
* 崩溃恢复

#### 消息广播

Zookeeper 使用单一的主进程 Leader 来接收和处理客户端所有事务请求，并采用 ZAB 协议的原子广播协议，将事务请求以 Proposal 提议广播到所有 Follower 节点，当集群中有过半的Follower 服务器进行正确的 ACK 反馈，那么Leader就会再次向所有的 Follower 服务器发送commit 消息，将此次提案进行提交。这个过程可以简称为 2pc 事务提交，整个流程可以参考下图，注意 Observer 节点只负责同步 Leader 数据，不参与 2PC 数据同步过程。

![](<../../../.gitbook/assets/image (3).png>)

#### 崩溃恢复

在正常情况消息广播情况下能运行良好，但是一旦 Leader 服务器出现崩溃，或者由于网络原理导致 Leader 服务器失去了与过半 Follower 的通信，那么就会进入崩溃恢复模式，需要选举出一个新的 Leader 服务器。在这个过程中可能会出现两种数据不一致性的隐患，需要 ZAB 协议的特性进行避免。

* 1、Leader 服务器将消息 commit 发出后，立即崩溃
* 2、Leader 服务器刚提出 proposal 后，立即崩溃

ZAB 协议的恢复模式使用了以下策略：

* 1、选举 zxid 最大的节点作为新的 leader
* 2、新 leader 将事务日志中尚未提交的消息进行处理

#### 选举机制&#x20;



*   SID：服务器ID。用来唯一标识一台 ZooKeeper集群中的机器，每台机器不能重复，和 `myid` 一致。

    编号越大在选举算法中的权重越大。
* epoch  逻辑时钟：epoch可以认为是Leader编号，每一次重新选举出一个新Leader时，都会为该Leader分配一个epoch，该值也是一个递增的，可以防止旧Leader活过来后继续广播之前旧提议造成状态不一致问题，只有当前Leader的提议才会被Follower处理。Leader没有进行选举期间，epoch是一致不会变化的。
* counter：ZooKeeper状态的每一次改变, counter就会递增加1.
* zxid=epoch+counter，其中epoch不会改变，counter每次递增1，,这样zxid就具有递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生。

**选举状态：**

* **LOOKING**: 竞选状态
* **FOLLOWING**: 随从状态，同步 leader 状态，参与投票
* **OBSERVING**: 观察状态，同步 leader 状态，不参与投票
* **LEADING**: 领导者状态

![
](<../../../.gitbook/assets/image (4).png>)

![](<../../../.gitbook/assets/Screen Shot 2022-02-27 at 11.30.25 PM.png>)



#### 1、服务器启动时的 leader 选举

每个节点启动的时候都 LOOKING 观望状态，接下来就开始进行选举主流程。这里选取三台机器组成的集群为例。第一台服务器 server1启动时，无法进行 leader 选举，当第二台服务器 server2 启动时，两台机器可以相互通信，进入 leader 选举过程。

* （1）每台 server 发出一个投票，由于是初始情况，server1 和 server2 都将自己作为 leader 服务器进行投票，每次投票包含所推举的服务器myid、zxid、epoch，使用（myid，zxid）表示，此时 server1 投票为（1,0），server2 投票为（2,0），然后将各自投票发送给集群中其他机器。
* （2）接收来自各个服务器的投票。集群中的每个服务器收到投票后，首先判断该投票的有效性，如检查是否是本轮投票（epoch）、是否来自 LOOKING 状态的服务器。
* （3）分别处理投票。针对每一次投票，服务器都需要将其他服务器的投票和自己的投票进行对比，对比规则如下：
  * a. 优先比较 epoch
  * b. 检查 zxid，zxid 比较大的服务器优先作为 leader
  * c. 如果 zxid 相同，那么就比较 myid，myid 较大的服务器作为 leader 服务器
* （4）统计投票。每次投票后，服务器统计投票信息，判断是都有过半机器接收到相同的投票信息。server1、server2 都统计出集群中有两台机器接受了（2,0）的投票信息，此时已经选出了 server2 为 leader 节点。
* （5）改变服务器状态。一旦确定了 leader，每个服务器响应更新自己的状态，如果是 follower，那么就变更为 FOLLOWING，如果是 Leader，变更为 LEADING。此时 server3继续启动，直接加入变更自己为 FOLLOWING。

#### 2、运行过程中的 leader 选举

当集群中 leader 服务器出现宕机或者不可用情况时，整个集群无法对外提供服务，进入新一轮的 leader 选举。

* （1）变更状态。leader 挂后，其他非 Oberver服务器将自身服务器状态变更为 LOOKING。
* （2）每个 server 发出一个投票。在运行期间，每个服务器上 zxid 可能不同。
* （3）处理投票。规则同启动过程。
* （4）统计投票。与启动过程相同。
* （5）改变服务器状态。与启动过程相同。

### &#x20;Zookeeper 分布式锁实现原理

#### 排他锁

排他锁（Exclusive Locks），又被称为写锁或独占锁，如果事务T1对数据对象O1加上排他锁，那么整个加锁期间，只允许事务T1对O1进行读取和更新操作，其他任何事务都不能进行读或写。

#### 共享锁

共享锁（Shared Locks），又称读锁。如果事务T1对数据对象O1加上了共享锁，那么当前事务只能对O1进行读取操作，其他事务也只能对这个数据对象加共享锁，直到该数据对象上的所有共享锁都释放。
