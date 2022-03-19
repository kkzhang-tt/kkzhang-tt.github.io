---

layout: post
title: "Google File System"
category: "system-design"
author: "kkzhang"
image: gfs/gfs_1.png
---
# 0-Keywords

容错(Fault Tolerance)，扩展性(scalability)，数据存储(Data Storage)，集群存储(Clustered Storage)

# 1-Introduction

GFS 以可用性，可伸缩性，可靠性，性能等作为目标，与之前的分布式文件系统有所区别的是，其设计思路主要结合 Google 内部的实际情况考虑：

1. 组件失效是正常现象（Normal）而非异常（Exception）
    * 需要支持系统监控，容灾，自动恢复机制
2. 存储的文件通常比较大（Huge）
    * 如果文件块（block）比较小的话（比如以 KB 为单位），那么单个文件需要由很多文件块组成，不利于文件管理，因此文件块的大小需要多方面衡量；大文件的 IO 操作也需要考虑
3. **大多数文件的更新方式为追加（Appending）而不是覆盖写**（Overwriting）
    * 文件写完之后多为顺序读，几乎不包含随机写的操作；因此，数据追加操作是性能优化与原子性的主要考量因素
4. 应用程序和文件系统 API 的协同设计提高了整个系统的灵活性（Flexibility）
    * **支持原子性地 Append 操作**，从而保证多个 Client 能够同时进行 File Appending，不需要额外的同步操作来保证一致性

# 2-Overview

## 2.1-Assumptions

针对上述的情况做进一步说明，阐述整个系统的设计预期：

1. 系统集群是由许多廉价机器组成，组件失效是比较常见的状态；因此需要实现对整个系统的持续监控，提供冗余并恢复失效的组件
2. 系统存储大量的大文件：**文件数量多，文件容量大**；不过也需要支持小文件（但是并不需要针对小文件进行特殊优化）
3. 读负载包含两种：**大规模的流式读** & **小规模的随机读**；同一个 Client 的操作通常是同一个文件的连续部分
4. 写负载包括：**大规模顺序 Append** & **小规模的随机写**；数据一旦被写入文件就会很少再次修改
5. 需要支持多个 Client 并行 Append 数据到同一个文件的语义，即需要**支持原子 Appending**
6. **高性能，大批量地处理数据**比低延迟更加重要

## 2.2-Interface

GFS  支持类似 POSIX 标准的 API，包括创建文件，删除文件，打开文件，关闭文件等；同时还支持快照与追加操作。

> 文件以分层目录的形式组织，并用路径来标识
> 

快照可以以较低的成本创建一个文件或者目录树的拷贝；**记录追加允许多个客户端同时对一个文件进行数据追加操作，并保证每个追加操作都是原子性的**。

> 客户端不需要额外的锁定同步操作就可以完成并行追加文件操作
> 

## 2.3-Architecture

![]({{site.baseurl}}/images/gfs/gfs_1.png)

一个 GFS 集群包含一个 Master 节点，多个 Chunk 服务器，并且同时被多个 Client 访问。

> 一个 Master 节点是指一个系统中只存在一个逻辑上的 Master 组件，但是该组件可能存在多个副本以实现容灾
> 

GFS 中存储的**文件被拆分成多个固定大小的 Chunk 块**。Chunk 创建时，Master 节点会给其分配一个**唯一且不变的 ID**。Chunk 服务器把 Chunk 以 Linux 文件的形式保存在本地磁盘上，并支持**根据 Chunk 标识 & 字节范围进行数据读写**。

> 为了提高可靠性，**每个 Chunk 都会被复制到多个 Chunk 服务器上**
> 

Master 节点用于管理 GFS 系统中的元数据，具体包括：

1. File NameSpace
2. 文件访问控制信息
3. ***文件与对应的 Chunk 块列表的映射关系***
4. ***Chunk 的位置信息***

除了系统元数据，Master 节点还用于控制系统的活动：

1. Chunk 租用管理
2. 孤儿 Chunk 的回收
3. Chunk 块在 Chunk 服务器之间的迁移

Master 节点维护与 Chunk 服务器之间的心跳，同时发送指令到各个 Chunk 服务器并接收 Chunk 服务器的状态信息。

**Client 与 Master 节点通信用于获取元数据**，对于**文件的数据操作是由 Client 直接与 Chunk 服务器进行交互的**。

Client 与 Chunk 服务器都不需要对文件数据进行缓存：

1. Client 一般是以流的方式读取一个很大的文件
2. Chunk 以本地文件的形式保存在 Chunk 服务器上，可以完全依赖 Linux 文件系统的缓存

## 2.4-Single Master

上面介绍，整个系统中只有一个 Master 节点；单一 Master 节点简化了系统设计，但是单个节点可能会导致其成为整个系统的瓶颈，因此要尽量减小 Master 的读写压力。

- Client 只会向 Master 询问其应该访问的 Chunk 服务器，并不从 Master 进行文件数据读取
- Client 会将从 Master 获取的元数据缓存在本地一段时间，后续将直接和 Chunk 服务器进行数据读写操作
- **Client 在向 Master 请求 Chunk 信息时，通常会一次性请求多个 Chunk，而且 Master 响应时也会返回被请求的 Chunk 后面的 Chunk 信息，减小了交互次数**

整体的文件读取流程如下：

1. Client 把业务程序中指定的文件字节偏移量，*根据固定的 Chunk 大小转换为对应的 Chunk 索引*
2. Client 把 FileName + Chunk Index 发送给 Master 节点
3. Master 将请求的 *Chunk 唯一标识与其副本位置信息返回给 Client*
4. Client 用 *FileName + Chunk Index 作为 key 将这些信息缓存在本地*
5. Client 访问其中较近的 Chunk 副本，请求中携带了 *Chunk 唯一标识 + 访问文件的字节范围*
  
    > 之后针对该 Chunk 的访问都不会再访问 Master，除非缓存失效或者文件被重新打开
    > 

## 2.5-Chunk Size

GFS 的 Chunk 大小为 64MB，远远大于一般文件系统的 Block Size(Linux 默认 4KB)；使用较大 Chunk 的原因有：

- **减小了 Client 与 Master 之间的交互频率**：在获取 Chunk 信息后就可以对同一个 Chunk 进行多次读写操作，并且不需要再次请求 Master；由于读写负载主要以连续读写大文件为主，因此可以有效降低负载
- 由于使用了较大尺寸的 Chunk，**Client 能够实现对一个 Chunk 的多次操作**，可以通过与 Chunk 服务器之间维护较长时间的 TCP 连接来降低网络负载
- 较大尺寸的 Chunk 能够**减少 Master 需要保存的元数据信息**，有利于将所有元数据加载到内存中

不过，使用较大尺寸的 Chunk 也可能会带来其他问题：对于比较小的文件会包含较小数目的 Chunk，甚至只有一个 Chunk，当有许多 Client 同时对一个 Chunk 进行多次访问时，会导致**存储该 Chunk 的服务器产生热点**。

> 由于 GFS 中的文件大多都是比较大的文件，因此热点问题并不重要；*为了降低热点问题，可以提高 Chunk 的副本数，或者允许 Client 从其他 Client 中读取数据*
> 

## 2.6-Metadata

Master 中主要存储了三种元数据，并且**所有元数据都保存在内存中**：

1. 文件和 Chunk 的命名空间
2. **文件和 Chunk 的映射关系**
3. 每个 Chunk 副本存放的位置信息

前两种元数据信息会以*记录变更日志的方式记录在操作系统的系统日志文件中*，日志文件存储在本地磁盘上，同时日志会被复制到其它的远程 Master 服务器上；从而降低 Master 服务器崩溃导致数据不一致的风险。

> Master 并不会持久化 Chunk 的位置信息，当 Master 节点重新启动时或者有新的 Chunk Server 加入集群时，Master 会向各个 Chunk 服务器轮询它们所存储的 Chunk 信息
> 

### *2.6.1-In-Memory Data Structures*

由于元数据都保存在内存中，因此 Master 的操作速度非常快；但是这种方式会使得 Chunk 的数量及整个系统的承载能力都受限于 Master 的内存大小。

为了减小内存占用，每个 Chunk 只需要 64 个字节的元数据进行管理，每个文件的命名空间所需要的元数据也在 64 个字节以下。

### *2.6.2-Chunk Locations*

Master 节点并不持久化 Chunk 的位置信息，只会在内存中缓存该信息。

Master 总是维护最新的 Chunk 信息列表：

- Master 控制了 Chunk 的位置分配
- Master 服务器启动时轮询所有的 Chunk 服务器，以获取当前 Chunk 信息
- Master 与 Chunk Server 之间维护周期性的心跳，以监控 Chunk 服务器的状态

> 只将数据缓存在内存中的设计简化了 Chunk Server 发生变更时（新加入集群，离开集群，重启，重命名等），Master 与 Chunk Server 之间的数据同步问题。
> 

### *2.6.3-Operation Log*

操作日志包含了重要元数据的变更记录，用于 Master Server 的容灾备份。

为了确保数据的完整性，**只有在元数据的变更记录持久化到本地日志文件，并且持久化到其他 Master 节点的磁盘后，才会响应 Client**。

有了操作日志，Master Server 在故障恢复时可以通过日志恢复到最近的系统状态。为了减小 Master 恢复日志的启动时间，当操作日志增长到一定程度之后会进行一次 **CheckPoint**；所有的状态数据会写入一个 CheckPoint 文件，并且新的 CheckPoint 文件包含之前的所有修改。当 Master 节点进行恢复时，只需要**最新的 CheckPoint 文件及其后的日志文件即可**。

## 2.7-Consistency Model

GFS 支持一个宽松的一致性模型

- 对文件 NameSpace 的修改（文件创建，重命名等）是原子性的，由 Master 节点统一控制：**Master 节点通过 NameSpace 锁保证了原子性与正确性**；同时有序的操作日志记录了变更的全局顺序。
- 对文件 Data 的修改分为两种操作：**覆盖写**（Write）& **记录追加**（Record Append）（文件数据操作直接与 Chunk Server 交互）；文件内容修改后的一致性状态取决于修改的类型，成功与否，是否并发修改等。
  
    ![]({{site.baseurl}}/images/gfs/gfs_2.png)
    

GFS 对文件状态有以下几个定义（修改的部分称作 region）：

1. ***Consistent***: 如果 region 在 Chunk Sever 的多个副本中都相同，即 Client 从各个副本能获取相同的文件内容，那么这个 region 被称为一致的
2. ***Consistent but undefined***: region 中的内容虽然在多个 Chunk 副本中一致，但是该 region 中的信息与 Client 预期更新的内容并不相同；比如 Client 想要将文件中的一段内容设置为 abcdefg，但是由于并发修改等原因导致该 region 被更新为 abc123g，与 Client 的预期不一致
3. ***Defined***: region 在多个 Chunk 副本中保持一致，并且与 Client 的预期相同，Client 能够正确读取更新的数据
4. ***Inconsistent***: 由于更新失败导致多个 Chunk 副本中的 region 信息并不一致

对于覆盖写（Client 指定写入的文件偏移位置）：

1. 当一个 Client *串行修改数据成功*后，由于没有其他 Client 干扰，那么修改的 region 就是已定义的(隐含了一致性): 所有的 Client 都可以看到正确写入的内容
2. 当多个 Client *并行修改数据成功*后，大部分情况下，该 region 中会***包含来自多个 Client 修改操作的、混杂的数据片段***，此时 region 处于一致但是未定义的状态：所有的 Client 看到同样的数据，但是无法正确读到任何一次写入操作写入的数据
3. 失败的修改操作导致一个 region 处于不一致状态(同时也是未定义的): 不同的 Client 在不同的时间会看到不同的数据

对于记录追加操作（文件偏移位置由 Chunk Server 主副本指定）：

1. 不管是单独一个 Client 还是多个 Client 并行执行文件追加操作，*GFS 至少可以把数据原子性的追加到文件中一次，并返回给 Client 一个偏移量，表示了包含了写入记录的、已定义的 region 的起点；*此时 region 处于已定义的状态
2. 在追加操作失败重试期间（重试之后如果成功也会被人为该追加操作是成功的），GFS 可能会在*文件中间插入填充数据或者重复记录；*这些数据占据的文件 region 被认定是不一致的， 这些数据通常比用户数据小的多。

经过一系列成功的修改操作后，GFS确保被修改的文件region是已定义的，且包含最后一次修改操作的数据。

# 3-System Interactions

系统设计时尽量减少与 Master 节点的交互，降低其负载。

## 3.1-Leases and Mutation Order

为了保证文件的变更顺序在多个 Chunk 副本之间的一致性，引入了 **Lease 机制**。

> 每次变更都需要按序在所有 Chunk 副本上执行，才能保证文件的一致性
> 
- Master 节点与一个 Chunk 副本建立 Lease，该副本被称为主 Chunk
- *主 Chunk 会对所有 Chunk 的更新操作进行序列化*，所有副本都会按照这个序列进行操作

引入了 Lease 机制后，文件更新的全局顺序为：**首先由 Master 节点选择 Lease 的顺序
决定，然后由 Lease 中主 Chunk 分配的序列号决定**。

引入 Lease 机制有助于减少 Master 节点的管理负载；Lease 初始值是 60s，当 Chunk 更新时，Lease 可以延长。

> Lease 的建立 or 延长 or 取消等操作信息均放在 Master 节点与 Chunk Server 之间维护的心跳中
> 

Client 进行文件更新操作的流程如下：

![]({{site.baseurl}}/images/gfs/gfs_3.png)

1. Client 请求 Master 获取持有 Lease 的 Chunk 以及其他 Chunk 副本信息
  
    > 如果暂时没有一个 Chunk 持有 Lease，则 Master 节点从 Chunk 副本中选择一个建立 Lease
    > 
2. Master 将主 Chunk 信息及其他副本信息返回给 Client
  
    > Client 缓存这些数据以便后续使用；并且只有在主 Chunk 不可用，或者主 Chunk 回复信息表明它已不再持有 Lease 的时候，Client 才需要重新与 Master 交互
    > 
3. Client 把需要修改的文件数据推送到所有 Chunk 副本上
  
    > 推送的副本顺序可以任意，可以根据网络拓扑情况进行推送，而不需要关心哪个是主 Chunk；Chunk Server 接收到数据并保存在其内部 LRU 缓存中，直到数据被使用或者过期
    > 
4. 当所有的 Chunk 副本都确认收到数据之后，Client 再次发送写请求到主 Chunk 
  
    > 该写请求中标识了之前推送的数据信息；主 Chunk 收到该写请求后，为其分配一个操作序列号（所有 Client 请求会被分配连续的序列号），该序列号保证了操作的顺序；之后，主 Chunk 按照操作序列依次将变更操作应用到本地，并更新自身记录状态
    > 
5. 主 Chunk 把变更请求发送到其他副本中；每个副本依照主 Chunk 分配的序列号以相同的顺序执行
6. 其他副本执行完成后，回复主 Chunk
7. 主 Chunk 将变更操作的结果返回给 Client
  
    > 任何副本产生的任何错误都会返回 Client；在出现错误的情况下，变更操作可能在主 Chunk 和一些二级副本中执行成功，（如果操作在主 Chunk 上失败了，操作就不会被分配序列
    号，也不会被传递）那么 Client 的请求被确认为失败，被修改的 region 处于不一致的状态。Client 通过重复执行失败的操作来处理这样的错误。在从头开始重复执行之前，客户机会先从 3 → 7 做几次尝试。
    > 

在上述流程中，如果一次变更操作写入的数据量很大，或者数据跨越了多个 Chunk，那么 GFS Client 会将其拆分成多个写入操作。

## 3.2-Data Flow

为了提高网络效率，GFS 将数据流与控制流分开。

**控制流**：Client 将变更请求发送到主 Chunk Server，主 Chunk 把该请求同步到其他 Chunk 副本中顺序执行。

**数据流**：Client 把文件数据推送到所有 Chunk 副本上（此时不需要关心哪个是主 Chunk）。

在数据流推送时，为了提高机器的带宽利用率，同时减少数据推送的延迟，**数据沿着一个 Chunk  Server 链顺序推送**，而不是以其它拓扑形式分散推送(如树型拓扑结构)。

> 线性推送模式下，每台机器所有的出口带宽都用于以最快的速度传输数据，而不是在多个接受者之间分配带宽

在推送时，尽量选择一台还没有接收数据，并且离自己最近的 Chunk Server。

## 3.3-Atomic Record Appends

GFS 提供了一个原子记录追加的操作方式。

对于传统覆盖写的方式，Client 需要指定写入位置的偏移量；如果对同一个 region 进行并发写，可能会导致 region 的尾部包含多个 Client 写入的数据片段。

> 对于覆盖写，为了保证原子性，需要额外的同步机制，比如分布式锁，用于确保数据能够完整地写入

使用记录追加的方式，Client 只需要指定写入的数据，至于写入的偏移量由 GFS 执行；GFS 保证至少有一次原子写入操作执行成功（即将文件数据正确地追加）。写入的数据追加到 GFS 指定的偏移量处，之后再将该偏移量返回给 Client。

如果追加操作在任何一个副本上执行失败了，Client 需要进行重试。由于重试机制，因此可能会导致同一个 Chunk 在不同副本上的数据不一致：有的副本可能包含重复的数据记录。

**GFS 不保证所有的Chunk 副本在 Byte 级别上是一致的，只保证数据作为一个整体至少被原子地写入一次**（可能会失败重试）。

# 4-Master Operation

Master 节点执行的操作有：

1. 文件 NameSpace 相关的所有操作
2. Chunk 副本管理
   - Chunk 的存储位置
   - 创建新的 Chunk 及相关副本
   - Chunk Server 之间的负载均衡
   - 回收不再使用的存储空间
   - ...

## 4.1-Namespace Management and Locking

GFS 的 NameSpace 就是一个**全路径与元数据（MetaData）的映射**表。通过使用前缀压缩，降低内存占用。

前缀树的***每个节点（即文件 or 目录路径）都有关联一个读写锁***。每个 Master 操作都需要先获取一系列锁，比如一个操作涉及到 /a/b/c 路径，则首先需要获取到 /a, /a/b 节点的读锁，同时要获取到 /a/b/c 节点的读写锁。

> 文件的创建操作不需要获取父目录的写锁，只需要读锁即可

采用这种锁方案可以*支持对同一个目录下的并行操作*；比如同一个目录下（/root/kz）同时创建多个文件(a,b,c)：每个创建文件的操作都需要获取目录 /root, /root/kz 的读锁，以及要创建的文件名(a,b,c)的写锁。

> 创建文件时对目录读锁的获取足够防止目录被删除，重命名等更新操作，对文件的写锁用于防止相同名称的文件多次创建

## 4.2-Replica Placement

Chunk 副本位置的选择有两个目标：

- 最大化数据可靠性与可用性
- 最大化网络带宽利用率

为了实现这两个目标，仅仅将副本分散在多个机器上是不够的，这只能预防机器损坏带来的影响及最大化没台机器的带宽利用率。因此，需要在多个机架之间分布存储 Chunk 副本。这样即使一个机架上的机器全部损坏，Chunk 仍然是可用的；网络带宽方面能够有效利用机架的带宽整合。

## 4.3-Creation & Re-replication & Rebalancing

Chunk 副本的创建有三个原因：新建 & 重新复制 & 重新负载均衡。

### *4.3.1-Creation*

当***新建***一个 Chunk 时，Master 需要选择该 Chunk 存放的机器，需要考虑的因素有：

- *该 Chunk Server 的磁盘使用率低于平均磁盘使用率*，用于最终平衡 Chunk Servers 之间的磁盘使用率
- *该 Chunk Server 最近创建 Chunk 的次数在限制之内*；虽然创建 Chunk 操作是廉价的，但是创建 Chunk 之后意味着在不久之后会有大量数据写入该 Chunk，所以要对 Chunk Server 一段时间内创建 Chunk 的次数进行限制
- 如 4.2 描述的，我们需要将 Chunk 分散在不同的机架上

### *4.3.2-Re-replication*

当 Chunk 的有效副本数小于用户指定的复制因数时，Master 需要对 Chunk 进行***重新复制***。出现这个现象的原因有：

- 一个 Chunk Server 不可用
- Chunk Server 上报其存储的一个副本损坏
- Chunk Server 的一个磁盘损坏
- Chunk 的复制因数提高了
- ...

同一时间可能有多个 Chunk 需要被重新复制，Master 需要根据一些因素对 Chunk 进行优先级排序，之后选择优先级最高的 Chunk 进行复制。

- Master 命令一个可用的 Chunk Server 直接从可用的副本克隆一个出来，之后再传输到新副本中；新副本的选择与创建 Chunk 时类似

### *4.3.3-Rebalancing*

Master 需要周期性的对 Chunk 副本进行负载均衡：首先检查当前的副本分布情况，之后再***移动副本***以更好地利用机器的磁盘空间，更好地进行负载均衡。

在确定需要移动的副本时，选择 Chunk Server 剩余空间低于平均值的 Chunk 副本；新副本的位置选择与创建时类似。

## 4.4-Garbage Collection

GFS 在文件删除后并不会立即回收 Chunk 的物理空间，而是采用惰性删除的策略；*只有在文件 & Chunk 级的垃圾回收机制中才会回收其物理空间*。

### *4.4.1-Mechanism*

- 当一个文件被用户删除时，Master 节点并不会立即回收文件资源，而是*将文件名更新为包含删除时间戳 & 隐藏的名字*。

  > 删除动作同样会记录到 Master 的操作日志中；在文件真正被删除之前，仍然可以通过更新后的特殊文件名进行访问，也可以通过把特殊文件名修改回原来的文件名进行文件恢复

- Master 节点对文件系统的 NameSpace 做定期扫描时，会删除三天前的所有隐藏文件。

- 当隐藏文件从 NameSpace 中删除后，*Master 才会将内存中与这个文件相关的元数据删除，从而切断删除文件与它所包含的 Chunk 之间的连接*。

  > 在对 NameSpace 进行扫描时，Master 会找到孤儿 Chunk（不被任何文件包含的 Chunk），并删除它们的元数据

- Chunk Server 在与 Master 周期的心跳交互中，会上报其包含的 Chunk 子集信息。*Master 会周知 Chunk Server 哪些 Chunk 的元数据已经被删除了*；之后，Chunk Server 可以任意删除这些 Chunk 副本。

### *4.4.2-Discussion*

垃圾回收策略相对于直接删除文件有几个优势：

1. 对于组件失效是常态的大规模分布式系统来说，垃圾回收的方式更加简单可靠
   1. 在 Chunk 创建时，有些副本可能会创建成功，有些可能会失败，创建失败的副本处于无法被 Master 节点识别的状态；副本删除消息可能会丢失，Master 需要重新发送删除消息
   2. 垃圾回收提供了一致的，可靠的清除无用副本的方法
2. 垃圾回收动作为 Master 节点的周期性后台进程，可以在 Master 节点相对空闲的时候进行回收动作，尽量减少 Master 节点的负载
3. 对于意外的文件删除动作提供了安全保障

不过，延迟空间回收可能产生的问题：在 Chunk Server 存储空间比较紧缺的时候，会阻碍用户存储空间的调优使用；特别是反复创建删除大量临时文件时。为了降低这个影响，可以为不同的文件设置不同的回收策略。

## 4.5-Stale Replica Detection

当 Chunk Server 失效时，保存在其中的 Chunk 可能会在其失效期间错过一些修改操作而过期失效。

为了判断 Chunk 是否有效，Master 节点会为每个 Chunk 保存其版本号；**每当与 Chunk 副本签订新的租约时，就会增加 Chunk 的版本号**，然后通知最新的副本。

> 每个 Chunk 都会被分配唯一的标识；Chunk 的版本号大小用于判断其是否有效

新的版本号记录会被 Chunk Server & Master 节点持久化到本地。

当 Chunk Server 与  Master 心跳交互时，会上报其保存的 Chunk 集合及对应的版本号。

- 当 Master 发现上报 Chunk 副本的版本号比自己存储的版本号低的时候，就会判定该 Chunk 副本失效；*在垃圾回收时会移除所有过期失效的副本*
- 如果 Master 发现上报的 Chunk 副本版本号中有比自己存储的版本号高的时候，就会认为它和 C hun

当 Client 请求 Master 获取 Chunk 副本信息时，Master 不会下发失效的 Chunk 副本（这些副本会在垃圾回收时移除），并且会告知 Client 当前 Chunk 的最新版本。

Client 与 Chunk Server 在对 Chunk 进行操作前会验证 Chunk 是否有效。

# 5-Fault Tolerance & Diagnosis

## 5.1-High Availability

为了提高集群的可用性，GFS 使用了两条策略：快速恢复 & 复制

### *5.1.1-Fast Recovery*

Master & Chunk Server 在关闭之后能够在几秒内重新启动恢复；重启期间，已经建立的请求会超时，之后会访问新的节点进行重试。

> Master 节点存在多个冗余，并且 Master 的操作日志会定时创建 Check Point；Chunk Server 也会定时创建快照

### *5.1.2-Chunk Replication*

每个 Chunk 都会被复制到不同机架上的不同机器中，复制因子默认为 3.

当 Chunk Server 离线，或者通过校验发现某一个 Chunk 的数据已被损坏，Master 通过重新克隆最新副本来保证每个 Chunk 数据的完整性。

### *5.1.3-Master Replication*

虽然 GFS 系统中只有一个 Master 节点进行交互，但是为了提高 Master 的冗余，需要将 Master 的状态复制到其他 Master Server 上。

如果当前 Master 节点失效了，处于 GFS 系统之外的监控进程会*选择其他保存完整操作日志的 Mater Server 启动一个新的 Master 进程*；之后 Client 的访问会请求到新的 Master 节点上。

> GFS  中 Master 的主节点选择不是利用 Raft 等一致性算法，而是其他监控进程进行判断选择

Master 节点除了有备份节点，还存在**影子节点**：影子 Master 中的状态会比 Master 主节点慢一些（Master 备份节点与主节点保持同步）；*用于在 Master 的主节点失效之后提供文件系统的只读功能*。

> 影子节点会定时从 Master 主节点拉取最新的操作日志并应用到自身状态中，并且也会定时与 Chunk Server 交互获取 Chunk 信息

## 5.2-Data Integrity

每个 Chunk Server 都会***使用 Checksum 来检查自己保存的数据***是否损坏。不选择通过其他 Chunk 副本来比较检查的原因：

- 跨越 Chunk Server 比较副本来检查数据是否损坏很不实际
- GFS 允许有歧义的 Chunk 副本存在；Chunk 在进行原子追加时，不能保证 Byte 级别的一致性，因此各个 Chunk 副本可能会不一致

每个 Chunk 被划分为 64KB 的块，每个块都对应一个 32 位的 Checksum（保存在内存中，同时也记录在操作日志中）。在将 Chunk 数据返回给 Client 或者其他 Chunk Server 之前，都会根据 Checksum 校验数据是否正确。

如果某一块数据错误，则返回异常，并将这个错误上报到 Master。收到错误之后，***Client 或者其他 Chunk Server 会从其他副本读取数据；Master 会从其他 Chunk 副本克隆数据进行恢复\***。

> 当 Chunk 副本恢复之后，*Master 会通知副本错误的 Chunk Server 删除掉错误副本*

在 Chunk Server 负载较低的时候，它会定期扫描 & 检验每个不活跃的 Chunk 副本的内容。一旦发现存在损坏的 Chunk，就会上报给 Master；从而可以创建一个新的，正确的 Chunk 副本；之后再把损坏的数据删除。

> Checksum 在 Chunk Server 的磁盘中保存；同时也会保存每个 Chunk 的最新版本号

# 6-Conclusions

GFS 设计思路中，组件失效是常态；大文件写入采用 Append 的方式优化性能；通过持续监控，复制关键数据，快速和自动恢复提供灾难冗余。

GFS 并没有在文件系统层面提供任何 Cache 机制，因为 Client 几乎不会重复读取数据，其读取方式要么是流式的读取一个大型的数据集，要么是在大型的数据集中随机 Seek 到某个位置，之后每次读取少量的数据。

分布式文件管理方面，并没有采用一致性算法来保证数据的一致性，而是采用中心节点的方式，这样简化了设计，提高可靠性，可扩展性。

为了实现大量的并发读写操作仍能够提供很高的吞吐量，采用控制流与数据流分离的设计方案。控制流在 Master 处理，数据流在 Client 与 Chunk Server 中处理。为了降低 Master 的负载，选择了较大尺寸的 Chunk；同时，通过 Lease 机制选择 Chunk 主副本，从而将控制权从 Master 转移到主 Chunk Server 中。
