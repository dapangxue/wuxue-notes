# Redis

[TOC]

## 一、Redis集群

Redis集群有两种解决方案，一种是第三方插件Codis，另一种是Redis官方的解决方案Redis Cluster。

### 1.Codis

Codis是来自豌豆荚的中间件团队开发的一款Redis的集群方案。Codis先于Redis Cluster开发出来，在Redis Cluster广泛使用起来之前，Codis一直是Redis集群方案的主流。

Codis相当于一个代理，它接受客户端的指令并转发到后面的Redis服务器，客户端操纵Codis和操纵Redis没什么区别。

一个Codis所支撑的QPS有限，可以开启多个Codis实例，这样的话挂掉一个Codis代理也没关系。

#### Codis的分片方案

Codis需要将请求转发到特定的Redis实例，它将所有的key默认划分为1024个slot，将客户端传送过来的key进行CRC32运算得到一个哈希值，然后采用除留余数法对1024求余数，得出所在的槽位。

不同的Redis节点槽位关系的同步需要依靠Zookeeper，Coids将槽位关系存储在Zookeeper中，如果槽位发生了修改，Codis Proxy会监听到变化并重新同步槽位关系。

### 2.Redis Cluster

Redis Cluster是Redis官方的集群解决方案，它将所有的key缺省划分为16384个slot，通过

> slot = CRC16(key) & 16383

来计算key所存储的槽。在集群中，如果分配槽后有一个主节点不可用，那么整个集群将不可用，所以为了保证高可用，需要给每个主节点分配从节点。

推荐使用Ruby实现的Redis集群管理工具redis-trib.rb，它内部通过Cluster相关命令简化了集群创建、检查、槽迁移和均衡等常见运维操作。

Redis没有借助于zookeeper来同步节点的槽位关系，而是通过P2P的方式。它采用了Gossip协议，节点之前彼此通信不断交换信息，这样交换信息后的一段时间，所有节点的信息就共通了。

#### 集群扩容

当Redis的集群不能再支撑客户端的读取时，需要考虑集群的扩容操作，加入新的节点。

在准备新节点加入集群后，就需要开始着手迁移槽和数据了。需要先制迁移的方案，然后从源节点上拿出分配的槽给目标节点并迁移数据。

在槽重新划分完成后，本身槽属于哪个节点是无意义的，只要能将key对应到槽，那么去槽对应的节点取数据即可。

#### 集群收缩

集群收缩意味着需要安全下线一些节点，首先判断需要下线的节点是否有分配槽，如果有分配槽，需要将槽迁移，如果没有分配，通知其他节点忘记下线节点。

#### Redis Cluster怎么发现故障

Redis节点的Gossip协议（ping-pong）不但可以传输槽信息，还可以传播其他的状态，如主从节点、节点故障等。

主观下线：当某个主节点认为另一个主节点不可用时，这并不是最终的故障判断，是当前节点主观的判断，可能会误判。‘

集群中的主节点会定时的和其他节点发送ping消息，接受节点需要返回pong消息作为响应，如果在一个超时时间内通信一直失败，则判定该节点出了问题，标记pfail，此时不一定是真正的下线。

客观下线：标记一个节点真正的下线，集群内多个节点认为该节点不可用，判定该节点真正发生了故障，需要对故障节点进行数据迁移。

当半数以上的主节点标记故障节点主观下线时，触发客观下线流程，同时需要通知故障节点的从节点触发故障转移流程，这样保证了集群的高可用。

集群的故障恢复涉及到一个选举的过程，选举并不是通过从节点进行领导者选举的，而是通过主节点投票选举，只要获得拥有槽的主节点中超过半数的票，就可以晋升为主节点，替代原来的故障节点，如果从节点的投票一直没有超过半数，那么就继续选举。

## 二、主从复制

为了解决单点问题，所以将主节点的数据复制到多个从节点，满足故障恢复和负载均衡的需求。

复制可以分为一主一从和一主多从，其中一主多从还可以配置从从复制。

主从复制主要分为`增量同步`和`快照同步`。

### 增量同步

Redis内部维护了一个环形的buffer数组，主节点将修改性影响的指令缓存在buffer数组中，然后异步同步到从节点，从节点一边同步指令流，一边反馈一个偏移量，告诉主节点自己同步到哪了。

如果环形buffer过小，或者修改性指令缓存太快都会使从节点来不及同步，使得后面的指令会覆盖前面的指令。这个时候就是换一种同步的方式`快照同步`。

### 快照同步

快照同步同步的是主节点一个时刻的数据，它需要将当前主节点内存中的数据全部快照到磁盘文件中，然后将快照文件传送到从节点，从节点接收快照文件完毕后，删除当前内存中的全部数据，然后执行全量的加载。

从节点执行快照的同步时，主节点同时也在执行增量同步，如果快照同步消耗的时间太多，或者增量同步太快，使得后续的同步指令覆盖掉前面的数据，又会再次触发快照同步，极有可能陷入快照同步的死循环。

刚加入的从节点必须要进行一次快照同步，然后再进行增量同步。

### 无盘复制

主节点快照磁盘文件是非常重的操作，开销极大，所以主服务器通过套接字将快照内容发送到从节点，生成快照是主节点遍历数据的过程，从节点先接收主节点发来的快照内容，然后存储到磁盘文件中，最后进行一次全量加载。

以上的这种主从同步方式在主节点故障后，需要人工干预，不利于维护，所以引入的哨兵机制。

### 哨兵

<img src="./img/SentinelModel.png" weight="40%" height="40%"/>

Sentinel可以自动完成故障发现和转移，这种方案相比如Redis主从复制只是多了若干个Sentinel节点，客户端连接Redis服务器时先连接Sentinel节点，Sentinel节点返回当前的主节点给客户端连接，当主节点故障时，客户端会从新向Sentinel要地址。

Sentinel至少部署三个且奇数个Sentinel节点，节点多点为了提高故障判断准确性，奇数个节点是为了选举Sentinel领导者时过半的要求。

#### 主节点的选取方法

在Sentinel领导者选取好后，就开始负责从从节点中选出主节点，选取的原则就是先过滤掉下线的节点，选择从节点优先级高的节点，如果不存在，那么选取复制偏移量最大的从节点。