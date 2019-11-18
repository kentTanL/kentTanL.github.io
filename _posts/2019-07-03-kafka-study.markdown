---
layout:     post
title:      "Kafka 学习与使用总结"
subtitle:   "Kafka 学习与使用总结"
date:       2019-07-03 00:08:28
author:     "Kent"
header-img: "img/post-bg-2016.jpg"
catalog: true
tags:
    - Kafka
---

* TOC
{:toc}

# 一、Kafka 简介

kafka 是一个分布式流处理平台，主要适用于以下场景：
1. 构造实时流数据管道，它可以在系统或应用之间可靠地获取数据。 (相当于message queue) ；
2. 构建实时流式应用程序，对这些流数据进行转换或者影响。 （就是流处理，通过 kafka stream topic 和 topic 之间内部进行变化）。

有如下特点：
- Kafka作为一个集群，运行在一台或者多台服务器上；
- Kafka 通过 *topic* 对存储的流数据进行分类；
- 每条记录中包含一个 key，一个 value 和一个 timestamp（时间戳）。

提供有 4 个核心的 API：
- The [Producer API](http://kafka.apachecn.org/documentation.html#producerapi) 允许一个应用程序发布一串流式的数据到一个或者多个Kafka topic。             
- The [Consumer API](http://kafka.apachecn.org/documentation.html#consumerapi) 允许一个应用程序订阅一个或多个 topic ，并且对发布给他们的流式数据进行处理。             
- The [Streams API](http://kafka.apachecn.org/documentation/streams) 允许一个应用程序作为一个*流处理器*，消费一个或者多个topic产生的输入流，然后生产一个输出流到一个或多个topic中去，在输入输出流中进行有效的转换。             
- The [Connector API](http://kafka.apachecn.org/documentation.html#connect) 允许构建并运行可重用的生产者或者消费者，将Kafka topics连接到已存在的应用程序或者数据系统。比如，连接到一个关系型数据库，捕捉表（table）的所有变更内容。 

# 二、Kafka 组件

Kafka 集群中的单台服务器被称为 Broker，Broker 中包含了多个 Partition。Partition 是一个有序的队列，消息最终将写入 Partition ，同时它也是 Topic 在物理上的分组。 每个 Topic 代表一类消息，一个 Topic 包含一个或多个 Partition，与数据库表类似，用户在发送或读取消息时需要指定具体的 Topic。Producer  意为生产者，代表着用户发送消息的一端，与此对应的是 Consumer，意为消费者，即接收消息并做出相应处理的一端。

![image](/img/2019-07-03-kafka-study/1.jpg)

## Broker
每台 Kafka 服务器被称为 Broker，多个 Broker 组成了 Kafka 集群。

## Topic & Partition  
Topic 就是数据主题，是数据记录发布的地方，可以用来区分业务系统。Kafka 中的 Topics 总是多订阅者模式，一个 Topic 可以拥有一个或者多个消费者来订阅它的数据。


对于每一个 topic， Kafka 集群都会维持一个分区日志，如下所示： 

![image](/img/2019-07-03-kafka-study/2.jpg)

每个 Partition  都是有序且顺序不可变的记录集，并且不断地追加到结构化的 commit log 文件。Partition  中的每一个记录都会分配一个 id 号来表示顺序，我们称之为 offset，offset 用来作为 Partition  中每一条记录的唯一标识。 

### segment 
Partition  是一个有序的队列，每个 Partition 都会被映射到一个逻辑的日志文件之上。这个逻辑的日志文件由一个或多个被称为 segment 的文件组成，分成多个 segment 文件可以有效的控制单个物理文件的大小。同时因为 Partition 中的消息是有序的，所以每当一个消息被发送到一个 Partition 之上时，它都会被追加到该 Partition 的逻辑文件中的最后一个 segment  的末尾，如上图所示。

### offset
事实上，在每一个消费者中唯一保存的元数据是 offset（偏移量），即消费者当前消费的消息在 log 中的位置。offset 由消费者控制，通常在读取记录后，消费者会以线性的方式增加偏移量，但是实际上，由于这个位置由消费者控制，所以消费者可以采用任何顺序来消费记录。例如，一个消费者可以重置到一个旧的偏移量，从而重新处理过去的数据；也可以跳过最近的记录，从"现在"开始消费。 

![image](/img/2019-07-03-kafka-study/3.png)

## Producer
Producer 意为生产者，代表着用户发送消息的一方。用户在 Producer 一端通过指定具体的 Topic 来发送消息，默认的情况下消息会被随机发送到该 Topic 下的某个 Partition ，当然我们也可以通过指定具体 Partition 编号，来将消息发送到指定的 Partition 中，这也是保证消息顺序的一种方式。

## Consuemr
Consuemr 意为消费者，代表着用户接收消息并做出相应处理的一方。每个 Consumer 都会属于一个特定的 Consumer Group，而一个 Consumer Group 则可能包含一个或多个 Consumer，这样子区分有以下几个主要的特点：

1. 正常情况下，同一条消息只会被发送到同一个 Consumer Group 中的其中一个 Consumer；
2. Kafka 为每个 Consumer Group 维护了一个 offset，用于记录该 Consumer Group 的消费位置；
3. 可以指定 Consumer Group 下的不同 Consumer 消费不同的 Partition 下的消息；

![image](/img/2019-07-03-kafka-study/4.png)

## Zookeeper

Zookeeper 作为 Kafka 集群管理的第三方中间件，其主要作用包括：

1. Leader 选举；
2. 在 Consumer Group 发生变化时进行 Rebalance；
3. 持久化 Kafka 维护的 Consumer Group 对应的 offset 值；
4. 其它；



# 三、问题与应用

## 消息丢失的问题

在旧的版本中，Producer 的发送类型分为同步与异步两种，通过参数 `producer.type=[sync | async]`进行设置。而在新的版本中去掉了`producer.type`参数，改用以下方式可得到相同的效果。

在新版本的 Kafka 中，批处理是提升性能的一个主要驱动，为了允许批量处理，kafka 生产者会尝试在内存中汇总数据，并用一次请求批次提交信息。通过设置`linger.ms`和`batch.size`等参数可控制请求间隔时间以及批处理大小等（详情参见：http://kafka.apachecn.org/documentation.html#producerconfigs）。当设置`linger.ms=0`时将立即发送消息（默认为 0），或者设置`batch.size=0`以禁用批处理。

而在使用 Producer API 发送消息时，使用的是异步发送消息方法，它将在确认发送时调用回掉函数，示例如下：

```java
ProducerRecord<byte[],byte[]> record = new ProducerRecord<byte[],byte[]>("the-topic", "key".getBytes(), "value".getBytes());
producer.send(record,
                new Callback() {
                     public void onCompletion(RecordMetadata metadata, Exception e) {
                         // doing something 
                     }
                });
```

注：即使异步的情形下，发送到同一分区的记录的回调也会保证按顺序执行。

`producer.send`方法返回一个`java.util.concurrent.Future<RecordMetadata>`对象，通过调用`Future#get()`方法可模拟同步阻塞的效果。

```java
producer.send（record).get()
```

针对以上情形，可得到以下两种主要的情形：
1. 当使用同步阻塞以及禁用批处理发送消息时，Producer 将会等待回调函数执行后，再继续执行，且消息为单条发送；
2. 当使用异步非阻塞且未启用批处理发送消息时，Producer 在调用完发送消息的方法后，将立即执行后续程序，且消息为批量发送；

同时为了保证发送消息的可靠性，在 Producer 端可通过参数`acks`配置在确认一个请求发送完成之前需要收到的反馈信息的数量，其中：

- `acks=0` 如果设置为0，则 producer 不会等待服务器的反馈。该消息会被立刻添加到 socket buffer 中并认为已经发送完成。在这种情况下，服务器是否收到请求是没法保证的，并且参数`retries`也不会生效（因为客户端无法获得失败信息）。每个记录返回的 offset 总是被设置为-1。
- `acks=1` 如果设置为1，leader节点会将记录写入本地日志，并且在所有 follower 节点反馈之前就先确认成功。在这种情况下，如果 leader 节点在接收消息之后，并且在 follower 节点复制数据完成之前产生错误，则这条消息会丢失。
- `acks=all || -1` 如果设置为all，这就意味着 leader 节点会等待所有同步中的副本确认之后再确认这条记录是否发送完成。只要至少有一个同步副本存在，记录就不会丢失。这种方式是对请求传递的最有效保证。acks=-1与acks=all是等效的。

基于以上配置情形，有如下情形可能发生消息丢失：

1. 当异步发送消息时， Producer  不会等待服务器的反馈，如果网络发生异常或其它情况，则可能会丢失消息；
2. 当异步批量发送消息时，如果 Producer  down 掉，则缓冲区消息可能丢失；
3. 当`acks=0`时，Producer  不会等待服务器的反馈，如果网络发生异常或其它情况，可能会丢失消息；
4. 当`acks=1`时，如果 leader 节点在接收到消息之后，并且在 follower 节点复制数据完成之前产生错误，则这条消息会丢失；

所以如果需要保证消息不丢失，至少需要满足以下条件：
1. 同步阻塞方式发送消息
2. 设置 `acks=-1`或者`acks=all`

## 消息重复消费的问题
在 Consumer 中，offset 提交的方式有两种：
1.  自动提交；
2.  手动提交；

在新的版本中，通过 `enable.auto.commit=[true || false]`（默认为 true）以及`auto.commit.interval.ms`（默认=5000）参数来控制是否由 Consumer 自动在后台提交 offset 以及自动提交 offset 的频率（以毫秒为单位），而在旧的版本中，则通过`auto.commit.enable=[true || false]`（默认为 true）以及`auto.commit.interval.ms`（默认=60 * 1000）参数来达到同样的效果。

当 Consumer 在消费完数据并提交 offset 之后，offset 将被持久化在 zookeeper 之中。如果程序发生异常或重启，那么它将接着上一次的 offset 继续消费消息。所以如果当 Consumer 消费完消费之后，却在提交 offset 时发生异常，那么将可能导致消息被重复消费，根据消息重复消费的数量，可分为以下情形：

1. 自动提交 offset 时，由于 offset 不会立即提交，所以可能会造成单次异常却重复消费多条连续的消息；
2. 手动提交 offset 时，如果选中每消费一条消息，都手动提交一次 offset，那么针对每个分区来讲，单次异常只会至多重复消费一条消息；

通常而言，如果我们需要保证消息是全局或以键为单位的顺序消息时，选择手动提交 offset 会是更保险的做法。

## 顺序消息

保证消息的顺序入队与消费，通常分为两种情况：
1. 全局有序，比如操作 【OP_1（id=1）, OP_2（id=2）, OP_3（id=1）】，它们的消费顺序也必须是 【OP_1, OP_2, OP_3】 ；
2.  以 key 为单位的有序，比如以上操作，允许它们的操作为【OP_1, OP_3，OP_2】；

Kakfka 保证以 Partition  为单位的分区有序，所以如果选择全局有序，那么只能选择单个分区写入，以及如果消费者如果需要保证异常重启后也严格按照之前的顺序消费，那么也仅能使用单线程消费且手动提交 offset 的方式。但是好在实际的业务中，更多的是保证以 key 为单位的消息有序，所以我们可以通过将数据发送至多个 Partition，以提高程序的并发量，只要保证相同的 key 在同一个分区即可。

综合消息丢失与重复消费的问题，如果我们需要实现一个可靠的且保证以 key 为单位的有序消息，且消费者也严格按照顺序消费的程序，那么必须保证以下条件：
1. Producer 端同步发送消息，且反馈的配置为`acks=-1 || all`；
2. 相同的 key 写入相同的 Partition；
3. Consumer 使用手动提交 offset，每消费完一个消息后，手动提交 offset；
4. 做好幂等消费操作，因为重复消费的问题理论上不可避免；

# 四、参考文档
- [kafka 官方文档（中文）](http://kafka.apachecn.org/documentation.html)
- [Kafka Producer API 文档（英文）](http://kafka.apache.org/082/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)

