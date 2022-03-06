---

layout: post
title: "Distributed Systems"
category: "system-design"
author: "kkzhang"
---
> 本文翻译自 Martin Kleppmann 的 Distributed Systems 课程
> [视频链接](https://www.youtube.com/watch?v=UEAMfLPZZhE&list=PLeKd45zvjcDFUEv_ohr_HdUFe97RItdiB); 
> [课件链接](https://www.cl.cam.ac.uk/teaching/2122/ConcDisSys/dist-sys-notes.pdf>)

# 1-Introduction

关于分布式系统，Lamport 给出的定义是：

- a system in which the failure of a computer you didn’t even know existed can render your own computer unusable.
- multiple computers communicating via a network
- trying to achieve some task together
- Consists of “nodes” (computer, phone, car, robot, . . . )

运行在同一个进程中的多个线程共享同一个地址空间，因此单台计算机上的并发也被称为“共享内存并发”（shared-memory concurrency）。在这种模式下，很容易将数据从一个线程传递到另一个线程：比如指针（pointer），变量（variable）。

在分布式系统中，上述场景就有所不同了。分布式系统中虽然仍存在并发（不同的计算机可以并发执行同一个应用程序），但这些计算机通常不会共享内存（每个计算机都有自己的地址空间，访问自己的内存），此时节点之间只能通过网络进行通信（network）。

> 在一些超级计算机和科研系统中存在有限形式的共享内存模型，比如 RDMA（remote directly memory access）技术可以使得不同计算机通过网络互相访问内存；数据库（database）在某种程度上也可以被看作是共享内存模型，只不过与常见的字节寻址模型不同。总体来说，大部分分布式系统都是基于消息传递（message-passing）。
> 

分布式系统中的每个节点被称为 Node。Node 含义比较广泛，可以是任何通信计算设备：如桌面计算机，数据中心的服务器，移动设备，网联车，传感器等。

## 1-1 About distributed systems

为什么要使用分布式系统？有以下原因：

1. 一些应用本质上就是分布式的（intrinsically distributed）：比如手机之间发送短信，该过程不可避免地要求不同手机间通过网络交互
2. 提高可靠性（more reliably）：如果系统中只有一个节点，该节点可能会故障，偶尔需要重启，那么系统就继续提供服务。但是如果系统中存在多个节点，当某个节点重启时，其他节点能够继续提供服务。所以，分布式系统要比单点系统更可靠。
3. 提升系统性能（performance）：假设一个服务被世界各地的用户访问，如果该服务是单点系统，那么总有一些地区的用户会觉得服务响应很慢，体验很差。如果该服务是分布式系统，可以将系统节点分布在世界各地，通过将用户路由到最近的节点以提高访问速度。
4. 提升系统处理能力（solve bigger problems）：随着数据规模增大，处理任务增多，单节点可能无法承载，或者处理速度很慢。通过将任务分散到不同的节点，以提高系统整体处理能力。

然而，分布式系统也存在不足：

1. 网络异常（network failure）：网络异常时，节点间将无法正常通信
2. 节点自身异常（node failure）：节点可能会崩溃（crash），运行过慢，或者由于软件硬件方面的异常导致不符合预期。如果我们想要实现故障转移，我们首先需要能够检测（detect）到故障确实发生了；而准确检测故障并不是一件容易的事
3. 网络故障及节点异常经常会发生，系统需要能够处理这些异常

在单点系统中，如果出现硬件异常，通常我们不会期望服务仍能正常运行，因此不会对这些异常在软件层面做额外的处理。但是在分布式系统中，我们需要容忍部分节点异常：一些节点不可用时，其他节点能够继续提供服务。

我们把系统组件故障称为 `fault`，分布式系统需要能够容忍部分组件故障（ `fault tolerance`）：尽管存在故障，整个系统仍然能够正常运行。

> 不过，故障处理通常比较难
> 

## 1-2 Distributed systems and computer networking

学习分布式系统，通常需要对硬件进行抽象。

我们假设节点间通过某种方式通信，而不关心消息如何被编码，通过何种方式发送等物理层表现方式。

![截屏2022-03-01 下午8.25.05.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/020d3643-4cbc-4782-bccb-59c808898c84/截屏2022-03-01_下午8.25.05.png)

从分布式系统的角度看，我们只需要关注通信延迟（latency）与带宽（boundwith）即可。

- 延迟：消息发送到接收的间隔
- 带宽：单位时间内能够被传输的数据量

分布式系统建立在基础设施之上，更专注于如何进行节点之间的协调，以共同完成任务。分布式系统算法主要关于节点发送什么消息，及节点如何处理收到的消息。

浏览器访问网页是一个我们经常使用的分布式系统例子：

![截屏2022-03-03 上午11.07.12.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8d2a7b2e-56d6-4bf3-bf3a-fa1f253e73ea/截屏2022-03-03_上午11.07.12.png)

该例子中的节点有两种：

1. server：网络站点提供服务
2. client: 浏览器用于展示从站点加载的内容

加载网页时，浏览器发送 HTTP request message 到合适的 server 节点；收到请求后，server 返回携带网页内容的 response message（可能是 HTML 文档，图片，视频等）。

HTTP 协议运行在 TCP 协议之上：当 HTTP 报文较大时，TCP 将 HTTP request 内容切分为适当大小的报文，对于接收到的 response 报文段会重新拼接。HTTP 允许多个 request & response 复用同一个 TCP 链接。

网络通信协议有很多细节，然而当我们从分布式系统的角度观察时，具体细节并不重要：**我们只需要把 request & response 分别看作一条消息，忽略底层网络实现**，使得事情变得更简单（独立于底层网络）。

## 1-3 Example: Remote Procedure Calls (RPC)

使用信用卡进行网络购物是另一个分布式系统的例子。

1. 当我们进行购物付款时，就会通过网络给专门处理信用卡支付的服务发送请求
2. 付款服务反过来会与发行信用卡的银行进行通信，以确保该信用卡能够完成支付

在线购物的支付服务实现的代码可能如下：

```
// Online shop handling customer's card details
Card card = new Card();
card.setCardNumber("1234 5678 8765 4321");
card.setExpiryDate("10/2024");
card.setCVC("123");

Result result = paymentsService.processPayment(card,
   3.99, Currency.GBP);
if (result.isSuccess()) {
   fulfilOrder();
}
```

`processPayment` 方法调用看起来与本地调用的其他方法类似，不过实际上是向 `PaymentService` 发送了一个网络请求并等待响应。`processPayment` 方法需要与发行信用卡的银行进行交互，实现逻辑并不在本地，而是在其他节点。

这种交互方式（一个节点的代码看起来是调用另一个节点的方法）被称为 **Remote Procedure Call（RPC）**。在 Java 中，也被称为 Remote Method Invocation（RMI）。实现 RPC 的软件被称为 RPC Framework，或者 Middleware。

> 并不是所有的中间件都基于 RPC 通信，还存在其他通信模型
> 

当一个服务想要调用运行在其他节点上的方法时，**RPC 框架会在 Client 本地创建 stub 来代替目标方法**。该 stub 的方法签名与真正的方法签名相同，但是并不真正执行方法，而是*将方法参数编码成特定的消息，并将该消息发送到远程节点，请求调用对应方法*。

> 方法参数编码的过程被称为 marshall；存在多种编码格式，比如 JSON
> 

从 RPC Client 发送消息到 RPC Server 可能会使用不同的网络协议（如果使用 HTTP 协议，则被称为 Web Service）。在 Server 端，RPC 框架会解码收到的消息，并通过解析出的方法参数调用目标方法。当调用方法返回时，返回结果同样会被编码成消息发送给 Client。Client 收到后会解码并通过 stub 返回给调用方。因此，对 stub 的调用方来说，方法看起来是在本地执行并返回。

对应在线支付的执行流程如下：

![截屏2022-03-03 下午1.11.12.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7b9413b8-16da-41b8-86e9-332dc9929a53/截屏2022-03-03_下午1.11.12.png)

不过，由于网络或者节点可能会出现故障，RPC 实现时有许多问题需要考虑。

- 方法调用期间，服务崩溃
- 消息传递过程中丢失了
- 消息延迟过大
- 调用出现问题时，能否进行安全重试

当 RPC Client 发送请求后却没有收到响应，Client 并不能确定 Server 是否收到消息。此时应该如何处理？

1. Client 可以进行重试，但是可能会导致请求被执行多次（如多次支付）；并且，我们也无法保证重试的消息能够成功被成功接收
2. Client 也不能长时间等待响应。因此，在实践中，Client 通常在超时之后放弃此次请求

在过去几十年，RPC 发展了很多变体，包括面向对象的中间件（如 CORBA）。如今最常见的 RPC 形式是**通过 HTTP 发送 JSON 格式数据**。这种基于 HTTP 的 API 设计原则被称为**表示状态传输（Representational state transfer）或者 REST**，遵守该设计原则的 API 被认为 RESTful。具体协议包括：

1. 资源（resources）通过 URL 表示
2. 使用标准方法类型的 HTTP 请求来更新资源状态（state），如 POST, PUT

REST 的兴起得益于运行在浏览器上的 JavaScript 代码可以很容易发送该请求。不过，虽然 RESTful API 或者其他以 HTTP 为基础的 RPC 是面向 Web 的，但是也可以用于其他类型的 Client 或者 Server-to-Server 的交互。

Server-to-Server 交互场景在大型企业服务上比较常见：企业服务由于过大且复杂，以至于单台机器无法运行，因此常被拆分为多个服务。每个服务被不同的团队开发 & 管理，不同的服务可能使用不同的开发语言；RPC 框架实现了不同服务之间的交互。

当相互通信的服务使用不同的开发语言时，RPC 框架需要能够转化数据类型，使得被调用方能够理解调用方的请求参数；对于返回值也是同样的。

常见的解决方案是使用 Interface Definition Language（IDL）提供与编程语言无关的 RPC 方法签名。通过 IDL 可以自动生成 marshall & unmarshall 代码，同时可以生成与语言无关的 Server stubs & Client stubs。下面是 gRPC 的 IDL 示例：

```go
message PaymentRequest {
	message Card {
		required string cardNumber = 1;
		optional int32 expiryMonth = 2;
		optional int32 expiryYear = 3;
	}
	
	required Card card = 1;
	required int64 amount = 2;
}

message PaymentStatus { 
	required bool success = 1;
	optional string errorMessage = 2;
}

// service
service PaymentService {
	rpc ProcessPayment(PaymentRequest) returns (PaymentStatus) {}
}
```

# 2-Models of distributed systems

系统模型是对系统中 nodes & network 行为的假设及属性的抽象。为了阐述通用的系统模型，先引用两个经典的模型：

1. The Two Generals Problem
2. The Byzantine Generals Problem

## 2-1 The two generals problem

问题描述：假设有两个将军，每人领导一个军队，并且这两个将军都想占领同一座城市。城市的防御很坚固，如果每次只有一个军队进攻，那么会进攻失败；如果两个军队同时进攻，则能够成功占领城市。因此，这两个将军需要协调他们的进攻计划。

![截屏2022-03-06 上午11.56.59.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/91d6ebbd-355a-49ba-9556-e83ff8d0727f/截屏2022-03-06_上午11.56.59.png)

> 目标：只有在军队 2 进攻的情况下，军队 1 才会发起进攻
> 

不过，由于这两个军队之间存在一定的距离，彼此间通信只能依靠信使。信使需要通过由城市控制的区域，并且有时候会被捕获，那么消息就无法成功送达。对于发送方来说，***除非发送方收到接收方的明确回复，否则也不确定之前发送的消息是否成功被接收***。

> 如果一方没有收到消息，可能是对方确实没有发送消息，也可能是对方发送的所有消息都被截获了。
> 

![截屏2022-03-06 上午11.57.47.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/55eac456-df41-4d62-97aa-2d91adb9bcc9/截屏2022-03-06_上午11.57.47.png)

如何使这两个将军就进攻计划达成一致？对每个将军来说，有两种选择：

1. **承诺在任何情况下都发起进攻**，即使没有收到对方的明确回复
    
    这种情况下，发起进攻的将军可能会冒险独自进攻
    
2. 直到收到对方明确回复才会发起进攻，否则就一直等待
    
    这种情况相当于将问题抛给了对方：接收方需要决定是否等待回复消息的确认，还是回复之后就发起进攻（也可能会独自进攻）
    

> 无法达成共识：只能通过消息传递来获取信息
> 

两将军的问题在于，不管双方交换了多少次消息，也无法确定对方一定能同时进攻。对于分布式系统来说，**一个节点无法准确判断另一个节点的状态；不过通过消息传递的方式，可以得到一些信息**。

上面介绍的在线商店支付流程可以看作是两将军问题的例子：在线商店服务与信用卡支付服务之间通过 RPC 进行消息传递，并且消息可能会丢失；尽管如此，商店只有在商品被支付成功的情况下才会发货，支付服务只有在商品被发货才会进行扣款。

![截屏2022-03-06 下午1.04.11.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/db00a9fe-4a4d-456c-9475-df54d0652c1e/截屏2022-03-06_下午1.04.11.png)

> 目标：只有商品被支付成功的情况下，在线商店才会发送货物
> 

不过，实际上该流程与两将军问题有点不同：

- 支付服务在任何情况下都可以先进行扣款操作，如果在线商店始终无法发送货物，可以再进行退款操作。由于**支付操作可以撤回**，使得在两将军中的问题可以解决。
- 即使由于网络等问题导致在线商店服务暂时无法确定支付状态，可以在连接恢复之后再主动查询确定交易状态。发送方可以稍后再次确认之前的消息送达情况。

## 2-2 The Byzantine generals problem

问题描述：拜占庭将军问题的设定与两将军问题类似。假设多个军队（可以是 3 个以上）需要占领一座城市，不同军队之间仍然通过信使进行通信，只不过此时消息总是能够被成功送达。

![截屏2022-03-06 下午1.49.09.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/d7f92209-e758-43f5-b608-b313bf7c72d7/截屏2022-03-06_下午1.49.09.png)

在拜占庭将军问题中，有些将军可能是叛徒：他们可能会故意误导及迷惑其他将军。我们把叛徒称为 “malicious”，其他将军称为 “honest”。

malicious 可能有以下行为：

![截屏2022-03-06 下午1.56.54.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/45012aa0-ec00-45f9-a16c-aa36fb0f711e/截屏2022-03-06_下午1.56.54.png)

1. 将军 3 分别从将军 1 & 将军 2 收到两条矛盾的消息
    - 将军 1 声明要进攻
    - 将军 2 声明将军 1 想要撤退
2. 将军 3 无法确定将军 2 是否在撒谎
    - 第一种场景，将军 2 在撒谎
    - 第二种场景，由于将军 1 先后声明了两个矛盾的消息，此时将军 2 并没有撒谎

honest 并不知道哪些是 malicious，但是 malicious 之间可以秘密串通协调他们的行动。拜占庭将军问题的目标是需要确保**所有的 honest 对计划达成一致**：不管是进攻还是撤退。

> 根据设定，我们无法确定 malicious 的行动，所能做的只有让 honest 达成一致
> 

然而由于 malicious 的存在及不可预测的通信延迟，使得上述目标很难实现；只有在 malicious 的数目严格少于 $1/3$ 时，拜占庭问题才能得到解决。如果系统中存在 $3*f+1$ 个节点，那么 malicious 数目不能多于 $f$。

> 通过引入密码学会使得拜占庭将军问题变得相对简单，但是问题仍然存在，例如数字签名：将军 2 需要向将军 3 证明将军 1 的声明。
> 

那么拜占庭将军问题是否具有实际意义？

在实际的分布式系统中，通常涉及比较复杂的信任关系。比如，消费者需要信任在线商店真的会发送订单中的商品，尽管消费者在商品无法送达或者其他原因向银行提出异议。但是如果在线商店允许消费者在不支付的情况下就可以下单，这种情况下可能会被欺诈者利用，因此在线商店会假设消费者是潜在的 malicious。另一方面，在线商店的多个服务同属一个数据中心，它们内部间基本互相信任，但是支付服务并不完全信任在线商店服务，因为其他人可能会设立欺诈性的商店或者使用被盗信用卡（不过在线商店服务通常信任支付服务）。最后，我们期望消费者，在线商店，支付服务就任何订单达成一致。

![截屏2022-03-06 下午3.22.28.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7f7713cb-1acc-4b45-b91b-7c9ef06df1a9/截屏2022-03-06_下午3.22.28.png)

> 拜占庭将军问题可以看作是复杂信任关系的简化
> 

在一些分布式系统中，会明确处理存在 malicious 节点的可能性，这类系统被称为 **Byzantine fault tolerant**。近些年来，该观点在区块链及加密货币中逐渐流行起来：即使系统的一些参与者试图欺骗或破坏系统，系统也能提供明确的保证。

## 2-3 Describing nodes and network behavior

当设计分布式算法时，system model 是对可能发生故障的假设。上文描述了两种 system model：

1. Two generals problem: a model of networks
2. Byzantine generals problem: a model of node behavior

在 system model 中传递的消息被捕获可能因为：

1. Network behavior：消息丢失等
2. Node behavior：节点崩溃等
3. Timing behavior：延迟等

### 2-3-1 Network behavior

**不存在完全可靠的网络**：即使是在精心设计且有冗余连接的网络中。
大多数分布式算法都假设网络在一对节点间提供双向通信服务，被称为**点对点通信（point-to-point）**或者**单播通信（unicast）**。在真实的网络中有时确实提供**广播（broadcast）**或**多播（multicast）**通信（*一个数据包同时发送给多个接收者*）。不过，通常来说，假设只存在单播（unicast）是一个比较好的网络模型。

接下来我们可以假设网络的可靠性，大部分算法假设如下：

1. **Reliable** (perfect) links
只有在消息被发送时，才会收到消息。不过，消息可能会被重排序。
2. **Fair-loss** links
消息可能会丢失，重复或重排序。如果继续重试，消息最终会被成功发送。
3. **Arbitrary** links (active adversary)
存在 malicious 节点干扰信息传递（窃听，篡改，丢弃，伪造，重放等）。

> **网络分区**（network partition）：一些连接长时间丢弃或延迟所有消息
> 

我们可以将上述某些类型的网络转换为其他类型，如下图：

![截屏2022-03-06 下午7.59.31.png](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/9fb0c820-12a9-4c0b-b193-25195362b239/截屏2022-03-06_下午7.59.31.png)

比如当前网络类型为 Fair-loss link，通过持续发送丢失的消息直到被成功接收，及在接收方过滤重复的消息，可以将 Fair-loss link 类型转换为 Reliable link 类型。

Fair-loss 类型意味着网络分区（network partition）只会持续一段时间，并不是永久故障；所以消息最终一定会被接收。

> 不过，网络分区期间发送的消息只能等到故障恢复之后才能被接收，而故障的时间可能比较长
> 

之前提到的 TCP 协议在网络数据包层面提供了重试及重复数据过滤的机制，不过 TCP 通常配置了超时时间，在重试了一个确定的时间之后就会放弃重试（一般为 1min）。因此为了应对较长时间的网络分区（network partition），需要在 TCP 之上额外实现一套重试及重复数据过滤机制。

**Arbitrary 类型是互联网通信的准确模型**：每当你通过互联网进行通信时（可能是通过咖啡店的 wifi），网络运营商可以任意操作 & 干扰你的网络数据包。操作网络流量的人也被称为 active adversary。不过，通过使用加密技术***几乎（almost）***可以将 Arbitrary 类型的网络转换为 Fair-loss 类型。**TLS**（Transport Layer Security）协议可以防止 active adversary 进行窃听，篡改，伪造，重放流量等操作。不过，TLS 唯一不能阻止 active adversary 进行丢弃（dropping/blocking）网络包操作。所以，只有在 active adversary 永远不会进行 block 操作时，才能将 Arbitrary 类型的网络转换为 Fair-loss 类型。

Reliable 类型的网络看起来并不是不可实现：**一般来说，只要我们在网络分区期间等待一段时间后进行重试，所有消息都能被成功接收**。不过，我们也需要考虑消息发送者在重试期间崩溃（crash）的情况，这种情况下消息会永远丢失（permanently lost）。

### 2-3-2 Node behavior

假设每个节点运行模型的算法都为下面的一种：

1. **Crash-stop** (fail-stop)
如果节点在任意时刻崩溃，则出现故障。崩溃之后，节点永久性地停止运行。
2. **Crash-recovery** (fail-recovery)
节点可能在任意时刻崩溃，导致内存数据会丢失，不过持久化到磁盘上的数据并不会丢失。节点可能在一段时间后重新恢复运行。
3. **Byzantine** (fail-arbitrary)
如果节点偏离运行的算法，则出现故障（A node is faulty if it deviates from the algorithm）。故障节点的表现可能有：崩溃或者一些恶意行为。

> 不存在故障的节点被称为正常节点（A node that is not faulty is called “correct”）。
> 

在 **Crash-stop** 模型中，我们假设节点崩溃之后，永远不会恢复。

- 对于无法恢复的硬件故障来说，这是一种合理的模型（reasonable）
- 对于软件崩溃，Crash-stop 模型看起来不太符合，因为在节点重启之后，将会恢复。尽管如此，一些算法为了简化问题，仍然将其看作 Crash-stop 模型。在这种情况下，崩溃后恢复的节点被认为是重新加入系统中的新节点。

在 **Crash-recovery** 模型中，允许节点崩溃后重启并恢复运行。当节点崩溃重启时，假设内存中的数据全部丢失，但是持久化到磁盘上的数据都会被保存下来。***该模型对节点崩溃后恢复所需要的时间并没有设定，因此崩溃后的节点可能永远都不会恢复***。

**Byzantine** 模型是最常见的节点行为模型：正如拜占庭将军问题中的描述，*故障节点也许不仅仅是崩溃，也可能以任何方式偏离指定的算法，包括一些恶意行为*。节点服务实现中的错误（bug）也可以归类为拜占庭故障。不过，如果所有节点运行同样的软件，那么这些节点就会有同样的 bug。因此，任何基于拜占庭故障节点数少于 $1/3$ 的算法都无法容忍此类 bug。因此，当涉及到偏离指定算法时，我们通常称为拜占庭问题，而不是 bug。

> 原则上，我们可以尝试使用同一算法的几种不同的实现，不过这并不是比较实际的选择。
> 

在 network behavior 的介绍中，通过使用特定的协议可以将一种网络模型转换为另一种。不过，在 node behavior 中并不适用；比如，为 Crash-recovery 模型设计的算法与 Byzantine 模型有很大不同。

### 2-3-3 Synchrony (timing) assumptions

对于同步（计时）的假设模型如下所示：

1. **Synchronous**
    
    消息延迟不会大于已知的上限。节点以已知的速度执行算法。
    
2. **Partially synchronous**
    
    系统在某些有限（但未知）的时间内是异步的，否则是同步的。
    
3. **Asynchronous**
    
    消息延迟时间不定。节点可以在任意时间暂停执行。完全没有时间保证。
    

> cs 的其他部分使用“同步”和“异步”这两个术语的方式不同
>