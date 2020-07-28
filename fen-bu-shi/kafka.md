# Kafka

Kafka是分布式发布-订阅消息系统。与其他MQ相比较，Kafka有一些优缺点，主要如下，

## 优点：

可扩展：Kafka集群可以透明的扩展，增加新的服务器进集群。

高性能：Kafka性能远超过传统的ActiveMQ、RabbitMQ等，Kafka支持Batch操作。

容错性：Kafka每个Partition数据会复制到几台服务器，当某个Broker失效时，Zookeeper将通知生产者和消费者从而使用其他的Broker。

## 缺点：

重复消息：Kafka保证每条消息至少送达一次，虽然几率很小，但一条消息可能被送达多次。

消息乱序：Kafka某一个固定的Partition内部的消息是保证有序的，如果一个Topic有多个Partition，partition之间的消息送达不保证有序。

复杂性：Kafka需要Zookeeper的支持，Topic一般需要人工创建，部署和维护比一般MQ成本更高。

基本概念：

1. Topic：特指Kafka处理的消息源（feeds of messages）的不同分类。
2. Partition：Topic物理上的分组，一个topic可以分为多个partition，每个partition是一个有序的队列。partition中的每条消息都会被分配一个有序的id（offset）。
3. Message：消息，是通信的基本单位，每个producer可以向一个topic（主题）发布一些消息。
4. Producers：消息和数据生产者，向Kafka的一个topic发布消息的过程叫做producers。
5. Consumers：消息和数据消费者，订阅topics并处理其发布的消息的过程叫做consumers。
6. Broker：缓存代理，Kafka集群中的一台或多台服务器统称为broker。

消息发送的流程：

[![](http://www.biaodianfu.com/wp-content/uploads/2014/08/message.png "message")](http://www.biaodianfu.com/wp-content/uploads/2014/08/message.png)

1. Producer根据指定的partition方法（round-robin、hash等），将消息发布到指定topic的partition里面
2. kafka集群接收到Producer发过来的消息后，将其持久化到硬盘，并保留消息指定时长（可配置），而不关注消息是否被消费。
3. Consumer从kafka集群pull数据，并控制获取消息的offset



