---
title: Kafka原理分析
date: 2019-12-12 20:14:00
categories: Kafka
tags:
- 消息中间件
- Kafka
---
我们都知道Kafka具有很高的吞吐量、数据分片、冗余和容错等优点，一般用于用户行为追踪以及分布式系统日志收集等场景，那么Kafka是如何做到这些优点的呢，今天就让我们来一一分析。

## Kafka入门

### 安装
1. 点击[下载](http://mirror.bit.edu.cn/apache/kafka/2.3.1/kafka_2.11-2.3.1.tgz)
2. 解压:`tar -zxvf kafka_2.11-2.3.1.tgz`
3. 启动zookeeper:`sh ${zookeeperDir}/bin/zkServer.sh start`，zookeeper集群则需要将集群中所有节点启动
4. 配置config目录下的server.properties中:`zookeeper.connect=192.168.3.224:2181`
5. 启动/停止:`sh kafka-server-start.sh -daemon ../config/server.properties`/`sh kafka-server-stop.sh ../config/server.properties`

### 集群配置
配置config目录下server.properties文件
1. 将`broker.id`属性配置为当前节点id，集群中的所有节点id不能相同，例如`broker.id=0/1/2...`
2. 将`advertised.listeners`属性改为当前节点的主机地址，例如`advertised.listeners=PLAINTEXT://192.168.3.224:9092`
这样，当`Broker`启动的时候，会向zookeeper注册自己的主机及端口，其他`Broker`就可以通过ip和端口来连接

## 基本操作
### 命令行操作
* 创建Topic:`sh kafka-topics.sh --create --zookeeper 192.168.3.224:2181 --replication-factor 1 --partitions 1 --topic test`
* 列出所有Topic:`sh kafka-topics.sh --list --zookeeper 192.168.3.224:2181`
* 查看Topic详情:`sh kafka-topics.sh --describe --zookeeper localhost:2181 --topic test`
```shell
[root@MiWiFi-R3L-srv bin]# sh kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
Topic:test	PartitionCount:1	ReplicationFactor:1	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
```
* 发送消息:`sh kafka-console-producer.sh --broker-list 192.168.3.224:9092 --topic test`
* 消费消息:`sh kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning`

### Java API使用
* producer
```java
public class Producer {
    public static void main(String[] args) {
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,"192.168.3.207:9092,192.168.3.9:9092,192.168.3.155:9092");
        properties.put(ProducerConfig.ACKS_CONFIG,"all");
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.IntegerSerializer");
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.StringSerializer");
        KafkaProducer producer = new KafkaProducer<Integer,String>(properties);
        for (int i=0;i<100;i++) {
            ProducerRecord<Integer,String> record = new ProducerRecord<Integer, String>("firstTopic","HelloWorld" + i);
            Future future = producer.send(record);
            System.out.println(future);
        }
        producer.close();
    }
}
```
* consumer
```java
public class Consumer {
    public static void main(String[] args) throws IOException {
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,"192.168.3.207:9092,192.168.3.9:9092,192.168.3.155:9092");
        properties.put(ConsumerConfig.GROUP_ID_CONFIG,"MrATooConsumer");
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG,"true");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.IntegerDeserializer");
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,"org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<Integer,String> consumer = new KafkaConsumer<Integer, String>(properties);
        /*TopicPartition partition = new TopicPartition("firstTopic",1);
        consumer.assign(Arrays.asList(partition));*/

        consumer.subscribe(Arrays.asList("firstTopic"));
        ConsumerRecords<Integer, String> records = consumer.poll(Duration.ofDays(7));
        Iterator<ConsumerRecord<Integer, String>> iterator = records.iterator();
        while (iterator.hasNext()) {
            ConsumerRecord<Integer, String> record = iterator.next();
            System.out.println(record.value());
        }
        consumer.close();
    }
}
```

## 配置分析
### Producer可选配置
* acks:该配置表示`producer`发送到`Broker`上的确认值，该值有三个选项，分别是：
    * 0:`producer`发送消息过后不需要等待`Broker`确认，该方式延时小但消息容易丢失
    * 1:`producer`发送消息过后只需要等待kafka集群的`leader`节点确认，该方式延时和可靠性适中
    * all(-1):`producer`发送消息过后需要等待ISR列表中的所有节点确认，该方式延时较长，但消息不容易丢失。但ISR可以缩小到1，所以并不能百分之百保证消息不丢失。
* batch.size:当生产者发送多个消息到`Broker`上时，为了节约网络开销，可以通过批量的方式来提交消息，可以通过该配置来设置批量提交消息的大小，默认是16kb。也就是，当一批消息达到了
batch.size大小的时候统一发送。
> 注意：这里的batch.size大小是针对同一个`partition`
* linger.ms:消息发送请求的延迟（间隔），即：当消息发送间隔时间较短，并且还没有达到batch.size大小时，这个时候客户端并不会立即发送请求到`Broker`上，而是延迟`linger.ms`过后，将多个消息
合并成一个消息发送，该配置为0，则代表没有延迟，如果配置成正整数值，则会减少请求数量，但也会有消息发送延迟。如果同时配置了`linger.ms`和`batch.size`，则满足一个条件就会发送。
* max.request.size:现在请求数据的最大字节数，默认为1M

### Consumer可选配置
* group.id:kafka中的每个消费者都有一个组，组和消费者是一对多的关系，对于一个Topic而言，如果Topic对应多个组，则类似于ActiveMQ中Topic的概念，如果Topic对应一个组，则类似于ActiveMQ
中Queue的概念，同一个组下的多个消费者可以同时消费一个Topic下的多个分区，一个分区只能分配给一个消费者进行消费。
* enable.auto.commit:消息消费后自动提交，只有当消息被提交过后才会确保消息不会再次被消费，可以接口`auto.commit.interval.ms`来优化自动提交的频率。当然，我们也可以通过
`consumer.commitSync()`方法来手动提交消息。
* auto.offset.reset:
* max.poll.records:配置每次poll消息的数量

## Topic和Partition
### Topic
Topic是一个逻辑的概念，可以认为是一个消息集合，不同的Topic是分开存储的，一个Topic可以有多个生产者向它发送消息，也可由有多个消费者消费消息。

### Partition
一个Topic可以个多个Partition（至少有一个分区），同一个Topic下的不同分区的消息是不同的，每个消息在分配到分区时，都会被分配一个offset（偏移量），kafka通过offset保证消息在同一个
分区的顺序，也就是说，同一个分区的消息是有序的。

### Partition存储
Partition以文件的形式存在于文件系统中，例如：`sh kafka-topics.sh --create --zookeeper 192.168.3.224:2181 --replication-factor 1 --partitions 3 --topic test`，以上命令会创建一个
有三个分区的名称为test的Topic，那么这三个分区会生成三个文件夹均匀分布在不同Broker中，文件夹的命名规则为`<topic_name>-<partition_id>`，partition_id范围为0~3。

## 消息分发策略
一个消息由key和value组成，发送一条消息之前，我们可以指定消息的key和value，然后kafka会根据指定的key和partition机制来决定当前这条消息应该被存储到那个分区中，默认情况下，kafka采用的消息分发机制
是Hash取模算法，如果key为空，则会随机分配一个分区，这个随机分区会在`metadata.max.age.ms`配置指定的时间内固定选择一个，这个值默认是10分钟，也就是说，每10分钟，随机分区会更新一次。

## 消息消费原理
每个Topic有多个Partition，每个Consumer Group有多个消费者，同一个Partition只允许被一个Consumer Group中的一个消费者消费。那么同一个消费者中的消费者是如何去消费同一个Topic下的
多个Partition的呢？这就牵扯到了分区分配策略

### 分区分配策略
kafka中提供两种分区分配策略，分别是Range（范围分区）、RoundRobin（轮询），通过`partition.assignment.strategy`配置来指定分区分配策略。

#### Range（范围分区）
首先将同一个Topic下的所有Partition通过分区ID进行排序，然后将同一个Consumer Group下的所有Consumer按照一定规则排序，然后用Partition总数除以消费者总数，如果除不尽，则将余数
按照顺序分配到排序过后的Consumer上。
例如：现在有一个Topic test，10个分区;一个Consumer Group,三个消费者;
* 将Partition通过ID排序过后得到`test-0,test-1,test-2,test-3,test-4,test-5,test-6,test-7,test-8,test-9`
* 将消费者排序，假如是`C0,C1,C2`
* 先计算`10/3=3`，然后计算`10%3=1`，最后得到三个组，分别是`0,1,2,3`/`4,5,6`/`7,8,9`
* 最后得到的结果是：
    * C0消费`test-0,test-1,test-2,test-3`
    * C1消费`test-4,test-5,test-6`
    * C2消费`test-7,test-8,test-9`
通过上面的例子，可以看出，消费者C0多消费了一个分区，这时设想一下，如果该消费组中同时订阅了n个Topic，采用范围分区算法，那么消费者C0将比该组中的其他消费者多消费了n个分区。

#### RoundRobin（轮询）
把所有的Partition和Consumer按照HashCode排序，然后通过轮询算法将各个Partition分配给Consumer。
例如：现在有一个Topic test，10个分区;一个Consumer Group,三个消费者；
* 将Partition通过HashCode排序，假如得到`test-5,test-8,test-2,test-4,test-3,test-6,test-7,test-9,test-0,test-1`
* 将消费者通过HashCode排序，假如是`C2,C0,C1`
* 最后得到的结果是：
    * C2消费`test-5,test-4,test-7,test-1`
    * C0消费`test-8,test-3,test-9`
    * C1消费`test-2,test-6,test-0`
虽然这里C2消费者比其他消费者多一个，但是如果该消费组订阅了多个Topic，那么将会从C0开始分配，也就是说，消费组中的所有消费者消费的分区差距不会超过1。
注意：使用轮询分区分配策略需要满足一下两个条件
    * 每个主题的消费者实例具有相同数量的流
    * 每个消费者订阅的主题必须是相同的

### 什么时候会触发分区分配策略？
以下几种情况会触发分区分配策略（也可称之为Rebalance），分别是：
* 当有新的消费者加入当前Consumer Group
* 有消费者离开当前消费组，例如宕机或者主动关闭
* Topic新增了分区

### 谁来执行分区分配？
kafka提供一种角色`Coordinator`来执行对消费组Consumer Group的管理，当消费者启动的时候，会向`Broker`确定谁是它们组的Coordinator，之后该组中的所有Consumer都会向Coordinator进行协调通信。

### 如何确定Coordinator角色？
消费者向kafka集群中任意一个Broker发送一个`GroupCoordinatorRequest`请求，Broker会返回一个负载最小的BrokerID，并且将其设置成为Coordinator。

### Rebalance（重新负载）
在执行Rebalance之前，需要保证Coordinator是已经确定好了的，整个Rebalance分为两个步骤
    1. joinGroup:所有Consumer都会想Coordinator发送`JoinGroupRequest`请求，请求中带有`group_id`/`member_id`/`protocol_matedata`等信息，Coordinator会从中选择一个Consumer作为Leader，
    并且把leader_id、组成员信息members、序列化后的订阅信息protocol_metadata以及generation_id(类似于zookeeper epoch)发送给消费者。
    2. syncJoin:Leader Consumer在确定好分区分配方案过后，所有消费者向Coordinator发送一个`SyncGroupRequest`请求，当然这里只有Leader Consumer会真正发送分区分配方案，其他的Consumer
    只是打酱油的，Coordinator在收到Leader Consumer的分区分配方案过后，将其封装成一个`SyncGroupResponse`响应返回给所有的Consumer，所有的Consumer在收到分区分配方案过后，自行消费
    方案中指定的分区。
    
## Offset
前面讲到每个Topic有多个Partition，每个Partition中的消息都不一样，并且每个Partition中的消息都会存在一个offset值，在同一个Partition中的offset是有序的，即kafka可以保证同一个Partition
中的消息是有序的，但是这一特性并不跨分区，也即kafka不能保证跨分区的消息的有序性。

### Offset在哪里维护？
kafka提供一个名为`__consumer_offsets_*`的Topic，该Topic就是来保存每个Consumer Group的消费的每个Partition某一时刻的offset信息，该Topic默认有50个分区，那么，kafka是如何将某个Consumer Group
保存到具体的那个分区的呢？其实，kafka是通过这样一个算法来决定该Consumer Group应该保存在那个分区的，公式：`Math.abs("group_id".hashCode())%groupMetadataTopicPartitionCount`。
确定了分区过后，我们可以通过如下命令查看当前Consumer Group的offset信息
```shell
sh kafka-simple-consumer-shell.sh --topic __consumer_offsets --partition 35 --broker-list 192.168.3.224:9092 --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter"
```

## 多个分区在集群中的分配
一个Topic有多个Partition，那么这么多的Partition是如何在Broker中分布的呢？
* 将i个Partition和n个Broker排序
* 将第1个Partition分配到i%n个Broker上

## 消息的存储
我们都知道在kafka中消息都是以日志文件存储在文件系统中的，由于kafka中一般都存储着海量的数据，所有，kafka中的消息日志分区并不是直接对应一个日志文件，而是对应着一个分区目录，
命名为`<topic_name>_<partition_id>`，例如一个名为test的Topic，有三个分区，那么在集群Broker的`/tmp/kafka-log`(该目录是一个临时目录，一般线上环境都会更改此目录)中就有三个目录，分别是`test-0`/`test-1`/`test-2`

### 消息的文件存储机制
我们知道了Partition的存储是指向一个目录的，其实目录并不具备数据存储的能力，那么kafka中的消息是如何存在于Partition中的呢。kafka为了以后消息的清理以及压缩的便利性和处于性能方面的考虑，
引入一个`LogSegment`的逻辑概念，但实际上消息是以文件的形式存在于Partition目录中的。在一个Partition中可以存在多个`LogSegment`，一个`LogSegment`由一下三个文件组成：
    1. 00000000000000000000.index:offset索引文件，对应offset和物理位置position
    2. 00000000000000000000.timeindex:时间索引文件，映射时间戳和offset的对应关系
    3. 00000000000000000000.log:日志文件，存储Topic消息，包含内容有offset、position、timestamp、消息内容等等
每个`LogSegment`分段的大小可以通过server.properties中的`log.segment.bytes`属性设置，默认是1GB。
`LogSegment`命名的规则是由一个最大支持64位long大小的20位数字字符串组成，每个Partition中的第一个`LogSegment`从0开始，后面的`LogSegment`命名为上一个`LogSegment`消息中最后一个offset+1，
我们可以通过`sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test-0/00000000000000000000.log --print-data-log`命令查看分区0的第一个`LogSegment`。

### LogSegment中index文件和log文件的关联关系
我们知道每个LogSegment都是由index,timeindex,log三个后缀结尾的文件组成。可以通过以下命令查看索引文件内容：
```shell
sh kafka-run-class.sh kafka.tools.DumpLogSegments --files /tmp/kafka-logs/test-0/00000000000000000000.index --print-data-log
```
index和log文件的对应关系如下图：

## 通过offset查找Message的原理分析

## 消息的写入性能
为什么kafka会有这么高的吞吐量，其原因在于kafka在很多地方做了优化，那么在网络传输和磁盘IO上，有很大的优化空间
* 顺序写入：kafka采用顺序写入的方式将消息持久化到磁盘，避免了常规随机写入数据寻址等一系列操作带来的性能损耗。
* 零拷贝：一般情况下，我们将文件数据发送到网络上时，需要将文件冲磁盘读取到操作系统内核空间中，然后拷贝到用户空间，最后将数据发送到网卡，通过网络传输，在kafka中，采用零拷贝的方式，直接
将数据从内核空间发送到网卡通过网络传输，节省了用户空间这一步骤，在性能上有一定的提升。
    * 在linux系统中使用sendFile实现零拷贝
    * 在Java中使用FileChannel.transfer实现零拷贝