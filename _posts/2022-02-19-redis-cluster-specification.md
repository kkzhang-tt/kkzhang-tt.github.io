---

layout: post
title: "Redis Cluster Specification"
category: "system-design"
author: "kkzhang"
---

# 1-Main properties and rationales of the design

## 1-1 Redis Cluster goals

Redis Cluster 是 Redis 的分布式实现，按照优先级有以下目标：

- **高性能**（high performance）
- 支持**线性扩展**（linear scalability）至 1000 个节点
- 没有代理；集群节点间通过**异步复制数据**；不支持数据合并操作
- 可接受的写入安全：在网络分区情况下，系统尽可能保存访问到多数派分区的 Client 的写操作。不过，存在一个小的时间窗口，在此期间的写入操作可能会丢失（*failover 前的写入可能会在 failover 过程中丢失*）。如果 Client 的写入操作连接到少数派分区，则这个丢失时间窗口会更大。
    
    > 网络分区时，集群节点被划分成多数派分区（majority），少数派分区（minority）
    
- 可用性：Redis Cluster 在**大部分 master 节点可用，并且对少部分不可用的 master，每一个 master 至少有一个当前可用的 slave** 场景下能够保证集群的可用性。
    
    另外，通过使用 **replicas migration** 技术，当前没有 slave 的 master 会从当前拥有多个 slave 的 master 接受到一个新 slave 来确保可用性。
    

## 1-2 Implemented subset

Redis Cluster 实现了所有在非分布式 Redis 版本中的单 key 命令；但是对于使用多个 key 的复杂操作没有实现，比如 set 中的 unions & intersections 操作。

> 不支持多个 key 的复杂操作是为了避免 key-value 在不同的 Cluster 节点间移动，使得情况更加复杂
> 

不过，Redis Cluster 实现了被称为 **Hash Tags** 的概念：多个 key 可以通过相同的 hash tag 存储在相同的 hash slot 中，从而避免了 key 的迁移。

> 在手动 resharding 期间，多 key 操作可能变的不可用
> 

除了支持的命令不同，Redis Cluster 只支持 database 0，并不像单机版 Redis 支持多个数据库。

### 1-2-1 Why merge operations are avoided

Redis Cluster 设计是避免在多个节点中存在相同 key-value 对的冲突版本，因为 Redis 中的值通常都是比较大的，数据类型也是语义复杂的，传输和合并这样的值将会影响性能。

## 1-3 Clients and Servers roles in the Redis Cluster protocol

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
> 

## 1-4 Write safety

Redis Cluster 通过**节点间异步复制数据**，及 **last failover wins** 避免合并功能 。

> last failover wins 指当 master 节点故障时，通过 failover 机制选取的新 master 节点将直接覆盖之前 master 节点的数据，并同步给其他 slaves（新 master 节点中的数据可能不是最新的，或者有所缺失）。通过 last failover wins 机制，最后选举出的新 master 副本数据会覆盖其他所有副本数据。
> 

在网络分区期间，会存在一个写操作丢失的时间窗口。*对于发生在 master 节点多数派（majority）分区的写操作丢失窗口与少数派（minority）的丢失窗口是不同的*。

1. 对于发生在多数派 master 的写操作，Redis Cluster 会尽量保存；但是以下两种场景除外：
    a. client 写操作请求到达 master 节点，当 master 执行完成并回复 client 成功之后，master 出现异常而不可访问；但是之前的写操作并未通过异步复制到其他 slaves 中。如果 master 不可访问的时间较长而导致其中的一个 slave 被选举成新的 master，那么之前的写操作将会丢失。
    b. 由于网络分区，某个 master 不可被访问。网络分区触发了一轮选举，导致其中的一个 slave 被选举成新的 master。网络分区恢复之后，old master 变成 new master 的 slave 之前，一个 client 通过过期的路由表对 old master 节点进行写入，此时的写入将会被全部丢失。
    
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
    

## 1-5 Availability

**Redis Cluster 能容忍集群中少数节点不可访问，但不适合要求大量网络分块的应用**（如多机房部署）。

1. 集群在少数派分区侧不可用
2. 对于多数派 master 分区，如果其他每个不可访问的 master 节点都至少有一个 slave 节点可达，那么在经过 NODE_TIMEOUT 重新选举之后，多数派分区仍然可用

**为了提高集群可用性，Redis Cluster 支持 Replicas Migration：**自动将转移副本节点到孤立的 master 节点（不再拥有 slave 的 master）；每次 failover 成功之后，都会重新配置 slave 副本分布以提高下一次故障期间的可用性。

# 2-Overview of Redis Cluster main components

## 2-1 Keys distribution model

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

## 2-2 Keys hash tags

**Hash Tags 提供了一种将多个 key 分配到同一个 hash slot 的方式**。通过 Hash Tags 可以在 Redis Cluster 中实现对多个 key 的同时操作。

Hash Tags 的规则如下：

- key 包含一个 `{` 字符
- *并且* 如果在这个`{`的右面有一个`}`字符
- *并且* 如果在`{`和`}`之间存在至少一个字符

那么，`{ }` 之间的字符将被用来计算 hash slot。如 `{user1000}.following`和`{user1000}.followers`这两个 key 会被分配到相同的 hash slot 中，因为只有`user1000`会被用来计算 hash slot 值。

## 2-3 Cluster nodes attributes

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

## 2-5 Cluster topology

Redis Cluster 是全网拓扑，**每个节点都与其他节点维护 TCP 连接**。

> N 个节点的集群中，每个节点有 N-1 个传出 TCP 连接，同时有 N-1 个传入 TCP 连接
> 

集群节点间使用 **Gossip 协议**和**配置更新机制**来避免正常情况下节点间交互过多的消息。

## 2-6 Nodes handshake

对于 Cluster bus port 连接，节点总是接受并回复 ping 请求，即使该 ping 请求来自一个不可信任的节点。但是如果发送节点被认为不是集群的一部分，那么该节点的其他数据包都会被丢弃。

通过两种方式可以判断一个节点是不是集群节点：

1. **节点出现在 `MEET` 消息中**：`MEET` 消息会强制接收者接受一个节点作为集群的一部分
    
    ```c
    CLUSTER MEET ip port
    ```
    
2. **节点出现在一个被信任的节点的 Gossip 消息中**：A 节点是被信任的集群节点，B 出现在 A 的Gossip 消息中，那么 C 收到 A 的消息后也会把 B 标记为集群节点

一旦我们将某个节点加入了连接图中，那么最终所有节点会自动形成一张全连接图（fully connected graph），即**集群可以自动发现其他节点**。

> 该机制使得集群更加健壮，可以防止不同的 Cluster 在 IP 地址变更或者其他网络相关事件导致意外混合
> 

# 3-Redirection and resharding

Redis Client 可以向集群中的任意节点发送查询请求，包括 slave 节点。

收到请求的集群节点会分析该查询请求：

1. 如果请求是可以接受的，则会判断 key 所属的 hash slot 及对应的节点
    
    > 可以接受的请求是指：a. 请求中只包含一个 key；b. 多个 key 同属一个 hash slot
    > 
2. 如果目标 hash slot 被当前节点管理，则直接处理请求
3. 否则，当前节点回复一个 **`MOVED` 异常：***异常包含了 key 所属的 hash slot 及管理该 hash slot 的节点（IP + Port）*。
    
    ```powershell
    GET x
    -MOVED 3999 127.0.0.1:6381
    ```
    
4. Client 收到重定向的回复之后，需要向指定的 IP + Port 重新补发查询请求。
    
    > 如果在补发请求之前，集群配置再次发生了变化，导致刚才的节点不再管理对应的 hash slot，那么也会返回 `MOVED` 异常，Client 仍需要再次补发请求
    > 
5. 对于 `MOVED` 异常，Client 除了重新补发，还**需要缓存 hash slot 与集群节点的映射关系**，以提高之后请求的效率。不过，该策略不是强制的，Client 还可以通过 `CLUSTER NODES` 命令**全量刷新 hash slot 与集群节点的映射关系并缓存**。
    
    > 当 Cluster 处于稳定状态时，所有的 Client 最终都可以维护 hash slot → cluster nodes 的映射关系，减少重定向的概率，提升集群处理的效率。
    > 

除了 `MOVED` 重定向，Client 需要能够处理 `ASK` 重定向。

## 3-2 Cluster live reconfiguration

为了支持 Cluster 动态重新配置，需要实现 **hash slot 在集群节点间迁移能力**。slot 迁移的场景有：

- 添加节点：需要将一些已经存在 hash slot 集合迁移到新节点上
- 删除节点：将被删除节点上的所有 hash slot 集合转移到其他节点
- 集群 rebalance：将给定的 hash slot 集合在节点间移动

**hash slot 迁移的核心是分布在该 slot 上的 key 集合迁移**。

下面是一些 slot 迁移的命令：

- *CLUSTER ADDSLOTS slot1 [slot2] ... [slotN]*
- *CLUSTER DELSLOTS slot1 [slot2] ... [slotN]*
- *CLUSTER SETSLOT slot NODE node*
- *CLUSTER SETSLOT slot MIGRATING node*
- *CLUSTER SETSLOT slot IMPORTING node*

`ADDSLOTS`和`DELSLOTS`只是用来简单地在 Redis Cluster 节点上分配或移除 slot。**在分配了 hash slots 之后，节点会通过 Gossip 协议在集群中传播这些信息**。

`SETSLOT slot NODE node` 用来给特定的节点分配 slot。

`CLUSTER SETSLOT slot MIGRATING node` 与 `CLUSTER SETSLOT slot IMPORTING node` 命令用于将 hash slot 从一个节点迁移到另一个节点。迁移过程总涉及到两个特殊的状态：**MIGRATING** & **IMPORTING**。

- 当 hash slot 处于 **MIGRATING** 状态时，如果某个查询请求的 key 在该 slot 中，则当前节点会处理该查询请求；否则，节点就会通过 **`ASK`** 命令重定向到迁移的目标节点
- 当 hash slot 处于 **IMPORTING** 状态时，如果某个查询请求后紧跟着 **`ASKING`** 命令，则该请求就会被执行；否则，该请求就会通过 **`MOVED`** 命令重定向到管理该 hash slot 的真正节点

### 3-2-1 Migration process

假设 Redis Cluster 中存在 A，B 两个节点，我们期望将 hash slot 8 从 A 迁移到 B，则：

- 向 B 发送命令 *CLUSTER SETSLOT 8 IMPORTING A*
- 向 A 发送命令 *CLUSTER SETSLOT 8 MIGRATING B*

所有其他节点在收到一个对属于 hash slot 8 的 key 的查询时，仍然会继续将 Client 重定向到 A，会导致：

- 所有对已经存在的 key 查询将会被 A 处理
- 所有在 A 上不存在的 key 都会被 B 处理，因为 A 会将其重定向到 B（**`ASK`**）
- 将**不会在 A 上创建新的 key**

> 在 key 的迁移过程中，从 Client 的视角来看，同一个 key 只会存在 A or B 中。
> 

## 3-3 ASK redirection

在 slot 迁移过程中，使用的是 **`ASK`** 命令进行重定向，为什么不使用 **`MOVED`** 命令进行重定向？

- `**MOVED**` 表明**目标 hash slot 永久地被一个不同的节点所管理**，并且以后的请求也应该继续指向该节点；而 **`ASK`** 表明**只是下次查询需要发送给另一个特定节点**。
- 我们期望 Client 总是**先尝试访问 A，然后在有必要的时候再访问 B**。因为同一个 hash slot 中有很多 key，可能某次查询的 key 已经不在 A 上了，需要再次重定向到 B 查询；但是可能下次查询的 key 仍然在 A 上。
    
    > **`ASK`** 重定向只发生在 hash slot 迁移过程中，对 Cluster 的影响可以接受
    > 

一般来说，`ASKING` 命令会为 client 设置一个**单次标签(one-time flag)**，以允许该 client 可以访问处于 `**IMPORTING**` 状态的 slot 一次。

从 client 的视角来看，当收到 **`ASK`** 重定向命令之后：

- 仅仅将这条查询重定向到指定的新节点，**之后的命令还是继续发送给老的节点**
- 重定向查询必须以一条`ASKING`命令开始
- 暂时不要在本地将 hash slot 8 映射为节点 B

当 hash slot 8 迁移完成之后，A 会返回一个 **`MOVED`** 重定向命令，client 需要在本地更新 hash slot 8 的映射节点为 B。

## 3-4 Clients first connection and handling of redirections

Redis Cluster 的 Client 如果不将 hash slots → cluster nodes 的映射关系缓存在本地，那么每次查询都需要根据 `MOVED` 重定向找到正确的节点，效率会很低。为了提高查询效率，Client 需要将 slots → nodes 的映射缓存在本地，但是该缓存并不总是最新的。

在以下两个场景中 Client 需要获取全量的 hash slots → cluster nodes 的映射关系：

1. **启动阶段初始化 slots 配置信息**
2. **收到 `MOVED` 的重定向信息**

> Client 在收到 MOVED 重定向时，通常会有多个 slots 发生了变动（比如 slave 晋升，该节点的 slots 会重新映射），此时 Client 全量更新 slots 会使问题处理起来比较简单
> 

## 3-5 Multiple keys operations

通过使用 hash tags 可以对多个 key 进行操作。

不过，在 resharding 期间，多个 key 操作可能会不可用：

- 如果这些 key 在同一个节点上，则可以继续可用
- 如果这些 key 在不同的节点上，则不可用
    
    > 请求的多 key 所属的 slot 在迁移过程中，这些 **key 可能同时分布在源节点与目标节点**上
    > 

## 3-6 Scaling reads using replica nodes

slave 节点默认不可读，**所有对 slave 节点的读请求都会被重定向到 master 节点**（返回 MOVED 异常）。不过可以通过 READONLY 命令将 slave 节点设置为可读。

# 4-Fault Tolerance

## 4-1 Heartbeat and gossip messages

Redis Cluster 存在两种**心跳包（heartbeat packets）**，分别称为 **ping** 包 & **pong** 包。这两种心跳包结构相同，并且都会携带重要的配置信息。

- 当节点收到 ping 包时，总会回复一个 pong 包
- 有时节点为了尽快将自身的配置信息发送出去，会直接向其他节点发送 pong 包

每个节点会随机挑选一些节点并发送一些 ping 包（**Gossip 协议**），每个节点在指定时间范围内发送的 ping 包与接收到的 pong 包是一个常数。

> 会主动对 *NODE_TIMEOUT/2 时间内没有发送过 ping 或者从其接收到 pong* 的节点发送 ping 包
> 

## 4-2 Heartbeat packet content

ping & pong 包的内容包含两部分：

1. **包头（header）**：该部分不仅可以用于 ping & pong 消息，也可以用于其他类型消息（如重新选举消息）
    
    
    | 内容 | 备注 |
    | --- | --- |
    | Node ID | 节点创建时生成，并在 Redis Cluster 生命周期内保持不变 |
    | currentEpoch & configEpoch | 由 Redis Cluster 用来加载分布式算法；如果发送者是 slave 节点，则 configEpoch 就是其 master 的 configEpoch |
    | node flags | 表明发送者是 slave 还是 master；同时包含一些其他信息 |
    | hash slots 的 bitmap | 如果发送者是 slave，则表示其 master 的 hash slots 的 bitmap |
    | TCP Port | 用于接受命令的普通端口（+10000 表示 Redis Cluster Bus 端口） |
    | Cluster state | 发送者视角下的集群状态（down or ok） |
    | master Node ID | master 节点的 Node ID（如果发送者是 slave） |
2. **Gossip section**：仅存在 ping & pong 中；包含***发送者视角下集群中其他节点状态，用于故障检测 & 节点发现***
    
    
    | 内容 | 备注 |
    | --- | --- |
    | Redis Cluster 中其他节点信息 | 包含 Node ID, IP, Port, Node flags(FAIL, PFAIL) |
    
    > Gossip section 段中包含的节点数与集群大小成正比
    

## 4-3 Failure detection

Redis Cluster 故障检测机制用来**识别一个 master or slave 节点对集群中大部分 master 节点是否是可达的（reachable）**。如果一个 master 节点不可达，则会将其一个 slave 节点晋升为 master；***如果无法将 slave 晋升为 master，则集群会被置为 error 状态，并停止接受客户端的请求***。

> 选举过程需要 master 节点参与
> 

如何定义不可达？

- 一个已经发送出去，但是没有接收到对方回复的 ping 被称为活跃 ping（active ping）。如果一个活跃 ping 挂起时间超过 `NODE_TIMEOUT`，则认为接收方不可达。
- `NODE_TIMEOUT` 必须大于正常的网络往返时间。为了增加可靠性，在 `NODE_TIMEOUT/2` 时间过后如果还没收到回复，会尝试联系其他节点，确保自身连接的活跃性。

### 4-3-1 PFAIL & FAIL

Redis Cluster 每个节点都会保存其他节点的一些 flags 信息，其中有两个 flag 用于**故障检测：`FAIL & PFAIL`**。

1. `PFAIL` flag (possible fail)
    
    可能故障，是一个不需要确认的故障类型。
    
    当节点发现某个节点失联超过 `NODE_TIMEOUT` 时，会在本地将其标记为 `PFAIL`。
    
    > 检测节点与被检测节点都可以是 master 或者 slave
    > 
2. `FAIL` flag（fail）
    
    `PFAIL` 只是每个节点针对其他节点状态的本地信息，不能以此来判断是否进行选举。为了确认节点确实不可达了，需要将 `PFAIL` 升级为 `FAIL`。
    

### 4-3-2 PFAIL → FAIL

每个 Gossip 消息中都会包含当前节点已知的一部分其他节点的状态（是否 `PFAIL`），节点之间会交换自己已知信息；将 `PFAIL` 升级为 `FAIL`的流程为：

- 节点 A 检测到 B 节点不可达，将其标记为 `PFAIL`
- 节点 A 通过 Gossip 消息收集大部分 master 节点对 B 节点状态标记的 flag
- 多数 master 节点在 `NODE_TIMEOUT * 2` 时间范围内将 B 标记为 `PFAIL` or `FAIL`
- A 将 B 标记为 `FAIL`
- A **发送一个 `FAIL` 消息给所有可达节点**
- **FAIL 消息会强制所有接收到消息的节点将不可达节点 B 标记为 FAIL**，而不管自己当前是否已经将 B 标记为 PFAIL

`FAIL` flag 是单向（one way）的：只能 `PFAIL` → `FAIL`。不过 `FAIL` flag 可以在以下情况中被清除：

1. **节点重新可达，并且该节点是 slave**：因为 slave 节点不会发生故障转移（failover）
2. **节点重新可达，并且没有管理任何 hash slots 的 master 节点**：这种节点实际上没有参与集群管理，需要等待被配置后加入集群
3. **节点重新可达，且为 master 节点，同时在较长时间内（N * NODE_TIMEOUT）没有检测到有 slave 节点晋升**：此时最好将其作为 master 节点重新加入集群

`PFAIL → FAIL` 的转换过程使用了**弱一致性**（weak agreement）:

1. 节点在一段时间内收集其他 master 节点的视图，即使多数 master 达成一致，也只能说明在不同的时间，不同的节点达成一致，无法确定在什么时刻获得了多数 master 的一致结果。不过，过期的结果会被抛弃掉（`NODE_TIMEOUT * 2` ），所以**可以确定多数 master 节点一定是在某个时间窗口内达成一致**。
2. `FAIL` 消息虽然可以强制其他节点接受该判断结果，但是无法保证消息被所有节点接收到：如出现了网络分区

### 4-3-3 Conner case

**Redis Cluster 所有节点最终会对一个给定的节点状态达成一致**。两个由于集群脑裂引起的场景：

1. 如果多数派 master 将一个节点标记为 FAIL，那么所有其他节点最终也会将该 master 标记为 FAIL：在指定的时间窗口内，集群中会有足够多的失败报告
2. 如果少数派 master 节点将一个节点标记为 FAIL，并不会发生 failover，最终该 FAIL 状态会被清除：在 N*NODE_TIMEOUT 内没有晋升

### 4-3-4 Moreover

`FAIL` flag 只是一种触发机制，用于触发执行 slave 晋升操作。slave 也可以在发现 master 不可达之后主动启动晋升操作，并尝试获得多数 master 节点的同意。

`FAIL` 机制虽然有些复杂，但是可以使得集群意识到自己处于一个 error 状态，从而拒绝写操作；而且可以减少由于 slave 自身问题触发错误的选举尝试。