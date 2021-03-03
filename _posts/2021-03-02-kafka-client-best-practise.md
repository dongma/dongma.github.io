---
layout: post
title: kafka client客户端实践及原理剖析
---
## kafka client客户端实践及原理剖析

> 主要描述`kafka java client`的一些实践，以及对`client`操作数据的一些原理进行剖析。

`kafka`对集群部署环境的一些考虑，`kafka` 由 `Scala` 语言和 `Java` 语言编写而成，编译之后的源代码就是普通的“`.class`”文件。本来部署到哪个操作系统应该都是一样的，但是不同操作系统的差异还是给 `Kafka` 集群带来了相当大的影响。

主流的操作系统有`3`种：`windows`、`linux`和`macOS`，考虑到操作系统与`kafka`的适配性，`linux`系统显然要比其它两个更加合适部署`kafka`，主要在`I/O`模式的使用、数据网络传输效率、社区支持度三个方面支持比较好。

`linux`中的系统调用`select`函数属于`I/O`多路复用模型，大名鼎鼎的`epoll`系统调用则介于`I/O` 多路复用、信号驱动`I/O`模型。因此在这一点上将`kafka` 部署在`Linux` 上是有优势的，因为能够获得更高效的 `I/O`性能。零拷贝（`Zero Copy`）技术，就是当数据在磁盘和网络进行传输时避免昂贵的内核态数据拷贝从而实现快速的数据传输，`Linux` 平台实现了这样的零拷贝机制。

对于磁盘`I/O`性能，普通环境使用机械硬盘，不需要搭建`RAID`。对于磁盘容量，需根据消息数、留存时间预估磁盘容量，实际使用中建议预留`20%`～`30%`的磁盘空间。对于网络带宽，需根据实际带宽速度和业务`SLA`预估服务器数量，对于千兆网络，建议每台服务器按照`700mps`来计算，避免大流量下的丢包问题。

<!-- more -->

**集群配置中一些重要的参数**，`Broker`端的一些参数有：

1）`log.dirs`指定了`broker`需要使用的若干个文件目录路径，而`log.dir`结尾没有`s`，说明它只能表示单个路径，它是补充上一个参数用的。当挂载多个目录时，其好处在于提升读写性能、能够实现故障转移；

2）`zookeeper`的配置，`zookeeper.connect`可以指定它的值为`zk1:2181,zk2:2181,zk3:2181`。

3）第三组是与`broker`连接相关的，`listeners`学名叫监听器，其实就是通过`PLAINTEXT://localhost:9092`协议连接`kafka` 服务的。`advertised.listeners`，和 `listeners` 相比多了个`advertised`，其是在外网连接`kafka`的地址。

4）第四组参数是关于 `topic` 管理的，`auto.create.topics.enable`，是否允许自动创建`topic`。`unclean.leader.election.enable`：是否允许 `unclean Leader` 选举。`auto.leader.rebalance.enable`：是否允许定期进行 `Leader`选举。

看一些`topic`级别的参数，在启动`kafka`时设置`jvm`的一些参数：

1）`retention.ms`：规定了该 `Topic` 消息被保存的时长。默认是` 7` 天，即该 `Topic` 只保存最近`7` 天的消息。一旦设置了这个值，它会覆盖掉 `Broker` 端的全局参数值。

2）`retention.bytes`：规定了要为该 `topic` 预留多大的磁盘空间。当前默认值是`-1`，表示可以无限使用磁盘空间。

3）`KAFKA_HEAP_OPTS`：指定堆大小，行业经验`kafka`默认堆栈大小为`6g`，`KAFKA_JVM_PERFORMANCE_OPTS`：指定 GC 参数。

```shell
$> export KAFKA_HEAP_OPTS=--Xms6g --Xmx6g
$> export KAFKA_JVM_PERFORMANCE_OPTS= -server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -Djava.awt.headless=true
$> bin/kafka-server-start.sh config/server.properties
```

**生产者消息分区机制原理剖析**，`Kafka` 的消息组织方式实际上是三级结构：主题 - 分区 - 消息。其实分区的作用就是提供负载均衡的能力，或者说对数据进行分区的主要原因，就是为了实现系统的高伸缩性（`scalability`）。

所谓分区策略是决定生产者将消息发送到哪个分区的算法，常见的分区策略有轮询策略（`Round-robin`）、随机策略（`Randomness`）、按消息键保序策略（`Key-ordering`）。如下为自定义分区策略，从所有分区中找出哪些`Leader` 副本在南方的所有分区，然后随机挑选一个进行消息发送。

```java
List partitions = cluster.partitionsForTopic(topic);
return partitions.stream()
  .filter(p ->isSouth(p.leader().host()))
  .map(PartitionInfo::partition).findAny().get();
```

在`kafka`中，压缩可能发生在两个地方：生产者端和`broker`端。让`broker`端重新压缩消息有`2`种例外情况，`broker`端指定了和`producer`端不同的压缩算法，`broker`端发生了消息格式转换。一句话总结压缩和解压缩的话，`producer`端压缩、`broker`端保持、`consumer`端解压缩。

客户端一些高级功能`interceptor`，与`spring`中的拦截器原理是一样的（`aop`），不影响真实业务逻辑调用。生产者要想添加`interceptor`，只需继承`ProducerInterceptor<String, String>`类。

无消息丢失配置如何实现？`producer` 永远要使用带有回调通知的发送 API，也就是说不要使用`producer.send(msg)`，而要使用 `producer.send(msg, callback)`。Kafka 中`consumer` 端的消息丢失就是这么一回事。要对抗这种消息丢失，办法很简单：维持先消费消息（阅读），再更新位移（书签）的顺序即可。

设置`acks = all`。`acks` 是 `Producer `的一个参数，代表了你对“已提交”消息的定义。

设置`retries` 为一个较大的值。这里的`retries` 同样是`Producer` 的参数，对应前面提到的`Producer`自动重试。

确保消息消费完成再提交。`consumer` 端有个参数 `enable.auto.commit`，最好把它设置成 `false`，并采用手动提交位移的方式。

设置`unclean.leader.election.enable = false`、设置`replication.factor >= 3`、设置 `min.insync.replicas > 1`的配置。

```java
public class ProducerClient {

    /* kafka用于防止消息丢失的因素: */
    // 1) 维持先消费消息（阅读），再更新位移（书签）的顺序即可。这样就能最大限度地保证消息不丢失。（消费者端 维持先消费， 再提交offset）

    // 2) unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。
    // 如果一个Broker落后原先的 Leader 太多，那么

    public static void main(String[] args) {
        Properties kafkaProp = new Properties();
        kafkaProp.put("bootstrap.servers", "localhost:9092");
        // 则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”
        kafkaProp.put("acks", "all");
        kafkaProp.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        kafkaProp.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        // 开启kafka的gzip压缩, 向broker发送的每条message都是压缩的
        kafkaProp.put("compression.type", "gzip");

        // 开启生产者消息的幂等性, 保证底层message消息只会发送一次(用空间换，msg会多传一个字段 用于去重)
        kafkaProp.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        // 2. producer生产者启用事务（在kafka 0.11开始的支持）
        kafkaProp.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "kafka-transactional");

        // 设置interceptor用于统计生产者发送消息延时
        List<String> interceptor = new ArrayList<>();
        interceptor.add("com.example.kakfa.interceptor.AvgLatencyProducerInterceptor");
        kafkaProp.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, interceptor);

        Producer<String, String> client = new KafkaProducer<>(kafkaProp);
        // 1. send调用时使用回调函数callback, exception 可判断消息是否提交成功，消费者 “位移”类似于我们看书时使用的书签
        client.send(new ProducerRecord<>("", ""), (recordMetadata, exception) -> {
//            RecordMetadata var1, Exception var2
        });

        // 2. 在kafka-client客户端中使用transactional事务机制, 用于提交kafka message消息
        client.initTransactions();
        try {
            client.beginTransaction();
            client.send(new ProducerRecord<>("topicA", ""));
            client.send(new ProducerRecord<>("topicB", ""));
            client.commitTransaction();
        } catch (ProducerFencedException ex) {
            client.abortTransaction();
        }
    }
}
```

`kafka`社区决定采用`tcp`而不是`http`，能够利用`TCP` 本身提供的一些高级功能，比如多路复用请求以及同时轮询多个连接的能力，目前已知的`HTTP` 库在很多编程语言中都略显简陋。

何时创建`TCP` 连接？目前我们的结论是这样的，`TCP`连接是在创建`KafkaProducer` 实例时建立的。`TCP`连接还可能在两个地方被创建：一个是在更新元数据后，另一个是在消息发送时。

何时关闭`TCP `连接？`Producer`端关闭`TCP`连接的方式有两种：一种是用户主动关闭，一种是`Kafka`自动关闭。

开启`kafka`生产者消息幂等性、`producer`生产者启用事务需要在`producer`的`properties`中设置以下配置：

```java
// 开启生产者消息的幂等性, 保证底层message消息只会发送一次(用空间换，msg会多传一个字段 用于去重)
kafkaProp.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// 2. producer生产者启用事务（在kafka 0.11开始的支持）
kafkaProp.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "kafka-transactional");
```

`Consumer Group`是`Kafka`提供的可扩展且具有容错性的消费者机制。既然是一个组，那么组内必然可以有多个消费者或消费者实例`Consumer Instance`，它们共享一个公共的 `ID`，这个 `ID` 被称为 `Group ID`。组内的所有消费者协调在一起来消费订阅主题`（Subscribed Topics）`的所有分区`（Partition）`。

Rebalance 本质上是一种协议，规定了一个Consumer Group下的所有Consumer如何达成一致，来分配订阅Topic的每个分区。比如某个Group下有20个Consumer 实例，它订阅了一个具有100个分区的Topic。正常情况下，Kafka平均会为每个Consumer分配5个分区。这个分配的过程就叫Rebalance。

那么 `Consumer Group` 何时进行 `Rebalance `呢？`Rebalance` 的触发条件有 `3 `个。

1）组成员数发生变更。比如有新的` Consumer `实例加入组或者离开组，抑或是有 `Consumer `实例崩溃被“踢出”组。

2）订阅主题数发生变更。`Consumer Group` 可以使用正则表达式的方式订阅主题，比如 `consumer.subscribe(Pattern.compile("t.*c")) `就表明该 `Group` 订阅所有以字母` t `开头、字母 `c `结尾的主题。在 `Consumer Group `的运行过程中，你新创建了一个满足这样条件的主题，那么该` Group` 就会发生` Rebalance`。

3）订阅主题的分区数发生变更。`Kafka` 当前只能允许增加一个主题的分区数。当分区数增加时，就会触发订阅该主题的所有 `Group` 开启 `Rebalance`。

揭开`kafka`位移主题，`__consumer_offsets`在`Kafka`源码中有个更为正式的名字，叫位移主题，即`Offsets Topic`。但是，`ZooKeeper`其实并不适用于这种高频的写操作。位移主题的`key`中应该保存`3`部分内容：<`GroupId, `topic`名称, 分区号>。

如果位移主题是`Kafka`自动创建的，那么该主题的分区数是`50（offsets.topic.num.partitions）`，副本数是`3（offsets.topic.replication.factor`）。`Kafka`提供了专门的后台线程定期地巡检待`compact`的主题，看看是否存在满足条件的可删除数据。这个后台线程叫`Log Cleaner`。

```java
    /*
     * __consumer_offsets 为消费者消费的内部主题, 主要由于zookeeper并不适合高频的写操作
     * 位移主题的 Key 中应该保存 3 部分内容：<GroupId, topic名称, 分区号>
     */
    public static void main(String[] args) {
        // 通常来说，当 Kafka 集群中的第一个 Consumer 程序启动时，Kafka 会自动创建位移主题 (何时创建 __consumer_offsets)
        Properties kafkaProp = new Properties();
        kafkaProp.put("bootstrap.servers", "localhost:9092");
        kafkaProp.put("group.id", "test");
        /*
         * Consumer 端有个参数叫 enable.auto.commit，如果值是 true，则 Consumer 在后台默默地为你定期提交位移，提交间隔由一个专属的参数
         * auto.commit.interval.ms 来控制
         */
        kafkaProp.put("enable.auto.commit", "true");
        kafkaProp.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        kafkaProp.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(kafkaProp);
        consumer.subscribe(Arrays.asList("foo", "bar"));
        /*while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord record : records) {
                System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
            }
        }*/

        /* 手动提交offset的示例：
         * (Compact 策略) 当__consumer_offsets 数据较大时，会使用 Compact 策略 (压实)，删除过期offsets; Log Cleaner后台启线程巡查
         */
        kafkaProp.put("enable.auto.commit", "false");
        while (true) {
            ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofSeconds(1));
//            process(records);
            consumer.commitAsync((offsets, exception) -> {
                if (null != exception) { /* handleException(exception); */ }
            });
        }
        /*
         * Consumer Group 如何确定为它服务的 Coordinator 在哪台 Broker 上呢？
         * 1、计算group.id的hashcode； 2、hashcode与分区数取模求绝对值，得到分区号； 3、找出该分区的leader副本所在的broker，即coordinator
         */
    }
```

避免`consumer`发生`rebalance()`重平衡的策略，去除`2`类非必要的`rebalance`。在提交位移时给出的最佳实践，同步与异步提交结合的方式（最后一次为同步）。

```java
/*
 * consumer避免rebalance的方法，2类非必要的rebalance：
 *  第一类非必要 Rebalance 是因为未能及时发送心跳，导致 Consumer 被“踢出”Group 而引发的 （session.timeout.ms = 6s，
 *      heartbeat.interval.ms = 2s，能够发送3次）
 *  第二类非必要 Rebalance 是 Consumer 消费时间过长导致的 （存在重操作-写mongo max.poll.interval.ms）
 */
private void bestPractise(KafkaConsumer<String, String> consumer) {
  try {
    while (true) {
      ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
      // process(records);  /* 处理消息，异步提交offsets内容 */
      consumer.commitAsync();
    }
  } catch (Exception ex) {
    // handleException(ex);  /* 处理异常的代码 */
  } finally {
    try { consumer.commitSync(); /* 最后一次提交使用同步阻塞式提交 */} finally { consumer.close();}
  }
}
```

另外一种每消费`100`条记录提交一次`offset`，将要提交的`offset`值保存在`OffsetAndMetadata`

```java
/*
 * (consumer offsets消费者位移)下一条消息的位移
 * 从用户的角度来说，位移提交分为自动提交和手动提交；从 Consumer 端的角度来说，位移提交分为同步提交和异步提交
 */
private void commitPer100(KafkaConsumer<String, String> consumer) {
  Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>(8);
  int count = 0;
  while (true) {
    ConsumerRecords<String, String> records =
      consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord record : records) {
      // process(record); /* 处理消息 */
      offsets.put(new TopicPartition(record.topic(), record.partition()),
                  new OffsetAndMetadata(record.offset() + 1));
      if (count % 100 == 0) {
        consumer.commitAsync(offsets, null); /* 回调处理逻辑是null */
      }
      count++;
    }
  }
}
```

`CommitFailedException`异常如何处理? `consumer`客户端在提交位移时出现了错误或异常，并且是不可恢复的严重异常。典型场景：当消息处理的总时间超过`max.poll.interval.ms`参数时，`kafka consumer`会抛出`CommitFailedException`。预防`CommitFailedException`异常`4`种方法，缩短单条消息处理时间、增加`consumer`端允许下游系统消费一批数据的最大时长、减少下游系统一次消费的消息总数、下游系统使用多线程加速消费。

`java`多线程开发消费者实例，有两种方案：第一种一个`consumer`一个线程，从头到尾处理完所有的流程。第二种为单线程拉取`kafka`消息，然后通过线程池处理`kafka`中的消息。

```java
public class ConsumerRunner implements Runnable {
    private final AtomicBoolean closed = new AtomicBoolean(false);
    public ConsumerRunner(KafkaConsumer consumer) {
        this.consumer = consumer;
    }
    private final KafkaConsumer<String, String> consumer;
    @Override
    public void run() {
        try {
            consumer.subscribe(Arrays.asList("topic"));
            while (!closed.get()) {
                consumer.poll(Duration.ofMillis(10000));
                // 执行消息处理的逻辑
            }
        } catch (WakeupException ex) {
            /* Ignore exception if closing */
            if (!closed.get()) {
                throw ex;
            }
        } finally {
            consumer.close();
        }
    }
    /* Shutdown hook which can be called from a separate thread */
    public void shutdown() {
        closed.set(true);
        consumer.wakeup();
    }
}
```

第二种使用线程池的方式，单个`consumer`通过`poll(200)`消费`kafka`中数据，然后使用`thread-pool`对数据进行业务处理。

```java
public static void main(String[] args) {
  /* 使用方案二，加速多线程消费消息, 根据kafka的broker配置进行构建 */
  KafkaConsumer<String, String> consumer = null;
  int workNum = 20;
  ExecutorService executors = new ThreadPoolExecutor(workNum, workNum, 
                                                     0L, TimeUnit.MILLISECONDS,
                                                     new ArrayBlockingQueue<>(1000),
                                                     new ThreadPoolExecutor.CallerRunsPolicy());
  /* 消费模式代码, 单个kafkaConsumer负责拉消息，获取的消息通过threadpool执行 */
  while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, String> record : records) {
      // executors.submit(new Worker(record));  /* 对于consumer poll到的消息，使用threadpool分别执行 */
    }
  }
}
```

