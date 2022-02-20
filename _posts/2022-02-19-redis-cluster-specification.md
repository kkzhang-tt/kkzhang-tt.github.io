---

layout: post
title: "Redis Cluster Specification"
category: "system-design"
author: "kkzhang"
---

# 1-Main properties and rationales of the design

## 1-1 **Redis Cluster goals**

Redis Cluster 是 Redis 的分布式实现，按照优先级有以下目标：

- **高性能**（high performance）
- 支持**线性扩展**（linear scalability）至 1000 个节点
- 没有代理；集群节点间通过**异步复制数据**；不支持数据合并操作
- 可接受的写入安全：在网络分区情况下，系统尽可能保存访问到多数派分区的 Client 的写操作。不过，存在一个小的时间窗口，在此期间的写入操作可能会丢失（*failover 前的写入可能会在 failover 过程中丢失*）。如果 Client 的写入操作连接到少数派分区，则这个丢失时间窗口会更大。
    
    > 网络分区时，集群节点被划分成多数派分区（majority），少数派分区（minority）
     
- 可用性：Redis Cluster 在**大部分 master 节点可用，并且对少部分不可用的 master，每一个 master 至少有一个当前可用的 slave** 场景下能够保证集群的可用性。
    
    另外，通过使用 **replicas migration** 技术，当前没有 slave 的 master 会从当前拥有多个 slave 的 master 接受到一个新 slave 来确保可用性。
    

## 1-2 ****Implemented subset****

Redis Cluster 实现了所有在非分布式 Redis 版本中的单 key 命令；但是对于使用多个 key 的复杂操作没有实现，比如 set 中的 unions & intersections 操作。

> 不支持多个 key 的复杂操作是为了避免 key-value 在不同的 Cluster 节点间移动，使得情况更加复杂 

不过，Redis Cluster 实现了被称为 **Hash Tags** 的概念：多个 key 可以通过相同的 hash tag 存储在相同的 hash slot 中，从而避免了 key 的迁移。

> 在手动 resharding 期间，多 key 操作可能变的不可用 

除了支持的命令不同，Redis Cluster 只支持 database 0，并不像单机版 Redis 支持多个数据库。

### 1-2-1 ****Why merge operations are avoided****

Redis Cluster 设计是避免在多个节点中存在相同 key-value 对的冲突版本，因为 Redis 中的值通常都是比较大的，数据类型也是语义复杂的，传输和合并这样的值将会影响性能。

## 1-3 ****Clients and Servers roles in the Redis Cluster protocol****

Cluster 节点主要任务有：

- 数据维护
- 集群状态获取
- 将 key 映射到正确的 Cluster 节点
- 自动发现其他 Cluster 节点
- 检测异常 Cluster 节点
- 当某个 master 节点故障时，提升其副本为 master 节点，以保证 Cluster 正常运行

为了实现上述功能，所有 Redis Cluster 节点间通过 **Redis Cluster Bus** 互相连接：由 TCP bus 及二进制协议组成。节点间通过 **Gossip 协议传递集群信息**，以此来实现新节点发现，节点探活及标定特定状态等功能。

由于 Cluster 节点不能代理请求，因此 Client 在收到重定向异常（MOVED, ASK）时，需要将请求重定向到其他节点。

> Client 通过缓存 key → cluster node 的映射关系，减少重定向，提高执行效率 

## 1-4 ****Write safety****

Redis Cluster 通过**节点间异步复制数据**，及 **last failover wins** 避免合并功能 。

> last failover wins 指当 master 节点故障时，通过 failover 机制选取的新 master 节点将直接覆盖之前 master 节点的数据，并同步给其他 slaves（新 master 节点中的数据可能不是最新的，或者有所缺失）。通过 last failover wins 机制，最后选举出的新 master 副本数据会覆盖其他所有副本数据。
> 

在网络分区期间，会存在一个写操作丢失的时间窗口。*对于发生在 master 节点多数派（majority）分区的写操作丢失窗口与少数派（minority）的丢失窗口是不同的*。

1. 对于发生在多数派 master 的写操作，Redis Cluster 会尽量保存；但是以下两种场景除外：
    1. client 写操作请求到达 master 节点，当 master 执行完成并回复 client 成功之后，master 出现异常而不可访问；但是之前的写操作并未通过异步复制到其他 slaves 中。如果 master 不可访问的时间较长而导致其中的一个 slave 被选举成新的 master，那么之前的写操作将会丢失。
    2. 由于网络分区，某个 master 不可被访问。网络分区触发了一轮选举，导致其中的一个 slave 被选举成新的 master。网络分区恢复之后，old master 变成 new master 的 slave 之前，一个 client 通过过期的路由表对 old master 节点进行写入，此时的写入将会被全部丢失。
    
    不过，对于第二种场景，在一些安全机制的条件下很难发生：
    
    - 少数派 master 与多数派 master 无法通信达到一定的时间后，将拒绝 client 的写操作请求
    - 当网络分区恢复后，该 master 仍需要继续拒绝写入一段时间用来感知 Cluster 的配置变化，因此留给 client 的时间窗口很小
    - 在分区恢复之后，其他节点会尽快尝试访问新加入的节点（携带最新的 Cluster 配置信息）
2. 对于发生在少数派 master 的写操作拥有更大的丢失窗口
    - 如果少数派 master 节点通过 failover 转移到多数派 master 节点的分区，那么所有发送到少数派分区的写操作都将会被永久丢失

**发生 failover 的前提是，其中的 master 节点至少在 NODE_TIMEOUT 时间内无法被多数派 master 节点访问**。

- 分区故障时间小于 NODE_TIMEOUT，则不会出现写操作数据丢失
- 分区故障时间大于 NODE_TIMEOUT，则对少数派 master 的写入操作将全部丢失
    
    > 不过少数派 master 会在进入不可用状态之后拒绝写入请求 

## 1-5 ****Availability****

**Redis Cluster 能容忍集群中少数节点不可访问，但不适合要求大量网络分块的应用**（如多机房部署）。

1. 集群在少数派分区侧不可用
2. 对于多数派 master 分区，如果其他每个不可访问的 master 节点都至少有一个 slave 节点可达，那么在经过 NODE_TIMEOUT 重新选举之后，多数派分区仍然可用

**为了提高集群可用性，Redis Cluster 支持 Replicas Migration：**自动将转移副本节点到孤立的 master 节点（不再拥有 slave 的 master）；每次 failover 成功之后，都会重新配置 slave 副本分布以提高下一次故障期间的可用性。

# 2-**Overview of Redis Cluster main components**

## 2-1 ****Keys distribution model****

**key 的空间范围被划分为 16834 个 slot**，间接使得一个集群的最大上限为 16834 个 master 节点。

> 一般建议最大节点数少于 1000
> 

Cluster 中的每个 master 节点处理 16834 个 hash slot 的其中一部分子集。当 Cluster 处于稳定状态时，每个 hash slot 只会由一个节点提供服务。

> 没有出现 slot 迁移的情况被认为是稳定状态
> 

将 key 映射为对应的 hash slot 方法为：

```c
HASH_SLOT = CRC16(key) mod 16384
```

## 2-2 ****Keys hash tags****

**Hash Tags 提供了一种将多个 key 分配到同一个 hash slot 的方式**。通过 Hash Tags 可以在 Redis Cluster 中实现对多个 key 的同时操作。

Hash Tags 的规则如下：

- key 包含一个 `{` 字符
- *并且* 如果在这个`{`的右面有一个`}`字符
- *并且* 如果在`{`和`}`之间存在至少一个字符

那么，`{ }` 之间的字符将被用来计算 hash slot。如 `{user1000}.following`和`{user1000}.followers`这两个 key 会被分配到相同的 hash slot 中，因为只有`user1000`会被用来计算 hash slot 值。

## 2-3 ****Cluster nodes attributes****

集群中的**每个节点都有全局唯一 ID 标识**。启动时生成，并持久化在配置文件中，一般不会改变。

每个节点还维护集群中其他节点的信息，包括：

- node id
- ip & port
- 标签
- master node id（如果节点是 slave）
- 最后一次被挂起的 ping 的发送时间 & 最后一次收到 pong 的时间
- 该节点的当前 **configuration epoch**
- 该节点维护的 hash slots

## 2-4 The Cluster bus

每个集群节点使用额外的 TCP 端口用于与集群中的其他节点交互；集群节点间的交互只使用 Cluster bus 及 Cluster bus 协议：一种二进制协议。

## 2-5 ****Cluster topology****

Redis Cluster 是全网拓扑，**每个节点都与其他节点维护 TCP 连接**。

> N 个节点的集群中，每个节点有 N-1 个传出 TCP 连接，同时有 N-1 个传入 TCP 连接 

集群节点间使用 **Gossip 协议**和**配置更新机制**来避免正常情况下节点间交互过多的消息。

## 2-6 ****Nodes handshake****

对于 Cluster bus port 连接，节点总是接受并回复 ping 请求，即使该 ping 请求来自一个不可信任的节点。但是如果发送节点被认为不是集群的一部分，那么该节点的其他数据包都会被丢弃。

通过两种方式可以判断一个节点是不是集群节点：

1. **节点出现在 MEET 消息中**：MEET 消息会强制接收者接受一个节点作为集群的一部分
    
    ```c
    CLUSTER MEET ip port
    ```
    
2. **节点出现在一个被信任的节点的 Gossip 消息中**：A 节点是被信任的集群节点，B 出现在 A 的Gossip 消息中，那么 C 收到 A 的消息后也会把 B 标记为集群节点

一旦我们将某个节点加入了连接图中，那么最终所有节点会自动形成一张全连接图（fully connected graph），即**集群可以自动发现其他节点**。

> 该机制使得集群更加健壮，可以防止不同的 Cluster 在 IP 地址变更或者其他网络相关事件导致意外混合 

# 3-**Redirection and resharding**