---
title: "一文理解Kafka的设计原理"
source: "https://zhuanlan.zhihu.com/p/610324541"
author:
  - "[[Lion 莱恩呀]]"
published:
created: 2026-06-23
description: "一、Kafka简介Kafka 本质上是一个 MQ（Message Queue），具有消息队列的优点： 解耦：允许独立的扩展或修改队列两边的处理过程。可恢复性：即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处…"
tags:
  - "clippings"
---
[收录于 · 消息队列Kafka](https://www.zhihu.com/column/c_1613298330271916033)

15 人赞同了该文章

目录

收起

一、Kafka简介

二、Kafka的架构

2.1、Kafka 一些重要概念

2.2、工作流程

2.3、副本原理

2.4、分区和主题的关系

2.5、生产者

2.5.1、分区可以水平扩展

2.5.2、分区策略

2.6、消费者

2.6.1、消费方式

2.6.2、分区分配策略

2.7、数据可靠性保证

2.7.1、副本数据同步策略

2.7.2、ACK 应答机制

2.7.3、可靠性指标

总结

## 一、Kafka简介

Kafka 本质上是一个 [MQ](https://zhida.zhihu.com/search?content_id=223759458&content_type=Article&match_order=1&q=MQ&zhida_source=entity) （Message Queue），具有消息队列的优点：

1. 解耦：允许独立的扩展或修改队列两边的处理过程。
2. 可恢复性：即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理。
3. 缓冲：有助于解决生产消息和消费消息的处理速度不一致的情况。
4. 灵活性和峰值处理能力：不会因为突发的超负荷的请求而完全崩溃，消息队列能够使关键组件顶住突发的访问压力。
5. 异步通信：消息队列允许用户把消息放入队列但不立即处理它。

## 二、Kafka的架构

![](https://pic2.zhimg.com/v2-3639ade453c6e68fa46c2bf9aa1dd4fd_1440w.jpg)

Kafka\_arch

Kafka只写数据到leader副本，也只从leader副本获取数据。如果leader失效，会重新选择出leader。

优点类似MySQL的主从关系，写数据都是到主机里面，但是读数据不一样，Kafka读数据只能从主机里面读。

1. Kafka 存储的消息来自任意多被称为 Producer 生产者的进程。数据从而可以被发布到不同的 Topic 主题下的不同 [Partition](https://zhida.zhihu.com/search?content_id=223759458&content_type=Article&match_order=1&q=Partition&zhida_source=entity) 分区。
2. 在一个分区内，这些消息被索引并连同时间戳存储在一起。其它被称为 Consumer 消费者的进程可以从分区订阅消息。
3. Kafka 运行在一个由一台或多台服务器组成的集群上，并且分区可以跨集群结点分布。

### 2.1、Kafka 一些重要概念

1. Producer：消息生产者，向 Kafka Broker 发消息的客户端。
2. Consumer：消息消费者，从 Kafka Broker 获取消息的客户端。
3. [Consumer Group](https://zhida.zhihu.com/search?content_id=223759458&content_type=Article&match_order=1&q=Consumer+Group&zhida_source=entity) ：消费者组（CG），消费者组内每个消费者负责消费不同分区的数据，提高消费能力。一个分区只能由组内一个消费者消费，消费者组之间互不影响。所有的消费者都属于某个消费者组，即消费者组是逻辑上的一个订阅者。
4. Broker：一台 Kafka 机器就是一个 Broker。一个集群(kafka cluster)由多个 Broker 组成。一个Broker 可以容纳多个 Topic。
5. Topic：可以理解为一个队列，Topic 将消息分类，生产者和消费者面向的是同一个Topic。
6. Partition：为了实现扩展性，提高并发能力，一个非常大的 Topic 可以分布到多个 Broker （即服务器）上，一个 Topic 可以分为多个 Partition，同一个topic在不同的分区的数据是不重复的，每个 Partition 是一个有序的队列，其表现形式就是一个一个的文件夹。
7. [Replication](https://zhida.zhihu.com/search?content_id=223759458&content_type=Article&match_order=1&q=Replication&zhida_source=entity) ：每一个分区都有多个副本，副本的作用是做备胎。当主分区（Leader）故障的时候会选择一个备胎（Follower）上位，成为Leader。 **在kafka中默认副本的最大数量是10个，且副本的数量不能大于Broker的数量** ，follower和leader绝对是在不同的机器，同一机器对同一个分区也只可能存放一个副本（包括自己）。
8. Message：消息，每一条发送的消息主体。
9. Leader：每个分区多个副本的“主”副本，生产者发送数据的对象，以及消费者消费数据的对象，都是 Leader。
10. Follower：每个分区多个副本的“从”副本，实时从 Leader 中同步数据，保持和 Leader 数据的同步。Leader 发生故障时，某个 Follower 还会成为新的 Leader。
11. Offset：消费者消费的位置信息，监控数据消费到什么位置，当消费者挂掉再重新恢复的时候，可以从消费位置继续消费。同一主题，不同的分区，它们的offset是独立的。
12. [ZooKeeper](https://zhida.zhihu.com/search?content_id=223759458&content_type=Article&match_order=1&q=ZooKeeper&zhida_source=entity) ：Kafka 集群能够正常工作，需要依赖于 ZooKeeper，ZooKeeper 帮助 Kafka 存储和管理集群信息。

### 2.2、工作流程

![](https://picx.zhimg.com/v2-1db7fbc5a8236cb177bec92c3c8a20fb_1440w.jpg)

kafka\_work

1. 不同的partition的offerset 是独立的。
2. Kafka 中消息是以 Topic 进行分类的，生产者生产消息，消费者消费消息，面向的都是同一个 Topic。
3. Topic 是逻辑上的概念，而 Partition 是物理上的概念，每个 Partition 对应于一个 log 文件，该 log 文件中存储的就是 Producer 生产的数据。Producer 生产的数据会不断追加到该 log 文件末端，且每条数据都有自己的 Offset。
4. 消费者组中的每个消费者，都会实时记录自己消费到了哪个 Offset，以便出错恢复时，从上次的位置继续消费。
5. 日志默认在：/tmp/kafka-logs

### 2.3、副本原理

副本机制（Replication），也可以称之为备份机制，通常是指分布式系统在多台网络互联的机器上保存有相同的数据拷贝。副本机制的好处在于：

1. 提供数据冗余（即提高可用性）。
2. 提供高伸缩性（支撑更高的读请求量）。
3. 改善数据局部性（降低系统延时）。

目前Kafka只实现了副本机制带来的第 1 个好处，即是提供数据冗余实现高可用性和高持久性。

在kafka生产环境中，每台 Broker 都可能保存有各个主题下不同分区的不同副本，因此，单个 Broker上存有成百上千个副本的现象是非常正常的。

比如了一个有 3 台 Broker 的 Kafka 集群上的副本分布情况。主题 1 分区 0 的 3 个副本分散在 3 台 Broker 上，其他主题分区的副本也都散落在不同的 Broker 上，从而实现数据冗余。

![](https://pic2.zhimg.com/v2-f10a4c968ff1b214f51b4b1e33fd0501_1440w.jpg)

kafka\_brokers

Kafka是基于领导者（Leader-based）的副本机制：

![](https://picx.zhimg.com/v2-6de32ba171e656cfa9d9c8aa920d5137_1440w.jpg)

1. 在 Kafka 中，副本分成两类：领导者副本和追随者副本。每个分区在创建时都要选举一个副本，称为领导者副本，其余的副本自动称为追随者副本。
2. Kafka 副本机制中的追随者副本是不对外提供服务的。
3. 当领导者副本挂掉了，或领导者副本所在的 Broker 宕机时，Kafka 依托于 ZooKeeper 提供的监控功能能够实时感知到，并立即开启新一轮的领导者选举，从追随者副本中选一个作为新的领导者。老 Leader 副本重启回来后，只能作为追随者副本加入到集群中。

### 2.4、分区和主题的关系

1. 一个分区只属于一个主题。
2. 一个主题可以有多个分区。
3. 同一主题的不同分区内容不一样，每个分区有自己独立的offset。
4. 同一主题不同的分区能够放置到不同节点的broker。
5. 分区规则设置得当可以使得同一主题的消息均匀落在不同的分区。

### 2.5、生产者

生产者是数据的入口。Producer在写入数据的时候永远是找leader，不会直接将数据写入follower。

![](https://pic3.zhimg.com/v2-26a2888d83aa363c9a75b369107bf156_1440w.jpg)

### 2.5.1、分区可以水平扩展

Kafka 的消息组织方式实际上是三级结构：主题 - 分区 - 消息。主题下的每条消息只会保存在某一个分区中，而不会在多个分区中被保存多份。

**分区的作用主要提供负载均衡的能力，能够实现系统的高伸缩性** （Scalability)。不同的分区能够被放置到不同节点的机器上，而数据的读写操作也都是针对分区这个粒度而进行的，这样每个节点的机器都能独立地执行各自分区的读写请求处理。这样，当性能不足的时候可以通过添加新的节点机器来增加整体系统的吞吐量。

分区原则：需要将 Producer 发送的数据封装成一个 ProducerRecord 对象。

该对象需要指定一些参数：

1. topic：string 类型，NotNull。
2. partition：int 类型，可选。
3. timestamp：long 类型，可选。
4. key：string 类型，可选。
5. value：string 类型，可选。
6. headers：array 类型，Nullable。

### 2.5.2、分区策略

所谓分区策略是决定生产者将消息发送到哪个分区的算法。Kafka 为我们提供了默认的分区策略，同时它也支持自定义分区策略。

**（1）轮询策略。** Round-robin 策略，即顺序分配。

![](https://pic4.zhimg.com/v2-302a8a701d50e3cc847f2399fca3eb95_1440w.jpg)

kafka\_round\_robin

轮询策略有非常优秀的负载均衡表现，它总是能保证消息最大限度地被平均分配到所有分区上，故默认情况下它是最合理的分区策略，也是最常用的分区策略之一。

**（2）随机策略-Randomness 策略。** 随机就是随意地将消息放置到任意一个分区上。

![](https://pica.zhimg.com/v2-331b6e1c1a96f53b4b86bb1f3318797c_1440w.jpg)

kafka\_randomness

**（3）按消息键保序策略-Key-ordering 策略。**

Kafka 允许为每条消息定义消息键，简称为 Key。这个 Key 的作用非常大，它可以是一个有着明确业务含义的字符串；也可以用来表征消息元数据。一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略。

**（4）默认分区规则。**

1. 如果指定的partition，那么直接进入该partition。
2. 如果没有指定partition，但是指定了key，使用key的 hash一选择partition。
3. 如果既没有指定partition，也没有指定key，使用轮询一的方式进入partition。

### 2.6、消费者

传统的消息队列模型的消息一旦被消费，就会从队列中被删除，而且只能被下游的一个Consumer 消费。这种模型的伸缩性（scalability）很差，因为下游的多个 Consumer 都要“抢”这个共享消息队列的消息。发布 / 订阅模型倒是允许消息被多个 Consumer 消费，但它的问题也是伸缩性不高，因为每个订阅者都必须要订阅主题的所有分区。这种全量订阅的方式既不灵活，也会影响消息的真实投递效果。

Kafka引入了consumer group概念。Kafka 仅仅使用 Consumer Group 这一种机制，却同时实现了传统消息引擎系统的两大模型：如果所有实例都属于同一个 Group，那么它实现的就是消息队列模型；如果所有实例分别属于不同的 Group，那么它实现的就是发布 / 订阅模型。

### 2.6.1、消费方式

Consumer 采用 Pull（拉取）模式从 Broker 中读取数据。

Pull 模式则可以根据 Consumer 的消费能力以适当的速率消费消息。Pull 模式不足之处是，如果 Kafka没有数据，消费者可能会陷入循环中，一直返回空数据。

因为消费者从 Broker 主动拉取数据，需要维护一个长轮询，针对这一点， Kafka 的消费者在消费数据时会传入一个时长参数 timeout。如果当前没有数据可供消费，Consumer 会等待一段时间之后再返回，这段时长即为 timeout。

### 2.6.2、分区分配策略

一个 Consumer Group 中有多个 Consumer，一个 Topic 有多个 Partition，所以必然会涉及到Partition 的分配问题，即确定哪个 Partition 由哪个 Consumer 来消费。

Kafka 有四种分配策略：

1. RoundRobin：针对集群中的所有topic；轮询的方式依次将分区分配给消费者。
2. Range，默认为Range：针对每个topic；通过 分区数 / 消费者数 决定每个消费者消费几个分区。如果除不尽则前面几个消费者会多消费1个分区（topic很多时容易产生数据倾斜）。
3. Sticky：首先会尽量均衡放置分区到消费者上面，出现同一消费组内消费者出现问题的时候，会尽量保持原有分配的分区不变化。
4. CooperativeSticky：在不停止消费的情况下进行增量再平衡。

可以通过参数partition.assignment.strategy来配置，默认是Range+ CooperativeSticky。

**（1）Range(默认策略)。** Range 方式是按照主题来分的，不会产生轮询方式的消费混乱问题。

假设有10个分区，3个消费者，排完序的分区将会是0,1,2,3,4,5,6,7,8,9；消费者线程排完序将会是C1-0,C2-0,C3-0。然后将partitions的个数除于消费者线程的总数来决定每个消费者线程消费几个分区。如果除不尽，那么前面几个消费者线程将会多消费一个分区。

回到例子，10个分 区，3个消费者线程， 10/3 = 3，而且除不尽，那么消费者线程 C1-0将会多消费一个分区。C1-0 将消费 0, 1, 2, 3 分区；C2-0将消费 4,5,6分区；C3-0将消费 7,8,9分区。

**（2）RoundRobin。**

RoundRobin 轮询方式将分区所有作为一个整体进行 Hash 排序，消费者组内分配分区个数最大差别为1，是按照组来分的，可以解决多个消费者消费数据不均衡的问题。

轮询分区策略是把所有partition和所有consumer线程都列出来，然后按照hashcode进行排序。最后通过轮询算法分配partition给消费线程。如果所有consumer实例的订阅是相同的，那么partition会均匀分布。

使用轮询分区策略必须满足两个条件：

1. 每个主题的消费者实例具有相同数量的流。
2. 每个消费者订阅的主题必须是相同的。

### 2.7、数据可靠性保证

为保证 Producer 发送的数据，能可靠地发送到指定的 Topic，Topic 的每个 Partition 收到 Producer发送的数据后，都需要向 Producer 发送 [ACK](https://zhida.zhihu.com/search?content_id=223759458&content_type=Article&match_order=1&q=ACK&zhida_source=entity) （ACKnowledge 确认收到）。如果 Producer 收到 ACK，就会进行下一轮的发送，否则重新发送数据。

![](https://pic1.zhimg.com/v2-dc8b2dc83df6f1632cba5ed9612bc958_1440w.jpg)

### 2.7.1、副本数据同步策略

确保有 Follower 与 Leader 同步完成，Leader 再发送 ACK，这样才能保证 Leader 挂掉之后，能在 Follower 中选举出新的 Leader 而不丢数据。

| 方案 | 优点 | 缺点 |
| --- | --- | --- |
| 半数以上完成同步，就发送ack | 延迟低 | 选举新的leader时，容忍n台节点故障，需要2n+1个副本。 |
| 全部完成同步，才发送ack | 选举新的leader时，容忍n台节点故障，需要n+1个副本。 | 延迟高。 |

当采用第二种方案时，所有 Follower 完成同步，Producer 才能继续发送数据，设想有一个 Follower 因为某种原因出现故障，那 Leader 就要一直等到它完成同步。这个问题怎么解决？

1. Leader维护了一个动态的 \*\*in-sync replica set（ [ISR](https://zhida.zhihu.com/search?content_id=223759458&content_type=Article&match_order=1&q=ISR&zhida_source=entity) ）\*\* ：和 Leader 保持同步的 Follower 集合。
2. 当 ISR 集合中的 Follower 完成数据的同步之后，Leader 就会给 Follower 发送 ACK。
3. 如果 Follower 长时间未向 Leader 同步数据，则该 Follower 将被踢出 ISR 集合，该时间阈值由replica.lag.time.max.ms 参数设定。Leader 发生故障后，就会从 ISR 中选举出新的 Leader。

### 2.7.2、ACK 应答机制

Kafka 为用户提供了三种可靠性级别，用户根据可靠性和延迟的要求进行权衡，选择以下的配置。

![](https://pic4.zhimg.com/v2-1f005f1bd6dfde04d02103f4b23deef7_1440w.jpg)

kafka\_ack

ACK 参数配置：

- 0：Producer 不等待 Broker 的 ACK，这提供了最低延迟，Broker 一收到数据还没有写入磁盘就已经返回，当 Broker 故障时有可能丢失数据。
- 1：Producer 等待 Broker 的 ACK，Partition 的 Leader 落盘成功后返回 ACK，如果在 Follower同步成功之前 Leader 故障，那么将会丢失数据。
- \-1（all）：Producer 等待 Broker 的 ACK，Partition 的 Leader 和 Follower 全部落盘成功后才返回 ACK。但是在 Broker 发送 ACK 时，Leader 发生故障，则会造成数据重复。

### 2.7.3、可靠性指标

1. 分区副本，你可以创建更多的分区来提升可靠性，但是分区数过多也会带来性能上的开销，一般来说，3个副本就能满足对大部分场景的可靠性要求。
2. ACKS，生产者发送消息的可靠性，也就是我要保证我这个消息一定是到了broker并且完成了多副本的持久化。
3. 保障消息到了broker之后，消费者也需要有一定的保证，因为消费者也可能出现某些问题导致消息没有消费到。
4. enable.auto.commit默认为true，也就是自动提交offset，自动提交是批量执行的，有一个时间窗口，这种方式会带来重复提交或者消息丢失的问题，所以对于高可靠性要求的程序，要使用手动提交。 对于高可靠要求的应用来说，宁愿重复消费也不应该因为消费异常而导致消息丢失。

## 总结

1. 一个主题多个分区的场景下，kafka只能保证同一个分区的消息顺序性，不能保证不同分区间的消息顺序性。
2. 一般，配置三个副本就可以满足绝大部分需求。
3. 一个消费者可以订阅多个主题，可以去消费多个分区，但一个分区不支持多个消费组（同一个消费组）读取。
4. kafka的逻辑理解：topic、broker、副本、消息模型（点对点和发布订阅）、消费组及其里面的消费者、生产者的分配策略、消费者的分区消费策略。
![](https://pic2.zhimg.com/v2-d8d28f8d5e2e77dcfbe6698b94197de3_1440w.jpg)

编辑于 2023-03-02 12:22・广东

赞同 15