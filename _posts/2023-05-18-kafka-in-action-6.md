---

layout: post
title: "Kafka: Reliable Data Delivery"
category: "system-design"
author: "kkzhang"
---

# 1. 可靠性保证

Kafka 可以作出如下保证：

- **分区消息的有序性**
    
    > 如果消息 B 在消息 A 之后写入同一个分区，那么消息 B 的偏移量一定比 A 的偏移量大，并且消费者会先读取消息 A，再读取消息 B
    > 
- 只有当消息被写入分区的所有同步（in-sync）副本时（并不一定刷到磁盘上），才会被认为是已经提交的（committed）
    
    > 生产者可以选择不同类型的确认机制，如消息被完全提交时的确认，消息被写入 Leader 副本时的确认，消息被发送到网络时的确认
    > 
- 只要还存在活跃副本，那么***已经提交的消息***就不会丢失
- 消费者***只能消费已经提交的消息***

上述机制可以用来构建可靠的系统，但是仅仅依赖上述机制无法保证系统完全可靠。我们需要在消息存储的可靠性，低延迟，高吞吐量，可用性，一致性等方面作出权衡。

# 2. 复制

**Kafka 的复制机制与分区多副本架构是 Kafka 可靠性保证的核心**。

> 多副本机制可以保证在 Kafka 发生崩溃时，仍能保证消息的持久性
> 

Kafka 主题被划分为多个分区。

1. 分区是基本的数据块，分区存储在单个磁盘上
2. Kafka 可以保证分区里的消息是有序的
3. 分区可以是在线的（available），也可以是离线的（unavailable）
4. 每个分区可以有多个副本，其中一个副本是 Leader 副本，其他副本是 Follower 副本
    1. 所有的消息都直接发送给 Leader 副本，或者直接从 Leader 副本中读取消息
    2. Follower 副本只需要与 Leader 保持同步，并及时从 Leader 复制最新的消息
    3. 当 Leader 副本不可用时，其中一个同步副本会称为新的 Leader 
5. Leader 副本是同步（in-sync）副本，对于 Follower 副本来说，需要满足以下条件才会被认为是同步的：
    1. *与 Zookeeper 之间存在活跃会话，并在 6s 内（可配置）向 Zookeeper 发送过心跳*
    2. *10s 内（可配置）从 Leader 副本处获取过消息*
    3. *10s 内（可配置）从 Leader 副本处获取过最新消息*
    
    > 如果 Follower 副本不能满足以上要求任意一条，那么将会被认为是不同步的（out of sync）。如与 Zookeeper 断开连接，或不再获取消息，或消息滞后了 10s 以上
    > 
    
    一个不同步的副本通过与 Zookeeper 重新建立连接，并且从 Leader 副本处获取最新消息，可以重新变成同步的。
    

滞后的同步副本会导致生产者与消费者变慢

- 在消息被提交(committed)之前，客户端会等待所有的同步副本复制消息

非同步副本虽然也同样滞后，但是并不会对性能产生任何影响

- 因为不再关心其是否接收到消息

**同步（in-sync）副本数目越少，丢失数据的风险越大**。

# 3. Broker 配置

Broker 有 3 个配置参数会影响消息存储的可靠性。

## 3.1 复制系数

> replication.factor
> 

**复制系数为 N，可以保证在 N-1 个 Broker 失效时，仍然可以向主题写入数据或者从主题读取数据**。

> 更高的复制系数带来更高的可用性，可靠性
> 

同时，**复制系数 N 需要至少 N 个 Broker，且有 N 个数据副本。**

> 需要占用 N 倍的磁盘空间
> 

因此，我们需要在可用性与硬件存储之间作出权衡。

## 3.2 不完全的 Leader 选举

> unclean.leader.election.enable 默认为 true
> 

当分区 Leader 不可用时，一个同步（in-sync）副本会被选举成新的 Leader。如果选举过程中没有丢失数据，即**已经提交（committed）的消息存在所有的同步（in-sync）副本上，那么这个选举就是完全的（clean）**。

如果分区 Leader 不可用时，所有副本都是不同步的，此时应当如何处理？参考以下场景：

1. 假设分区有 3 个副本，其中两个 Follower 副本不可用（如 Broker 崩溃）。如果生产者继续往分区 Leader 写入消息，那么所有的消息都会被提交，因为只存在 Leader 副本这一个同步副本。如果一段时间后 Leader 副本也不可用了，同时 Follower 副本重新可用。
2. 假设分区有 3 个副本，其中两个 Follower 副本不同步（如网络延迟导致复制滞后）。Leader 仍在继续接收消息；如果此时 Leader 不可用，那么另外两个 Follower 副本不可能变成同步的。

对于上述场景，我们需要作出选择：

1. ***不同步的 Follower 副本不能被选举为新 Leader ⇒ 可用性降低***
    
    在分区旧 Leader 副本恢复之前，分区不可用，不可用时间不确定
    
2. ***不同步的 Follower 副本可以被选举为新 Leader ⇒ 数据不一致风险***
    
    那么在这个 Follower 副本变得不可用之后，所有写入旧 Leader 副本的消息全部丢失，导致数据不一致（从消费者的角度来看，消费的消息可能会比较混乱）
    

## 3.3 最少同步副本

> min.insync.replicas
> 

在上面的场景描述中，假设一个分区有 3 个副本，那么可能会出现只有一个同步副本的情况。如果该副本不可用，我们需要在可用性与一致性之间作出选择。

如果要确保已提交（committed）的消息不止写入一个同步副本，那么可以把最少同步副本设为较大的值。对于 3 个副本的分区，可以设置 *min.insync.replicas = 2，这样至少需要存在两个同步副本，才能向分区写入数据*。

如果可用的的同步副本数小于 min.insync.replicas 值，那么 *Broker 就会拒绝生产者的写入请求；消费者可以继续读取已经提交的消息。*

> 此时分区变成只读状态
> 

# 4. 在可靠的系统中使用生产者

除了对 Broker 进行配置，也需要对生产者进行可靠性方面的配置，否则仍有数据丢失的风险。

举例如下：

1. 假设分区副本数为 3，禁用不完全选举，参数 acks = 1
    
    生产者将消息发送给 Leader 副本，Leader 写入成功，但是 Follower 副本还未复制该消息。Leader 向生产者返回消息写入成功的响应，随后崩溃。此时消息仍未同步到 Follower 副本，并且 Follower 副本仍被认为是同步的。当其中一个 Follower 副本被选举成新的 Leader 后，之前生产者发送的消息将丢失，并且生产者认为该消息是写入成功的；不过消费者看不到这条丢失的消息（Follower 副本未收到，那么该消息不被认为是已经提交的）。但是从生产者角度来看，确实丢失了消息
    
2. 假设分区副本数为 3，禁用不完全选举，参数 acks = all
    
    生产者发送消息给 Leader 副本，但是由于 Leader 不可用，返回 “Leader not Available” 异常。生产者没有对异常进行处理，也没有重试，那么消息将丢失（不过这并不算 Broker 的可靠性问题）
    

因此，当使用生产者时，我们需要配置恰当的 acks 参数，并正确处理异常。

## 4.1 发送确认

生产者可以选择 3 中确认模式：

1. acks = 0
    
    如果生产者能够通过网络把消息发出去，那就认为消息已经成功写入 Broker。
    
    这种模式下，消息吞吐量非常高，但是很大可能会丢失消息。
    
2. acks = 1
    
    Leader 副本在收到消息并写入分区数据文件（不一定刷新到磁盘上）时，会返回确认或者错误响应。
    
    如果发生正常的 Leader 选举，那么生产者会收到一条 LeaderNotAvailableException 异常。在这种模式下，仍可能会丢失消息：消息成功写入 Leader 副本，但是未及时同步到 Follower 副本，随后 Leader 崩溃。
    
3. acks = all
    
    Leader 副本在返回给生产者确认或者错误的响应之前，会等待所有同步副本都收到消息。
    
    配合 min.insync.replica 参数，可以决定在返回给生产者之前至少有多少个副本能够收到消息。
    
    这个模式最安全，但是吞吐量会降低。
    

## 4.2 错误处理

通常，为了让生产者不丢失消息，最好让生产者在遇到可重试的错误时进行重试。但是，重试会带啦消息重复的风险。

> 重试 + 错误处理可以保证每个消息“至少被保存一次”
> 

生产者内置的重试机制可以处理大部份异常，但是对于不可重试的错误来说，仍需要开发人员手动处理。

# 5. 在可靠的系统中使用消费者

只有已经被提交的消息（写入所有同步副本的数据）才对消费者可见，意味着消费者消费的消息具备一致性。为了保证消费的可靠性，消费者需要做的就是跟踪哪些消息已经被消费过，哪些还未被消费。

- 从分区读取数据时，消费者会获取一批消息，检查这批消息的最大偏移量，然后从这个偏移量处开始消费另一批消息。
- 如果一个消费者退出，另一个消费者需要知道从哪个位置开始继续处理。如果前一个消费者提交了偏移量，但是消息并未全部消费完，那么可能会造成消息丢失。

> 区分已提交消息与已提交偏移量：
已提交消息：已经被写入所有同步副本，并且对消费者可见的消息
已提交偏移量：消费者发送给 Kafka 的偏移量，用于确认它已经收到并且处理好的消息位置
> 

## 5.1 消费者的可靠性配置

为了保证消费者行为的可靠性，需要关注以下参数：

- group.id：如果两个消费者的 group.id 相同，并且订阅了同一个主题，那么每个消费者只会消费主题分区的一个子集。如果想要消费者看到所有消息，需要为其设置唯一的 group.id
- auto.offset.reset：在消费者没有偏移量可以提交时，或者请求的偏移量不存在时，消费者应当如何做；有 earliest 与 latest 两种方式
- enable.auto.commit：消费者提交偏移量的模式：自动提交与手动提交
- auto.com mit.interval.ms：如果配置了自动提交的模式，可以通过该参数配置提交的频率

## 5.2 显示提交偏移量

为了提高消费者的可靠性，我们需要注意以下几点。

1. 总是在消息处理完之后再提交
2. 提交频率是性能与重复消息数量之间的权衡
3. 确保对提交的偏移量有清晰的认知
4. 再均衡
5. 消费者可能需要重试
6. 消费者可能需要维护状态
7. 长时间处理：注意消费者需要一直保持轮询
8. 仅一次传递：需要外部系统支持

