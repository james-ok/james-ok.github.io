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
