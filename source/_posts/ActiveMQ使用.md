---
title: ActiveMQ使用
date: 2019-11-30 10:09:29
categories: ActiveMQ
tags:
- ActiveMQ
- JMS
---
相信大家遇到过这样的场景，用户注册这个简单的功能里面集成了太多不是很重要步骤，但又不得不做。比如发送邮件、发放优惠券、发送推销短信、记录日志，这样就导致了我们注册功能特别繁重，
极大的拉低了接口性能，给用户带来体验度大大降低，明明就一个注册用户信息持久化的功能居然需要做这么多不是主线流程的事情。当你遇到这样的业务场景的时候就可以考虑使用消息队列来实现
解耦，经过优化过后，我们的注册功能就只需要将用户信息持久化到数据库，然后向MQ中间件发送一条消息，然后返回，如果说之前的每个操作需要一秒，那总得就需要5S，但是经过使用MQ解耦过后
只需要1S左右，大大提升了用户体验。

## JMS
JMS(Java Message Service)是Java为各个消息中间件提供的一套统一API规范，其目的是规避各个中间件协议、接口的不同而带来的不便。以下是JMS连接流程图：
![JMS连接流程图](ActiveMQ使用/JMS流程.png)
### 消息传递模式
JMS提供两种常见的消息传递模式或域，分别是：
* P2P(点对点的消息传递模式):一个消息生成者对应一个消费者，两者之间不存在时间上的相关性（即，就算消费者不在线，生产者照样可以发送消息到`Broker`上，等消费者上线过后继续消费）
* PUB/SUB(发布订阅的消息传递模式):一个消息生产者对应多个消息消费者，两者之间存在时间上的相关性（即，消费者只能收到订阅过后并且在线时生产者发送的消息，但不是绝对，JMS允许
消费者创建持久化订阅，持久订阅允许消费者消费他不在线时发送的消息）

### 消息类型或结构组成
消息的结构由消息头、消息体、属性组成
* 消息头：消息头包含消息识别和路由信息
* 消息体：一般是我们发送的消息内容
* 消息属性：属性分为应用设置的属性、标准属性、中间件定义的属性
JMS提供六种消息类型，分别是：
* TextMessage:文本消息
* MapMessage:键值对消息，键是String类型，值可以是Java的任何类型
* BytesMessage:字节流消息
* StreamMessage:输入输出流消息
* ObjectMessage:可序列化对象消息
* Message:空消息，不包含有消息体，只有消息头和属性

## ActiveMQ
### 安装

* [下载](http://www.apache.org/dyn/closer.cgi?filename=/activemq/5.15.10/apache-activemq-5.15.10-bin.tar.gz&action=download)
* 解压:`tar -zxvf apache-activemq-5.15.9-bin.tar.gz`
* 启动:`sh activemq start`
* 访问:[http://localhost:8161](http://localhost:8161)

### P2P(Queue)消息传递方式

* 消息生产者
```java
public class QueueProvider {
    public static void main(String[] args) throws JMSException {
        //创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://192.168.3.224:61616");
        //创建连接
        Connection connection = connectionFactory.createConnection();
        //建立连接
        connection.start();
        //创建会话
        Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
        //创建目的地
        Destination destination = new ActiveMQQueue("testQueue");
        //创建消息生产者
        MessageProducer producer = session.createProducer(destination);
        //创建消息
        TextMessage message = new ActiveMQTextMessage();
        message.setText("Hello World");
        //发送消息
        producer.send(message);
        //提交消息事务，该方法只有在事务型会话时使用
        session.commit();
        //关闭会话
        session.close();
        //关闭连接
        connection.close();
    }
}
```

* 消息消费者
```java
public class QueueConsumer {
    public static void main(String[] args) throws JMSException {
        //创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://192.168.3.224:61616");
        //创建连接
        Connection connection = connectionFactory.createConnection();
        //建立连接
        connection.start();
        //创建会话
        Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
        //创建目的地
        Destination destination = new ActiveMQQueue("testQueue");
        //创建消费者
        MessageConsumer consumer = session.createConsumer(destination);
        //消费消息
        TextMessage message = (TextMessage)consumer.receive();
        //输出消息（处理消息）
        System.out.println(message.getText());
        //确认消息，该方法只有在事务型会话时使用
        session.commit();
        //关闭会话
        session.close();
        //关闭连接
        connection.close();
    }
}
```
消息消费还可以使用监听器的方式，代码如下(片段)：
```java
//...
//创建消费者
MessageConsumer consumer = session.createConsumer(destination);
MessageListener messageListener = new MessageListener() {
    public void onMessage(Message message) {
        TextMessage textMessage = (TextMessage) message;
        System.out.println(textMessage);
    }
};
//设置消息监听
consumer.setMessageListener(messageListener);
//确认消息，该方法只有在事务型会话时使用
session.commit();
//...
```

### PUB/SUB(发布/订阅)消息传递方式

* 消息生产者
```java
public class TopicProvider {
    public static void main(String[] args) throws JMSException {
        //创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://192.168.3.224:61616");
        //创建连接
        Connection connection = connectionFactory.createConnection();
        //建立连接
        connection.start();
        //创建会话
        Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
        //创建目的地
        Destination destination = new ActiveMQTopic("testTopic");
        //创建消息生产者
        MessageProducer producer = session.createProducer(destination);
        //创建消息
        TextMessage message = new ActiveMQTextMessage();
        message.setText("Hello World");
        //发送消息
        producer.send(message);
        //提交消息事务，该方法只有在事务型会话时使用
        session.commit();
        //关闭会话
        session.close();
        //关闭连接
        connection.close();
    }
}
```

* 消息消费者
```java
public class TopicConsumer {
    public static void main(String[] args) throws JMSException {
        //创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://192.168.3.224:61616");
        //创建连接
        Connection connection = connectionFactory.createConnection();
        //建立连接
        connection.start();
        //创建会话
        Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
        //创建目的地
        Destination destination = new ActiveMQTopic("testTopic");
        //创建消费者
        MessageConsumer consumer = session.createConsumer(destination);
        //消费消息
        TextMessage message = (TextMessage)consumer.receive();
        //输出消息（处理消息）
        System.out.println(message.getText());
        //确认消息，该方法只有在事务型会话时使用
        session.commit();
        //关闭会话
        session.close();
        //关闭连接
        connection.close();
    }
}
```
前面讲到JMS允许消费者创建持久化订阅，持久订阅允许消费者消费他不在线时发送的消息，实现这一需求需要改动消费者三个地方，分别是：
```java
public class TopicConsumer {
    public static void main(String[] args) throws JMSException {
        //创建连接工厂
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://192.168.3.224:61616");
        //创建连接
        Connection connection = connectionFactory.createConnection();
        //配置客户端ID
        connection.setClientID("MrAToo-001");//[1]
        //建立连接
        connection.start();
        //创建会话
        Session session = connection.createSession(Boolean.TRUE, Session.AUTO_ACKNOWLEDGE);
        //创建目的地
        Topic destination = new ActiveMQTopic("testTopic");//[2]
        //创建消费者
        MessageConsumer consumer = session.createDurableSubscriber(destination,"MrAToo-001");//[3]
        //消费消息
        TextMessage message = (TextMessage)consumer.receive();
        //输出消息（处理消息）
        System.out.println(message.getText());
        //确认消息，该方法只有在事务型会话时使用
        session.commit();
        //关闭会话
        session.close();
        //关闭连接
        connection.close();
    }
}
```

* 标注[1]:`connection.setClientID("MrAToo-001");`为客户的设置一个ID
* 标注[2]:`Topic destination = new ActiveMQTopic("testTopic");`接受参数使用`Destination`的子类`Topic`
* 标注[3]:`MessageConsumer consumer = session.createDurableSubscriber(destination,"MrAToo-001");`调用`session.createDurableSubscriber`方法

通过上面的配置，在`Broker`上会存在一条客户端记录
![Broker截图](ActiveMQ使用/ActiveMQ-Broker.jpg)

## JMS消息的可靠方式
正常情况下，消息消费有三个阶段：消息接收、消息处理、消息确认，在消息在被收到处理完毕并且确认过后被视为消息被成功消费。

### 事务型会话
在事务型会话中，消息生产者和消息消费者均需要调用`session.commit`方法，对于生产者而言，`commit`表示消息提交，只有提交的消息才会存在`Broker`中，才能被消费者消费
对于消费者而言，`commit`表示消息被确认，只有被确认的消息，`Broker`才不会再次重发消息，很大程度上能避免消息重发的问题（但是并不能正在意义上解决消息的重复消费）。
相反的还有`session.rollback`，该方法表示对之前做的所有操作进行作废处理，对于生成者而言，已经发送的消息回滚。对于消费者而言，当前消息标记为未接受，`Broker`会重发消息。
> 注意：必须保证生产者和消费者都是事务型会话

### 非事务型会话
在非事务型会话中，消息何时被确认取决于创建会话时的应答模式(acknowledgement mode)，应答模式有三种：
* Session.AUTO_ACKNOWLEDGE(自动确认):消息在被收到时自动确认消息
* Session.CLIENT_ACKNOWLEDGE(手动确认):消费者在收到消息过后，处理完毕通过手动调用`message.acknowledge();`进行手动确认，需要注意的是：该方法确认该会话中所有被处理的消息。
* Session.DUPS_ACKNOWLEDGE(消息延迟确认):该选择只是会话迟钝的确认消息的提交

## 消息的持久化存储

### 非持久化
该模式不会将消息存储到可靠的存储介质中（例如：磁盘，DB），只会存在于内存中，如果`Broker`出现宕机，则消息会丢失

### 持久化
该模式会将生产者发送到`Broker`的消息持久化到可靠存储介质中，即使是`Broker`出现宕机，也不会出现消息丢失的情况，但是，由于生产者或者消费者在发送或者确认消息的过程中，
`Broker`需要将消息从可靠存储介质中保存或者删除，从而带来了IO开销，性能上比非持久化存储方式相对来说较低

## 持久化消息和非持久化消息的发送策略

### 消息的同步发送和异步发送

同步发送：消息生产者发送一条消息到`Broker`上，会被阻塞直到`Broker`返回一条确认收到ACK，线程才会被释放，该方式确保了消息的可靠投递，但由于会阻塞，因此会有性能上的损耗。
异步发送：消息生产者发送一条消息过后立即返回，当`Broker`处理完成过后，会回调返回消息确认ACK，这种方式性能相对较高，但丢失消息的可能性相对较高。

默认情况下：非持久化的消息都是异步发送的。持久化消息在非事务模式下是同步发送的。在开启事务的情况下，消息都是异步发送。

除了默认的发送策略外，我们可以设置消息发送的策略，通过在连接URL中添加参数`tcp://localhost:61616?jms.useAsyncSend=true`，也可以调用`ActiveMQConnectionFactory`的`setUseAsyncSend`为`true`

## 消息发送原理分析

源码分析我们从`producer.send(message);`开始，当然前面还有`producer`的创建过程，先不看。`producer.send(message);`方法首先会调用到`ActiveMQMessageProducer`的`send`方法。该方法如下：
```java
class ActiveMQMessageProducer {
    public void send(Destination destination, Message message, int deliveryMode, int priority, long timeToLive, AsyncCallback onComplete) throws JMSException {
        checkClosed();
        if (destination == null) {
            if (info.getDestination() == null) {
                throw new UnsupportedOperationException("A destination must be specified.");
            }
            throw new InvalidDestinationException("Don't understand null destinations");
        }
        ActiveMQDestination dest;
        if (destination.equals(info.getDestination())) {
            dest = (ActiveMQDestination)destination;
        } else if (info.getDestination() == null) {
            dest = ActiveMQDestination.transform(destination);
        } else {
            throw new UnsupportedOperationException("This producer can only send messages to: " + this.info.getDestination().getPhysicalName());
        }
        if (dest == null) {
            throw new JMSException("No destination specified");
        }
        if (transformer != null) {
            Message transformedMessage = transformer.producerTransform(session, this, message);
            if (transformedMessage != null) {
                message = transformedMessage;
            }
        }
        if (producerWindow != null) {
            try {
                producerWindow.waitForSpace();
            } catch (InterruptedException e) {
                throw new JMSException("Send aborted due to thread interrupt.");
            }
        }
        this.session.send(this, dest, message, deliveryMode, priority, timeToLive, producerWindow, sendTimeout, onComplete);
        stats.onMessage();
    }
}
```
该方法中首先判断当前会话状态是否关闭，然后如果`producerWindow`不为null则判断当前消息根据发送窗口的大小判断是否阻塞，最后调用`ActiveMQSession`的`send`方法，该方法如下：
```java
class ActiveMQSession {
    protected void send(ActiveMQMessageProducer producer, ActiveMQDestination destination, Message message, int deliveryMode, int priority, long timeToLive,
                        MemoryUsage producerWindow, int sendTimeout, AsyncCallback onComplete) throws JMSException {
        checkClosed();
        if (destination.isTemporary() && connection.isDeleted(destination)) {
            throw new InvalidDestinationException("Cannot publish to a deleted Destination: " + destination);
        }
        synchronized (sendMutex) {
            // tell the Broker we are about to start a new transaction
            doStartTransaction();//[1]
            TransactionId txid = transactionContext.getTransactionId();
            long sequenceNumber = producer.getMessageSequence();

            //Set the "JMS" header fields on the original message, see 1.1 spec section 3.4.11
            message.setJMSDeliveryMode(deliveryMode);
            long expiration = 0L;
            if (!producer.getDisableMessageTimestamp()) {
                long timeStamp = System.currentTimeMillis();
                message.setJMSTimestamp(timeStamp);
                if (timeToLive > 0) {
                    expiration = timeToLive + timeStamp;
                }
            }
            message.setJMSExpiration(expiration);//[2]
            message.setJMSPriority(priority);//[3]
            message.setJMSRedelivered(false);//[4]

            // transform to our own message format here
            ActiveMQMessage msg = ActiveMQMessageTransformation.transformMessage(message, connection);
            msg.setDestination(destination);
            msg.setMessageId(new MessageId(producer.getProducerInfo().getProducerId(), sequenceNumber));

            // Set the message id.
            if (msg != message) {
                message.setJMSMessageID(msg.getMessageId().toString());
                // Make sure the JMS destination is set on the foreign messages too.
                message.setJMSDestination(destination);
            }
            //clear the brokerPath in case we are re-sending this message
            msg.setBrokerPath(null);

            msg.setTransactionId(txid);
            if (connection.isCopyMessageOnSend()) {
                msg = (ActiveMQMessage)msg.copy();
            }
            msg.setConnection(connection);
            msg.onSend();
            msg.setProducerId(msg.getMessageId().getProducerId());
            if (LOG.isTraceEnabled()) {
                LOG.trace(getSessionId() + " sending message: " + msg);
            }
            //[5]
            if (onComplete==null && sendTimeout <= 0 && !msg.isResponseRequired() && !connection.isAlwaysSyncSend() && (!msg.isPersistent() || connection.isUseAsyncSend() || txid != null)) {
                this.connection.asyncSendPacket(msg);
                if (producerWindow != null) {
                    // Since we defer lots of the marshaling till we hit the
                    // wire, this might not
                    // provide and accurate size. We may change over to doing
                    // more aggressive marshaling,
                    // to get more accurate sizes.. this is more important once
                    // users start using producer window
                    // flow control.
                    //[6]
                    int size = msg.getSize();
                    producerWindow.increaseUsage(size);
                }
            } else {
                if (sendTimeout > 0 && onComplete==null) {
                    this.connection.syncSendPacket(msg,sendTimeout);
                }else {
                    this.connection.syncSendPacket(msg, onComplete);
                }
            }

        }
    }
}
```
该方法中也是先判断当前会话，然后采用同步的方式有序的执行.
* 标注[1]:这里表示开启一个事务
* 标注[2]:设置过期时间
* 标注[3]:设置优先级
* 标注[4]:设置为非重发消息
* 标注[5]:这里的if判断决定消息是异步发送还是同步发送，这里有两种情况：当`onComplete`没有设置，并且发送超时时间小于0，并且不是必须返回`response`响应，并且不是同步发送模式，并且消息是非持久化或者连接器是异步发送模式或者存在事务ID时走异步发送，否则走同步发送
* 标注[6]:异步发送会设置消息发送的大小

### 异步发送

异步发送会调用`ActiveMQConnection`中的`doAsyncSendPacket`方法，该方法中会调用`transport.oneway`方法，那么这里的`transport`是什么呢，其实`transport`在创建`ActiveMQConnection`链接的时候就已经创建了
代码在`ActiveMQConnectionFactory.createActiveMQConnection`方法中，`Transport transport = createTransport();`通过`createTransport`方法创建一个`transport`，代码如下：
```java
class ActiveMQConnectionFactory{
    protected Transport createTransport() throws JMSException {
        try {
            URI connectBrokerUL = brokerURL;
            String scheme = brokerURL.getScheme();
            if (scheme == null) {
                throw new IOException("Transport not scheme specified: [" + brokerURL + "]");
            }
            if (scheme.equals("auto")) {
                connectBrokerUL = new URI(brokerURL.toString().replace("auto", "tcp"));
            } else if (scheme.equals("auto+ssl")) {
                connectBrokerUL = new URI(brokerURL.toString().replace("auto+ssl", "ssl"));
            } else if (scheme.equals("auto+nio")) {
                connectBrokerUL = new URI(brokerURL.toString().replace("auto+nio", "nio"));
            } else if (scheme.equals("auto+nio+ssl")) {
                connectBrokerUL = new URI(brokerURL.toString().replace("auto+nio+ssl", "nio+ssl"));
            }
            return TransportFactory.connect(connectBrokerUL);
        } catch (Exception e) {
            throw JMSExceptionSupport.create("Could not create Transport. Reason: " + e, e);
        }
    }
}
```
通过`TransportFactory.connect`静态方法创建一个`Transport`
```java
class TransportFactory {
    
    private static final FactoryFinder TRANSPORT_FACTORY_FINDER = new FactoryFinder("META-INF/services/org/apache/activemq/transport/");
    
    public static Transport connect(URI location) throws Exception {
        TransportFactory tf = findTransportFactory(location);
        return tf.doConnect(location);
    }
    
    public static TransportFactory findTransportFactory(URI location) throws IOException {
        String scheme = location.getScheme();
        if (scheme == null) {
            throw new IOException("Transport not scheme specified: [" + location + "]");
        }
        TransportFactory tf = TRANSPORT_FACTORYS.get(scheme);
        if (tf == null) {
            // Try to load if from a META-INF property.
            try {
                tf = (TransportFactory)TRANSPORT_FACTORY_FINDER.newInstance(scheme);
                TRANSPORT_FACTORYS.put(scheme, tf);
            } catch (Throwable e) {
                throw IOExceptionSupport.create("Transport scheme NOT recognized: [" + scheme + "]", e);
            }
        }
        return tf;
    }
}
```
这里大概的逻辑是：先从`META-INF/services/org/apache/activemq/transport/`路径下找到指定`scheme`(这里的`scheme`是`tcp`)然后通过反射加载得到`org.apache.activemq.transport.tcp.TcpTransportFactory`，
然后调用`TcpTransportFactory`的`doConnect`(该方法在父类`TransportFactory`中实现)，在该方法中，有这样一句代码`Transport rc = configure(transport, wf, options);`，该方法代码如下：
```java
class TransportFactory {
    public Transport configure(Transport transport, WireFormat wf, Map options) throws Exception {
        transport = compositeConfigure(transport, wf, options);

        transport = new MutexTransport(transport);
        transport = new ResponseCorrelator(transport);

        return transport;
    }
}
```
该方法的作用是包装`Transport`，所以，最终得到的是`ResponseCorrelator(MutexTransport(WireFormatNegotiator(InactivityMonitor(TcpTransport))))`调用链，这是几个`Filter`，这几个`Filter`大致的作用是：
* ResponseCorrelator：用于实现异步请求
* MutexTransport：实现写锁，作用是保证了客户端向`Broker`发送消息时是按照顺序进行的，即同一时间只允许一个请求
* InactivityMonitor：心跳机制，客户端每10s发送一次心跳，服务端每30s接受一次心跳
* WireFormatNegotiator：实现客户端连接`Broker`时先发送协议数据信息
然后调用`TcpTransportFactory`的`createTransport`方法，最终`new TcpTransport`对象，然后回到`ActiveMQConnectionFactory`中，在`createActiveMQConnection`方法中调用了`transport.start`方法，这里在后面讲。
在这里面建立和`Broker`的连接，然后将该连接的`Socket`输出流保存到`dataOut`对象中。

回到`ActiveMQConnection`中的`doAsyncSendPacket`方法中，调用`transport.oneway`方法，其实是调用的`TcpTransport.oneway`方法，这里会通过`dataOut`将消息发送到`Broker`上。

### 同步发送

在ActiveMQ中，同步发送其实也是调用的异步发送的方法，然后阻塞等待异步结果返回。

## 持久化消息和非持久化消息的存储原理
当我们的应用场景不允许消息的丢失的时候，可以采用消息的持久化存储的方式来达到消息的永久存在，ActiveMQ支持五种消息的持久化机制。

### 持久化消息的物种存储方式
* KahaDB：默认ActiveMQ官方推荐的消息持久化方式，配置方式：
```xml
<persistenceAdapter>
    <kahaDB directory="${activemq.data}/kahadb"/>
</persistenceAdapter>
```
* JDBC：将消息持久化到关系型数据库中，支持MySQL，Oracle等主流数据库，该方式会在数据库中生成三张表，分别是：
    * ACTIVEMQ_MSGS:用于存储持久化消息，Queue和Topic消息都在该表中
    * ACTIVEMQ_ACKS:存储持久订阅消息和最后一个持久订阅接收的消息ID
    * ACTIVEMQ_LOCKS:锁表，用来确保同一时刻只有一个`Broker`访问数据
    配置方式：
    ```xml
    <persistenceAdapter>
      <jdbcPersistenceAdapter dataSource="#MySQL-DS " createTablesOnStartup="true" />
    </persistenceAdapter>
    ```
* LevelDB：性能高于KahaDB，并且支持LevelDB+Zookeeper实现数据复制，但是官方不推荐
* Memory：内存，不做消息的持久化时的默认方式
* JDBC With ActiveMQ Journal：该方式是为了优化JDBC的方式，延迟批量将消息持久化到关系型数据库中，`ActiveMQ Journal`使用高缓存写入技术，大大提示性能，当消费者的消费能力很强的时候能大大减少
关系型数据库的事务操作，配置方式：
```xml
<persistenceFactory>
    <journalPersistenceAdapterFactory dataSource="#Mysql-DS" dataDirectory="activemqdata"/>
</persistenceFactory>
```

## 消息消费原理分析

消息消费从`ActiveMQMessageConsumer`的`receive`开始，该方法首先检查连接，然后检查是否设置了`Listener`（`ActiveMQ`消费端只允许一种方式接受消息，原因是多种方式消息消费的事务性不好管控），
然后判断`prefetchSize`和`unconsumeMessages`是否为空，如果为空则向`Broker`发送一个拉取消息的`pull`命令，然后调用`dequeue`方法，该方法从`unConsumeMessages`中获取一个消息（如果`unConsumeMessages`
中没有消息，则会阻塞当前线程直到`Broker`push一个消息或者超时释放），`unConsumeMessages`是一个未消费消息的通道，该通道的作用是每次从`Broker`上拉取`prefetchSize`条消息保存到本地，减少了客户端和服务端
频繁请求造成的网络开销。
继续往下，会调用`beforeMessageIsConsumed(md);`方法，该方法主要作用是做一些消息消费前的一些准备工作，如果ACK类型不是`DUPS_OK_ACKNOWLEDGE`或者不是队列类型（也就是除了`Topic`类型和`DUPS_OK_ACKNOWLEDGE`）
所有的消息先放到`deliveredMessages`链表的开头，并且如果是事务类型，则判断`transactedIndividualAck`，如果是true，表示单条消息直接返回ACK，否则，调用`ackLater`批量应答，消费端在消费
消息过后，先不发送ACK(`pendingACK`)，等到堆积的`pendingACK`达到一定的阈值过后，通过一个ACK指定将之前的所有全部确认，在性能上，这种方式会高很多。
然后继续往下，会调用`afterMessageIsConsumed`方法，该方法主要作用是执行应答，这里有以下几种情况
* 如果消息过期，则返回消息过期的ack
* 如果是事务类型的会话，则不做任何处理
* 如果是AUTOACK或者（DUPS_OK_ACK且是队列），并且是优化ack操作，则走批量确认ack
* 如果是DUPS_OK_ACK，则走ackLater逻辑
* 如果是CLIENT_ACK，则执行ackLater
代码如下：
```java
class ActiveMQMessageConsumer {
    private void afterMessageIsConsumed(MessageDispatch md, boolean messageExpired) throws JMSException {
        if (unconsumedMessages.isClosed()) {
            return;
        }
        if (messageExpired) {
            acknowledge(md, MessageAck.EXPIRED_ACK_TYPE);
            stats.getExpiredMessageCount().increment();
        } else {
            stats.onMessage();
            if (session.getTransacted()) {
                // Do nothing.
            } else if (isAutoAcknowledgeEach()) {
                if (deliveryingAcknowledgements.compareAndSet(false, true)) {
                    synchronized (deliveredMessages) {
                        if (!deliveredMessages.isEmpty()) {
                            if (optimizeAcknowledge) {
                                ackCounter++;

                                // AMQ-3956 evaluate both expired and normal msgs as
                                // otherwise consumer may get stalled
                                if (ackCounter + deliveredCounter >= (info.getPrefetchSize() * .65) || (optimizeAcknowledgeTimeOut > 0 && System.currentTimeMillis() >= (optimizeAckTimestamp + optimizeAcknowledgeTimeOut))) {
                                    MessageAck ack = makeAckForAllDeliveredMessages(MessageAck.STANDARD_ACK_TYPE);
                                    if (ack != null) {
                                        deliveredMessages.clear();
                                        ackCounter = 0;
                                        session.sendAck(ack);
                                        optimizeAckTimestamp = System.currentTimeMillis();
                                    }
                                    // AMQ-3956 - as further optimization send
                                    // ack for expired msgs when there are any.
                                    // This resets the deliveredCounter to 0 so that
                                    // we won't sent standard acks with every msg just
                                    // because the deliveredCounter just below
                                    // 0.5 * prefetch as used in ackLater()
                                    if (pendingAck != null && deliveredCounter > 0) {
                                        session.sendAck(pendingAck);
                                        pendingAck = null;
                                        deliveredCounter = 0;
                                    }
                                }
                            } else {
                                MessageAck ack = makeAckForAllDeliveredMessages(MessageAck.STANDARD_ACK_TYPE);
                                if (ack!=null) {
                                    deliveredMessages.clear();
                                    session.sendAck(ack);
                                }
                            }
                        }
                    }
                    deliveryingAcknowledgements.set(false);
                }
            } else if (isAutoAcknowledgeBatch()) {
                ackLater(md, MessageAck.STANDARD_ACK_TYPE);
            } else if (session.isClientAcknowledge()||session.isIndividualAcknowledge()) {
                boolean messageUnackedByConsumer = false;
                synchronized (deliveredMessages) {
                    messageUnackedByConsumer = deliveredMessages.contains(md);
                }
                if (messageUnackedByConsumer) {
                    ackLater(md, MessageAck.DELIVERED_ACK_TYPE);
                }
            }
            else {
                throw new IllegalStateException("Invalid session state.");
            }
        }
    }
}
```

## unconsumedMessages数据获取过程

`unconsumedMessages`未消费的消息通道是在什么时候被赋值的，这应该从连接的创建过程说起，在`ActiveMQConnectionFactory#createActiveMQConnection`连接创建是调用了`TcpTransport#start`方法（实际上是`ServiceSupport#start`），该方法中
调用`TcpTransport#doStart`，在该方法中通过`connect`方法和`Broker`创建连接，然后调用`TransportThreadSupport#doStart`，该方法中创建了一个线程，线程的内容在`TcpTransport`中，也就是`TcpTransport#run`，然后在
该方法中，只要`TcpTransport`没有停止，则一直调用`TcpTransport#doRun`，然后调用`Object command = readCommand();`从`Broker`上读取一个`command`，最后调用`TransportSupport#doConsume`消费消息。
整个过程调用链如下：
```
ActiveMQConnectionFactory#createConnection -> ActiveMQConnectionFactory#createActiveMQConnection -> ServiceSupper#start -> TcpTransport#doStart -> TransportThreadSupport#doStart
-> TcpTransport#run -> TcpTransport#doRun -> TransportSupport#doConsume -> ActiveMQConnection#onCommand
```
### ActiveMQConnection#onCommand
该方法中所有消息都会调用`visit`方法，该方法接受一个`CommandVisitor`，针对不同的消息做不同的处理，代码如下：
```java
class ActiveMQConnection {
    public void onCommand(final Object o) {
        final Command command = (Command)o;
        if (!closed.get() && command != null) {
            try {
                command.visit(new CommandVisitorAdapter() {
                    @Override
                    public Response processMessageDispatch(MessageDispatch md) throws Exception {
                        waitForTransportInterruptionProcessingToComplete();
                        ActiveMQDispatcher dispatcher = dispatchers.get(md.getConsumerId());
                        if (dispatcher != null) {
                            // Copy in case a embedded broker is dispatching via
                            // vm://
                            // md.getMessage() == null to signal end of queue
                            // browse.
                            Message msg = md.getMessage();
                            if (msg != null) {
                                msg = msg.copy();
                                msg.setReadOnlyBody(true);
                                msg.setReadOnlyProperties(true);
                                msg.setRedeliveryCounter(md.getRedeliveryCounter());
                                msg.setConnection(ActiveMQConnection.this);
                                msg.setMemoryUsage(null);
                                md.setMessage(msg);
                            }
                            dispatcher.dispatch(md);
                        } else {
                            LOG.debug("{} no dispatcher for {} in {}", this, md, dispatchers);
                        }
                        return null;
                    }

                    @Override
                    public Response processProducerAck(ProducerAck pa) throws Exception {
                        if (pa != null && pa.getProducerId() != null) {
                            ActiveMQMessageProducer producer = producers.get(pa.getProducerId());
                            if (producer != null) {
                                producer.onProducerAck(pa);
                            }
                        }
                        return null;
                    }

                    @Override
                    public Response processBrokerInfo(BrokerInfo info) throws Exception {
                        brokerInfo = info;
                        brokerInfoReceived.countDown();
                        optimizeAcknowledge &= !brokerInfo.isFaultTolerantConfiguration();
                        getBlobTransferPolicy().setBrokerUploadUrl(info.getBrokerUploadUrl());
                        return null;
                    }

                    @Override
                    public Response processConnectionError(final ConnectionError error) throws Exception {
                        executor.execute(new Runnable() {
                            @Override
                            public void run() {
                                onAsyncException(error.getException());
                            }
                        });
                        return null;
                    }

                    @Override
                    public Response processControlCommand(ControlCommand command) throws Exception {
                        return null;
                    }

                    @Override
                    public Response processConnectionControl(ConnectionControl control) throws Exception {
                        onConnectionControl((ConnectionControl)command);
                        return null;
                    }

                    @Override
                    public Response processConsumerControl(ConsumerControl control) throws Exception {
                        onConsumerControl((ConsumerControl)command);
                        return null;
                    }

                    @Override
                    public Response processWireFormat(WireFormatInfo info) throws Exception {
                        onWireFormatInfo((WireFormatInfo)command);
                        return null;
                    }
                });
            } catch (Exception e) {
                onClientInternalException(e);
            }
        }

        for (Iterator<TransportListener> iter = transportListeners.iterator(); iter.hasNext();) {
            TransportListener listener = iter.next();
            listener.onCommand(command);
        }
    }
}
```
如果传入的消息是`MessageDispatch`，则会调用`processMessageDispatch`方法，在该方法中最终会调用`ActiveMQMessageConsumer`中的`dispatch`方法，`unConsumedMessages`的值就是在该方法中`enqueue`的。

总结：消费者在启动的时候会创建一个线程不断的从客户端和`Broker`的`Socket`连接中读取数据，然后交给`TransportListener`（这里的实现是`ActiveMQConnection`）处理，
消息的消费其实是从一个未消费的消息通道`unConsumedMessages`里面拿的，拿消息之前会判断当前`unConsumedMessages`中是否存在未消费的消息，如果不存在消息并且`prefetchSize`等于0，
则向`Broker`发送一条`pullCommand`指令，然后调用`dequeue`方法（该方法会被阻塞知道拿到消息后返回），然后`Broker`会向客户端`push`指定条数（prefetchSize）的消息（这里是异步实现，消息会
被`Transport`线程读取，然后交给`ActiveMQConnection#onCommand`监听器分发消息，最终会把消息`enqueue`到`unConsumedMessages`中），当`unConsumedMessages`有消息过后，`dequeue`方法解除
阻塞，返回消息，然后执行消息确认过程。

## prefetchSize与optimizeAcknowledge

* prefetchSize:窗口机制（消息的批量拉取）
不同的类型的队列，prefetchSize 的默认值也是不一样的，如下：
    1. 持久化队列和非持久化Queue（队列），prefetchSize默认值为1000；
    2. 持久化 topic ，prefetchSize 默认值为100；
    3. 非持久化消息，prefetchSize 默认值为 Short.MAX_VALUE -1
    
配置方式：
```
Destination destination = session.createQueue("myQueue?consumer.prefetchSize=88");
```

* optimizeAcknowledge:消息优化确认，优化ACK，只有`optimizeAcknowledge`为true时，`prefetchSize`和`optimizeAcknowledgeTimeout`才有意义。消息的批量确认，也是一种减少网络开销的一种手段，
如果我们不开启优化ACK，那么`Broker`push一批消息到客户端过后，客户端消费一条消息向`Broker`确认一次，`Broker`向客户端push一条消息，这样达不到批量的效果（假批量），所以一般情况下，这两个
配置是同事存在的，默认消息消费超过`65%`会发送一次批量确认（也就是1000*.65=650）。

配置方式：
```
ConnectionFactory connectionFactory= new ActiveMQConnectionFactory("tcp://localhost:61616?jms.optimizeAcknowledge=true&jms.optimizeAcknowledgeTimeOut=10000");
```

## 消息的重发机制
正常情况下，触发消息重发的有两种情况
* 事务性会话中，没有调用`session.commit`或者调用`session.rollback`
* 非事务性会话中，没有调用`acknowledge`或者调用`recover`

一个消息被`redelivedred`超过6次，客户端会给`Broker`发送一个`poisonACK`，告诉`Broker`不要再重发消息了，然后`Broker`会将该条消息放入到DLQ（死信队列）中。

## 死信队列

ActiveMQ中默认的死信队列是`ActiveMQ.DLQ`，没有特殊的的配置，重发超过6次的消息都会被放到该队列中，默认情况下，如果持久消息过期后，也会被放到该死信队列中。
默认所有队列的死信队列都是`ActiveMQ.DLQ`，不便于管理，可以通过配置来针对某个队列配置特定的私信队列，配置如下：
```xml
<destinationPolicy> 
    <policyMap> 
        <policyEntries> 
            <policyEntry topic=">" > 
                <pendingMessageLimitStrategy> 
                    <constantPendingMessageLimitStrategy limit="1000"/> 
                </pendingMessageLimitStrategy> 
            </policyEntry>  
            <!-->:表示对所有队列生效，指定队列直接写队列名称--> 
            <policyEntry queue=">"> 
                <deadLetterStrategy> 
                    <!--queuePrefix:设置死信队列前缀--> 
                    <individualDeadLetterStrategy queuePrefix="DLQ." useQueueForQueueMessages="true" processExpired="false"/> 
                    <!--是否丢弃过期消息-->
                    <!--<sharedDeadLetterStrategy processExpired="false" />-->
                </deadLetterStrategy> 
            </policyEntry> 
        </policyEntries> 
    </policyMap> 
</destinationPolicy> 
```

### 死信队列的再次消费
死信队列也是一个队列，在定位到问题原因过后，可以手动消费死信队列的消息。

## ActiveMQ静态网络配置
ActiveMQ支持使用网络配置的方式来达到集群的效果，ActiveMQ中的网络配置方式有两种，静态网络配置和动态网络配置。
* 静态网络配置，配置方式如下

```xml
<networkConnectors>
    <networkConnector uri="static://(tcp://192.168.10.1:61616,tcp://192.168.10.2:61616)" duplex="true"/>
</networkConnectors>
```

* 动态网络配置，该方式使用广播协议将其他的`Broker`连接起来，可以自动发现其他的`Broker`节点，这种方式替代了静态网络连接配置方式。

消息回流：从5.6版本开始，ActiveMQ的网络配置方式集群支持消息回流，该功能解决了当`Broker1`上有需要转发的消息未消费时，将消息回流到原来的`Broker`上。需要配置如下：
```xml
<policyEntry queue=">" enableAudit="false">
    <networkBridgeFilterFactory>
        <conditionalNetworkBridgeFilterFactory replayWhenNoConsumers="true"/>
    </networkBridgeFilterFactory>
</policyEntry>
```
配置消息回流需要配置`networkConnector`节点的`duplex`的属性为true。

## 参考文献

> https://blog.csdn.net/lzb348110175/article/details/100132006