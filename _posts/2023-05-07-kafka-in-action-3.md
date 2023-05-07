---

layout: post
title: "Kafka: Writing Messages to Kafka"
category: "system-design"
author: "kkzhang"
---
# 1. 生产者概述

需要往 Kafka 写消息的场景有：

- 记录用户活动
- 记录度量指标，监控数据
- 保存日志消息
- 与其他应用程序进行异步通信
- 缓冲即将写入到数据库中的数据
- ……

对于这些不同的使用场景，我们需要分析：

- 是否每个消息都很重要
- 是否允许丢失一小部分消息
- 是否可以接受偶尔的消息重复
- 对消息的延迟及吞吐量是否严格要求
- ……

不同的使用场景对生产者 API 的使用和配置都会有直接影响。

![]({{site.baseurl}}/images/kafka/chapter_2/1.png)

上图展示了向 Kafka 发送消息的主要步骤：

> **消息先被放入缓冲区，之后由单独的线程发送到 Broker**
> 

1. 创建 ProducerRecord 对象，包含目标主题，发送的内容，还可以指定 Key，分区
2. 数据被序列化二进制字节数组
3. 数据被传送给分区器
    1. 如果提前指定了分区，那么分区器将不会做任何事情
    2. 如果没有指定分区，那么分区器就会根据 ProducerRecord 对象的 Key 选择一个分区
4. 生产者已经明确应该往哪个主题与分区发送消息了，该记录会被添加到一个记录批次里，该批次里的所有消息都会被发送到同一个主题与分区上
    1. **存在一个独立的线程负责把这些记录批次发送到相应的 Broker 上**
5. 服务器收到消息后会返回一个响应
    1. 如果成功写入 Kafka，则返回一个 RecordMetaData 对象，包含了主题和分区信息，以及记录在分区中的偏移量（Offset）
    2. 如果写入失败，则返回一个错误消息
6. 生产者在收到错误之后就会尝试重新发送消息，几次重试之后如果还是失败，则返回错误信息

# 2. 创建 Kafka 生产者

创建 Kafka 生产者需要设置 3 个必要属性：

1. bootstrap.servers
    
    指定 Broker 的地址列表，格式为 host:port。（该列表不需要包含所有 Broker 地址，生产者会从给定的 Broker 中找到其他 Broker 信息）
    
2. key.serializer
    
    Broker 接收的消息的 key & value 都是字节数组。因此，生产者需要把 key 序列化成字节数组。
    
    > key.serializer 需要实现 org.apache.kafka.common.serialization.Serializer 接口
    > 
3. value.serializer
    
    同 key.serializer
    

创建生产者代码 Demo:

```java
private Properties kafkaProps = new Properties();
kafkaProps.put("bootstrap.servers", "broker1:9092,broker2:9092");
kafkaProps.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
kafkaProps.put("value.serializer",
  "org.apache.kafka.common.serialization.StringSerializer");
producer = new KafkaProducer<String, String>(kafkaProps);
```

# 3. 发送消息到 Kafka 生产者

生产者创建完成后，可以向 Broker 发送消息。有 3 种发送消息的方式：

1. **发送并忘记**（*Fire-and-forget*）
    
    将消息发送给 Broker 服务器，但是并不关心是否正常到达。
    
    > 多数情况下，消息会成功被接收（失败后会自动重试），但是有时候会丢失一些消息。
    > 
2. **同步发送**（*Synchronous send*）
    
    `send()` 方法返回一个 `Future` 对象，调用 `Future.get()` 方法等待，可以判断消失是否成功发送。
    
3. **异步发送**（*Asynchronous send*）
    
    调用 `send()` 方法并指定回调函数，当接收到 Broker 的响应之后，回调函数被触发。
    

简单的发送消息代码如下：

```java
ProducerRecord<String, String> record =
            new ProducerRecord<>("CustomerCountry", "Precision Products",
"France");
try {
		// send() 方法会返回一个包含 RecordMetaData 的 Future 对象
		// 但是我们直接忽略返回值，所以无法感知到消息是否发送成功
	  producer.send(record);
} catch (Exception e) {
		// 我们可以忽略发送消息时可能发生的异常，或者在服务端发生的异常
		// 但是发送消息之前，生产者可能发生其他异常，如序列化异常，缓冲区已满，发送线程被中断等
	  e.printStackTrace();
}
```

## 3.1 同步发送消息（**Sending a Message Synchronously**）

同步发送消息代码如下：

```java
ProducerRecord<String, String> record =
            new ProducerRecord<>("CustomerCountry", "Precision Products", "France");
try {
		// 如果服务器返回错误，get() 方法会抛出异常
		// 如果没有发生错误，get() 方法会返回 RecordMetaData 对象，可以用来获取消息的偏移量
    producer.send(record).get();
} catch (Exception e) {
		e.printStackTrace();
}
```

生产者发送消息一般会发生两类异常：

1. **可重试异常**
    
    可以通过重发消息解决；常见的错误有：连接异常，无主（no leader）异常等。
    
    KafkaProducer 可以设置成自动重试，但是如果多次重试之后仍无法解决问题，会收到一个重试异常。
    
2. **不可重试异常**
    
    对于这类异常，生产者不会进行任何重试，直接抛出异常；常见的错误有：消息太大（message size too large）异常。
    

## 3.2 异步发送消息（**Sending a Message Asynchronously**）

如果每个消息发送之后都需要同步等待响应，那么应用程序的吞吐量将会大大下降。虽然 Kafka 会把目标主题，分区信息，消息偏移量发送回来，但是对于很多发送端的应用程序来说并不是必需的。

**多数情况下，我们不需要等待 Broker 的响应。不过，当消息发送失败时，我们可能需要对异常进行处理分析**，如抛出异常，记录错误日志等。

为了在消息发送异常时进行处理，生产者提供了回调支持。代码如下：

```java
// 需要实现 org.apache.kafka. clients.producer.Callback 接口
private class DemoProducerCallback implements Callback {
    @Override
    public void onCompletion(RecordMetadata recordMetadata, Exception e) {
     if (e != null) {
         e.printStackTrace();
        }
} }

ProducerRecord<String, String> record =
        new ProducerRecord<>("CustomerCountry", "Biomedical Materials", "USA");
producer.send(record, new DemoProducerCallback());
```

# 4. 配置生产者

除了之前提到的 3 个必要参数，生产者还有其他可配置的参数；其中一些参数在内存使用，性能，可靠性方面对生产者影响较大。

## 4.1 acks

**该参数指定了必须要多少个分区副本收到消息，生产者才会认为消息写入是成功的**。

> acks 参数配置影响消息丢失的可能性
> 

1. acks = 0
    
    **发送消息之后，生产者不会等待任何服务器的响应。**
    
    - 如果发送过程中出现了问题，导致服务器没有收到消息，生产者无从得知，消息也会丢失
    - 由于不需要等待服务器响应，发送消息的吞吐量会很大
2. acks = 1
    
    **只要 Leader 副本收到消息，生产者就会收到来自 Broker 的成功响应。**
    
    - 如果消息无法成功写入 Leader 副本（如 Leader 崩溃），那么生产者就会收到一个错误响应；为避免消息丢失，生产者会重试发送。
        
        > 如果 Leader 收到消息后崩溃，并且该消息没有来得及同步到其他副本中，那么消息仍会丢失
        > 
    - 此时的吞吐量取决于是采用同步发送还是异步发送
3. acks = all
    
    **只有当所有同步副本（in-sync replicas）都收到消息时，生产者才会收到来自 Broker 的成功响应**。
    
    - 这种模式可以使得消息发送更可靠，更不易丢失
    - 但是，消息发送的延迟更高（需要等待多个副本收到消息）

## 4.2 buffer.memory

该参数用来设置生产者内存缓冲区的大小；生产者用其缓存将要发送到服务器的消息。

如果应用程序发送消息的速度超过将缓冲区消息发送到服务器的速度，会导致生产者空间不足。此时，`send()` 方法会被阻塞或者抛出异常。

## 4.3 compression.type

默认情况下，消息发送不会被压缩。不过，可以通过该参数指定压缩算法，如 snappy, gzip, lz4。

## 4.4 retries

消息发送时可能会遇到临时且可重试的错误，可以通过 retries 参数配置生产者可重发消息的次数。

> 如果重试次数达到该配置，生产者就会放弃重试并返回异常
> 

## 4.5 batch.size

当多个消息被发送到同一个分区时，生产者会将其放到同一个批次中。该参数指定了一个批次可用内存大小（按字节计算，而不是消息个数）。

> 当批次被填满时，该批次里的消息被发送
> 

## 4.6 linger.ms

该参数指定了生产者在发送批次前等待其他消息加入批次的时间。

当批次被填满或者 linger.ms 达到上限时，将批次消息发送出去。

## 4.7 **max.in.flight.requests.per.connection**

该参数指定**生产者在收到服务器响应之前，可以发送多少个消息**。

> 如果该值为 1，可以保证消息时按照发送的顺序写入服务器，即使发生重试
> 

该参数值设置得较高可以增加内存使用量，同时提高吞吐量，但设置得太高可能会降低吞吐量，因为批处理效率会降低。

## 4.8 **max.request.sizesh**

控制生产者发送请求的大小

## 4.9 顺序保证

**Kafka 可以保证同一个分区内的消息是有序的**。即，如果生产者按照一定的顺序发送消息，Broker 就会按照这个顺序将消息写入分区。

- 如果 `reties > 0, max.in.flight.requests.per.connection > 1`，那么如果第一个批次的消息写入失败，而第二个批次消息写入成功。Broker 会重试第一个批次的消息，如果重试成功，那么两个批次的消息就会反过来。
- 如果 `reties > 0, max.in.flight.requests.per.connection = 1`，那么生产者在尝试发送第一批次消息时，不会有其他消息发送给 Broker。
    
    > 这种配置会严重影响生产者的吞吐量，只有在对消息顺序有严格要求时才能这么配置
    > 

# 5. 分区

在发送消息时，可以指定消息的 key，也可以将 key = null。key 有两个用途：

1. 作为消息的附加信息
2. 用来决定该消息被发送到主题的哪个分区。
    
    > 拥有相同 key 的消息会被写入到同一个分区中
    > 

如果 key == null，并且使用默认的分区器，那么消息将会被随机发送到各个可用的分区上。

如果 key ≠ null，并且使用默认的分区器，那么 Kafka 会先对 key 进行 hash，之后根据 hash 值将消息映射到特定的分区上，使得同一个 key 的消息总会被映射到同一个分区上。

> 在进行映射时，会使用所有的分区，而不是可用的分区。所以，如果分区不可用，写入将出错
> 

需要注意的是，只有在主题分区数量不变的情况下，key 与分区的映射次啊会保持不变。如果分区数发生变化，那么新的消息可能别写入到其他分区上，因此在创建主题时需要把分区规划好。

## 5.1 自定义分区策略

默认分区器能够满足大部分需求，不过也可以使用自定义分区进行消息管理。

```java
public class BananaPartitioner implements Partitioner {

	  public void configure(Map<String, ?> configs) {}
	
	  public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
				List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
				int numPartitions = partitions.size();
				if ((keyBytes == null) || (!(key instanceOf String)))
						throw new InvalidRecordException("We expect all messages to have customer name as key")
					
				// Banana will always go to last partition
				if (((String) key).equals("Banana"))
						return numPartitions; 
				// Other records will get hashed to the rest of the partitions
				return (Math.abs(Utils.murmur2(keyBytes)) % (numPartitions - 1))
	}
}
```



