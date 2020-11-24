# Kafka 入门

Apache Kafka 是消息引擎系统，也是一个分布式流处理平台

> 维基百科：消息引擎系统是一组规范。企业利用这组规范在不同系统之间传递语义准确的消息，实现松耦合的异步式数据传递。

## 作用

- 削峰填谷
- 应用解耦
- 异步处理

## Kafka 术语

- 消息（Record）：Kafka 是消息引擎，这里的消息就是指 Kafka处理的主要对象。
- 主题（Topic）：主题是承载消息的逻辑容器，在实际使用是用来分区具体的业务。
- 分区（Partition）：一个有序不变的消息队列。每个主题下可以有多个分区。（与 Elasticsearch 中的 Sharding）
- 消息位移（Offset）：表示分区中每条消息的位置信息，是一个单调递增且不变的值。
- 副本（Replica）：Kafka 中同一条消息的能够被拷贝到多个地方提供数据冗余，这些地方就是所谓的副本。副本是在分区成绩下的，即每个分区可配置多个副本实现高可用。
  - 领导者副本（Leader Replica）：外提供服务，这里的对外指的是与客户端程序进行交互。生产者总是向领导者副本写消息；而消费者总是从领导者副本读消息。
  - 追随者副本（Follower Replica）：追随者副本，它只做一件事：向领导者副本发送请求，请求领导者把最新生产的消息发给它，这样它能保持与领导者的同步。通过副本选举实现故障转移。
- 生产者（Producer）：向主题发布新新消息的应用程序。
- 消息者（Consumer）：从主题订阅新消息的应用程序。
- 消费者位移（Consumer Offset）：表征消费者消费进度，每个消费者都有自己的消费者位移。
- 消费者组（Consumer Group）：多个消费者实例共同组成的一个组，同时消费多个分区以实现高吞吐。
- 重平衡（Rebalance）：消费者组内某个消费者实例挂掉后，其他消费者实例自动重新分配订阅主题分区的过程。Rebalance 是 Kafka 消费者端实现高可用的重要手段。
- Broker：服务代理节点，Kafka服务实例。

![](https://raw.githubusercontent.com/dddygin/image-storage/main/blog/image/middleware/kafka/geek/02-01.png)

<center> 图01 Kafka 模型</center>