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
* linger.ms:
* max.request.size:

### Consumer可选配置