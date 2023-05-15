---

layout: post
title: "Kafka: Reading Data from Kafka"
category: "system-design"
author: "kkzhang"
---

# 1. KafkaConsumer 概念

## 1.1 消费者与消费者组

Kafka 消费者从属于**消费者群组（Consumer Group）**。一个群组里的消费者订阅的是同一个主题，每个消费者消费该主题的一部分分区消息。
![]({{site.baseurl}}/images/kafka/chapter_4/1.png)

![]({{site.baseurl}}/images/kafka/chapter_4/2.png)

![]({{site.baseurl}}/images/kafka/chapter_4/3.png)

![]({{site.baseurl}}/images/kafka/chapter_4/4.png)

如上所示，主题 Topic 1 有 4 个分区，消费者群组 Consumer Group 1 订阅该主题：

1. 消费者群组中只有一个消费者 C1，该消费者将接收 4 个分区的消息
2. 消费者群组中添加一个消费者 C2，则 C1, C2 分别接收 2 个分区的消息
3. 如果群组中有 4 个消费者，则每个消费者分别接收 1 个分区的消息
4. 如果群组中的消费者继续增加，超过了主题分区的数量，那么会有一部分消费者被闲置，无法接收任何分区的消息

通过**往消费者群组中添加消费者，是横向扩展消费能力的主要方式**。但是，**消费者的数量不要超过分区数**，不然会有部份消费者无法消费消息。

- 单个消费者有时候无法跟上数据生产的速度，此时可以增加消费者数目，让每个消费者分担部份分区的消息
- 群组中的每个消费者只承担部份分区的消费

同时，Kafka 还支持多个消费组订阅同一个主题，每个消费组都可以接收全部主题消息，且互不影响。

- 该设计适用于多个不同的应用程序订阅同一个主题的场景

![]({{site.baseurl}}/images/kafka/chapter_4/5.png)

**不管是往消费组中添加消费者，还是新增新的消费组，都不会对 Kafka 的性能造成负面影响**。

## 1.2 消费者群组与分区再均衡

### 1.2.1 Rebalance

主题分区所有权由一个消费者转移到另一个消费者，被称为**再均衡**。

以下场景会发生再均衡操作：

- **新的消费者加入群组**：新增的消费者会读取原本属于其他消费者的分区消息
- **消费者（关闭或者发生崩溃）离开群组**：该消费者负责的分区将由其他消费者处理
- **主题分区发生变化**：由管理员新增或者删除分区

再均衡可以使得消费者群组具有更好的可用性与伸缩性，但是**再均衡期间，消费者将无法消费消息**，使得整个群组短时间不可用。

> 需要避免不必要的再均衡操作
> 

### 1.2.2 Consumer Group

消费者通过向**被指派**为**群组协调器**（group coordinator ）的 Broker 发送心跳（heartbeats）来维持**消费者与群组的从属关系**以及**消费者与分区的所有权关系**。

> 不同的消费者群组可以有不同的群组协调器
> 
- 如果消费者在指定时间间隔内正常发送心跳，则会被认为是活跃的
- 如果消费者停止发送心跳的时间较长，会话过期，群组协调器就会认为消费者已经死亡，触发再均衡

**消费者会在消息轮询或者提交消息偏移量时发送心跳**。

消费者退出群组分为两种场景：

1. ***消费者崩溃退出***
    - 消费者崩溃，并停止读取消息。协调器并不会立即触发再均衡，而是等待几秒，确认消费者已经死亡，之后再均衡
2. ***消费者主动退出***
    - 消费者会通知协调器其将要离开群组，那么协调器就会立即触发一次再均衡

> 在 0.10.1 版本中，Kafka 引入一个独立的心跳线程，可以在消息轮询间隙发送心跳；使得发送心跳的频率与消息轮询的频率（与处理消息花费的时间有关）相互独立。
> 

### 1.2.3 分区分配过程

1. 当消费者加入群组时，会向群组协调器发送一个 JoinGroup 的请求。
    - 第一个加入群组的消费者会成为群组的 Leader
2. 群组 Leader 从协调器获得群组的成员列表
    - 最近向协调器发送过心跳，被认为是活跃的消费者列表
3. 群组 Leader 为每个消费者分配分区子集
    - 分配策略可以自定义实现，也可以使用 Kafka 内置策略
4. 群组 Leader 把分配结果发送给群组协调器
5. 协调器将这些信息发送给所有消费者
    - 每个消费者只能看到自己的分配结果；群组 Leader 知道所有分配结果
6. 再均衡发生时，重复上述流程

# 2. 创建 Kafka 消费者

创建 KafkaConsumer 对象也需要 3 个必要属性：

- bootstrap.servers：指定 Kafka 集群连接字符串
- key.deserializer, value.deserializer：反序列化器，用于将字节数组转换成 java 对象
- grop.id：指定了消费者属于哪个消费者组（非必需）；如果不指定，则不属于任何消费者组

```java
Properties props = new Properties();
props.put("bootstrap.servers", "broker1:9092,broker2:9092");
props.put("group.id", "CountryCounter");
props.put("key.deserializer",
  "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer",
  "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<String,
String>(props);
```

# 3. 订阅主题

使用 `subcribe()` 方法订阅主题：

```java
// 1. 订阅单个主题
consumer.subscribe(Collections.singletonList("customerCountries"));
// 2. 通过正则表达式订阅多个主题
// 当有新的主题添加时，立即触发 rebalance，消费者就可以消费新的主题
consumer.subscribe("test.*");
```

# 4. 轮询

消息轮询方法不仅能够获取消息，还包含其他操作：**群组协调，分区 Rebalance，发送心跳**等。

- 当首次调用消费者的 poll 方法时，会查找群组协调器，加入群组，并接收被分配的分区
- 如果发生了 Rebalance，也需要在轮询过程中处理
- 心跳发送也是在轮询过程中发送的

> 我们需要确保轮询期间的操作尽快结束
> 

```java
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records)
        {
            log.debug("topic = %s, partition = %s, offset = %d,
               customer = %s, country = %s\n",
               record.topic(), record.partition(), record.offset(),
               record.key(), record.value());
            int updatedCount = 1;
            if (custCountryMap.countainsValue(record.value())) {
                updatedCount = custCountryMap.get(record.value()) + 1;
            }
            custCountryMap.put(record.value(), updatedCount)
            JSONObject json = new JSONObject(custCountryMap);
            System.out.println(json.toString(4))
        }
}
} finally {
    consumer.close();
}
```

Consumer 必须持续轮训 Kafka Broker，否则会被认为已经死亡，触发 Rebalance。

- `poll()` 方法返回记录列表，包含：主题信息，分区信息，偏移量，记录的 key-value 对
- `close()` 方法用于关闭消费者，除了关闭网络连接，也会触发一次 Rebalance（而不是等待消费者协调器去判断是否仍存活）

**在一个消费者群组中，无法让一个线程运行多个消费者，也无法让多个线程共享一个消费者。安全起见，需要让一个线程运行一个消费者**。

# 5. 消费者配置

1. **fetch.min.bytes**
    
    该参数指定了从 Broker 获取消息记录时的最小字节数。
    
    - Broker 收到消费者请求时，如果当前可用数据量小于该值，则会等待有足够多的数据时才返回
    - 适当调大该值，可以降低 Broker 负载：Broker 不需要频繁处理请求
2. **fetch.max.wait.ms**
    
    该参数用于指定 Broker 最长等待时间。
    
    - 当请求的数据量得不到满足时，最多等待该参数指定的时间
3. **max.partition.fetch.bytes**
    
    该参数指定 Broker 从每个分区返回给消费者的最大字节数。
    
    - 该参数值的设置需要考虑消费者处理数据的时间。消费者需要频繁调用 `poll()` 方法来维持会话，如果返回的数据量过多，数据处理时间较长，可能无法及时进行下次 `poll()` 导致会话过期。
4. **session.timeout.ms**
    
    消费者于 Broker 之间会话过期时间，默认 3 秒。
    
    - 如果消费者在该时间范围内没有向 Broker 发送心跳，则会被认为已经死亡，触发 Rebalance
    - 该参数表明消费者可以多长时间不发送心跳；另一个参数 heartbeat.interval.ms 则指定了 `poll()` 方法发送心跳的频率。这两个参数通常配合使用，heartbeat.interval.ms 一般为 session.timeout.ms 的 1/3
    - 该参数值如果设置的比较小，有利于更快监测和恢复崩溃的节点，但是可能会导致非预期内的 Rebalance
5. **auto.offset.reset**
    
    该参数指定了消费者在读取一个**没有偏移量**的分区或者**偏移量无效**的情况下的处理方式
    
    - 默认是 latest: 从最新记录处开始重新消费
    - earliest: 从分区起始位置开始重新消费
6. **enable.auto.commit**
    
    该参数用于配置消费者是否自动提交偏移量，默认 true.
    
    - 如果设置为自动提交，可以配置 auto.commit.interval.ms 用来控制提交间隔
7. **partition.assignment.strategy**
    
    用于指定分区分配给消费者的策略。默认有两种策略：
    
    1. **Range**
        
        把主题若干连续分区分配给消费者。假设有 5 个分区，2 个消费者，那么其中一个消费者被分配 [0,1,2] 分区，另一被分配 [3,4] 分区
        
    2. **RoundRobin**
        
        把主题的所有分区逐个分配给消费者。假设有 5 个分区，2 个消费者，那么其中一个消费者被分配 [0,2,4] 分区，另一被分配 [1,3] 分区
        
8. **max.poll.records**
    
    用于控制单次 `poll()` 调用能够返回的消息数量。
    
9. **receive.buffer.bytes & send.buffer.bytes**
    
    用于设置 Sockets 在读写数据时用到的 TCP 缓冲区大小。
    
    - 默认为操作系统的默认值
    - 如果消费者或者生产者与 Broker 位于不同的数据中心，可以适当提高缓冲区大小，因为网络通信延迟会比较高

# 6. 提交偏移量

当调用 `poll()` 方法时，会返回已经写入 Kafka Broker 但是消费者还未读取的记录。

> Kafka 消费消息不需要得到消费者的确认，相反，消费者可以追踪消息在分区的位置（偏移量）
> 

**更新分区当前位置的操作被称为提交偏移量**。

**消费者如何提交偏移量？**

- 消费者通过往 **__consumer_offsets** 特殊主题发送消息，消息中包含每个分区的偏移量

**偏移量的用途？**

- 如果消费者一直稳定运行消费，那么偏移量没有什么用处
- 如果发生 Rebalance（消费者崩溃或者有新的消费者加入群组），那么消费者可能会被分配新的分区。为了能够继续按照之前的位置继续消费，**消费者需要读取每个分区最后提交的偏移量，然后从偏移量的位置继续处理**。

**发生 Rebalance 时，提交的偏移量可能带来的影响？**

1. 如果最近一次提交的偏移量小于消费者处理的最后一个消息的偏移量，那么两个偏移量之间的消息会被重复消费
    
    ![]({{site.baseurl}}/images/kafka/chapter_4/6.png)
    
2. 如果最近一次提交的偏移量大于消费者处理的最后一个消息的偏移量，那么两个偏移量之间的消息会被丢失
    
    ![]({{site.baseurl}}/images/kafka/chapter_4/7.png)
    

Kafka 提供了多种方式用来提交偏移量。

## 6.1 自动提交

如果配置 enable.auto.commit=true，那么默认每隔 5 秒，**消费者会自动把 `poll()` 返回的最大偏移量提交**。

> 可以设置 auto.commit.interval.ms 参数调整提交间隔
> 

自动提交是在 `poll()` 中处理的

- 当消费者每次轮询时，都会检查是否该提交偏移量了；如果可以提交，则将上一次 `poll()` 返回的最大偏移量进行提交

自动提交偏移量可能带来的影响

1. 上次提交之后发生 Rebalance，导致**消息被重复处理**
    
    假设提交间隔为 5s，如果在上次提交之后的 3s 发生 Rebalance，新的消费者会从最后一次提交的偏移量位置开始消费，那么这 3s 内的消息会被重复消费。
    
    > *这种情况不能完全避免*
    > 
2. 自动提交时，上次轮询的消息未被处理完，如果消费者之后发生崩溃，可能会导致消息丢失
    
    每次调用 `poll()` 时都会检查是否提交上次返回的最大偏移量，但是并不知道上次返回的消息是否全部被处理完成。因此，需要确保再次调用 `poll()` 时，之前的消息已经被处理完。
    

> `close()` 调用也会自动提交偏移量
> 

## 6.2 提交当前偏移量

如果配置 auto.commit.offset=false，可以**使用 `commitSync()` 方法手动提交当前偏移量**。

> `commitSync()` 会提交 `poll()` 返回的最新偏移量，提交成功则返回，否则抛出异常。
> 

> 只要没有发生不可恢复的错误，`commitSync()` 会一直重试直到成功。
> 

> `commitSync()` 是同步方法，在 Broker 响应之前，会一直阻塞。
> 

在调用 `commitSync()` 提交最新偏移量之前，需要确保 `poll()` 返回的消息被处理完成；否则，如果发生 Rebalance，仍会有消息丢失的风险。

下面是处理完消息之后手动提交偏移量的代码：

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records)
    {
        System.out.printf("topic = %s, partition = %s, offset =
          %d, customer = %s, country = %s\n",
             record.topic(), record.partition(), record.offset(), record.key(), record.value());
} try {
		// 手动提交偏移量
    consumer.commitSync();
  } catch (CommitFailedException e) {
    log.error("commit failed", e)
  }
}
```

## 6.3 异步提交

由于 `commitSync()` 方法会使得消费者被阻塞，使得整体tun tu lia

> `commitAsync()` 提交之后，不会等待 Broker 的响应；并且，该方法不会重试提交
> 

不过，这种方式在发生 Rebalance 之后可能会导致消息被重复消费。

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records)
    {
        System.out.printf("topic = %s, partition = %s,
        offset = %d, customer = %s, country = %s\n",
        record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
}
    consumer.commitAsync();
}
```

`commitAsync()` 支持回调（当 Broker 响应时被触发），可以在回调方法中进行异常重试。

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("topic = %s, partition = %s,
        offset = %d, customer = %s, country = %s\n",
        record.topic(), record.partition(), record.offset(),
        record.key(), record.value());
    }

    consumer.commitAsync(new OffsetCommitCallback() {
        public void onComplete(Map<TopicPartition,
        OffsetAndMetadata> offsets, Exception exception) {
            if (e != null)
                log.error("Commit failed for offsets {}", offsets, e);
} });
}
```

## 6.4 同步与异步提交组合

组合使用 `commitSync()` 与 `commitAsync()` ，确保在消费者关闭前提交成功。

```java
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s, partition = %s, offset = %d,
            customer = %s, country = %s\n",
            record.topic(), record.partition(),
            record.offset(), record.key(), record.value());
			}
        consumer.commitAsync();
    }
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try {
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```

## 6.5 提交特定的偏移量

`commitSync()` 与 `commitAsync()` 方法只能提交 `poll()` 返回消息的最大偏移量，如果想要提交该批次中间的偏移量，可以将分区及偏移量作为参数传入。

```java
private Map<TopicPartition, OffsetAndMetadata> currentOffsets =
        new HashMap<>();
    int count = 0;
    ....
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(100);
        for (ConsumerRecord<String, String> record : records)
        {
            System.out.printf("topic = %s, partition = %s, offset = %d,
            customer = %s, country = %s\n",
            record.topic(), record.partition(), record.offset(),
            record.key(), record.value());
            currentOffsets.put(new TopicPartition(record.topic(),
            record.partition()), new
            OffsetAndMetadata(record.offset()+1, "no metadata"));
            if (count % 1000 == 0)
								// 提交特定的偏移量
                consumer.commitAsync(currentOffsets, null);
            count++;
		} 
}
```

# 7. Rebalance 监听器

消费者在退出或者 Rebalance 之前可能需要做一些清理工作，如关闭文件，数据库连接等。消费者在调用 `subscribe()` 监听主题时，可以传入 `ConsumerRebalanceListener` 实例对 Rebalance 操作进行监听。

`ConsumerRebalanceListener` 接口有两个方法需要实现：

```java
// 该方法会在 Rebalance 之前，消费者停止读取消息之后调用
// 如果在此时提交偏移量，那么新的消费者就可以知道上一次消费的位置
public void onPartitionsRevoked(Collection<TopicPartition> partitions)
```

```java
// 该方法会在 Rebalance 之后，消费者开始读取消息之前调用
public void onPartitionsAssigned(Collection<TopicPartition> partitions)
```

监听示例：

```java
private Map<TopicPartition, OffsetAndMetadata> currentOffsets = new HashMap<>();

private class HandleRebalance implements ConsumerRebalanceListener {
		
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {}

		// 在失去分区所有权之前，提交已经处理过的消息偏移量，而非最大偏移量
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("Lost partitions in rebalance. Committing current offsets:" + currentOffsets);
        consumer.commitSync(currentOffsets);
    }
} 

try {
		consumer.subscribe(topics, new HandleRebalance());
    while (true) {
        ConsumerRecords<String, String> records =
          consumer.poll(100);
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s, partition = %s, offset = %d,
             customer = %s, country = %s\n",
             record.topic(), record.partition(), record.offset(),
             record.key(), record.value());
             currentOffsets.put(new TopicPartition(record.topic(),
             record.partition()), new
             OffsetAndMetadata(record.offset()+1, "no metadata"));
					}
        consumer.commitAsync(currentOffsets, null);
    }
} catch (WakeupException e) {
    // ignore, we're closing
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try {
        consumer.commitSync(currentOffsets);
    } finally {
        consumer.close();
        System.out.println("Closed consumer and we are done");
    }
}
```

# 8. 从特定偏移量处开始消费

`poll()` 方法可以从分区的最新偏移量处开始消费消息，如果想从分区的起始位置开始消费，可以使用 `seekToBeginning(TopicPartition tp)` 方法；如果想从分区的末尾位置开始消费，可以使用 `seekToEnd(TopicPartition tp)` 方法。

同时，Kafka 还提供 `seek(TopicPartition partition, long offset)` 方法从特定偏移量处开始消费。

> seek() 方法只是用来更新当前正在消费的位置
> 

下面示例使用 `ConsumerRebalanceLister` 与 `seek()` 方法实现在分区分配时，从本地 DB 中获取特定的 Offset 并消费。

```java
public class SaveOffsetsOnRebalance implements ConsumerRebalanceListener {
      public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
          commitDBTransaction();
			}
      public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
	        for(TopicPartition partition: partitions)
	            consumer.seek(partition, getOffsetFromDB(partition));
			} 
}

consumer.subscribe(topics, new SaveOffsetOnRebalance(consumer));
consumer.poll(0);

// 更新各个 Partition 的消费位点
for (TopicPartition partition: consumer.assignment())
  consumer.seek(partition, getOffsetFromDB(partition));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(100);
    for (ConsumerRecord<String, String> record : records) {
        processRecord(record);
        storeRecordInDB(record);
        storeOffsetInDB(record.topic(), record.partition(), record.offset());
    }
    commitDBTransaction();
}
```

# 9. 如何退出

目前的消费者示例中，消费者都是在无限循环中，如果要退出循环，需要通过**另一个线程**调用 `consumer.wakeup()` 方法。

> `consumer.wakeup()` 是消费者唯一一个可以从其他线程安全调用的方法
> 

调用 `consumer.wakeup()` 可以退出 `poll()`，并抛出 `WakeupException` 异常。

> 如果在调用 `consumer.wakeup()` 时，消费者线程并没有等待轮询，那么异常将会在下一次调用 `poll()` 时抛出
> 

需要注意，在消费者退出之前，有必要调用 `consumer.close()` 方法，该方法会提交还未提交的信息，并向群组协调器发送离开群组的消息，触发 Rebalance。

# 10. 独立消费者

如果一个消费者只是想从主题的所有分区或者特定分区读取消息，并不需要消费者组和 Rebalance。这样的话就不需要订阅主题，取而代之的是**为自己分配分区**。

> 订阅主题（并加入消费者群组），为自己分配分区两件事不能同时执行
> 

消费者为自己分配分区并消费消息的示例：

```java
List<PartitionInfo> partitionInfos = null; partitionInfos = consumer.partitionsFor("topic"); 
if (partitionInfos != null) {
  for (PartitionInfo partition : partitionInfos)
      partitions.add(new TopicPartition(partition.topic(), partition.partition()));
			consumer.assign(partitions); 

	while (true) {
	    ConsumerRecords<String, String> records = consumer.poll(1000);
	    for (ConsumerRecord<String, String> record: records) {
	        System.out.println("topic = %s, partition = %s, offset = %d,
	          customer = %s, country = %s\n",
	          record.topic(), record.partition(), record.offset(),
	          record.key(), record.value());
			}
	    consumer.commitSync();
  }
}
```



