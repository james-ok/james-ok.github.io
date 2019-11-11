---
title: Zookeeper基本使用及原理分析
date: 2019-11-08 11:09:47
categories: Zookeeper
tags:
- Zookeeper
---
Zookeeper相信大家都不陌生，应用场景也颇为广泛，注册中心、配置中心、分布式锁这些场景都有它的身影。

## Zookeeper是什么
Zookeeper是一个分布式协调服务，由雅虎创建，最初的目标是解决分布式服务有序性问题，例如分布式锁，虽然分布式服务协调的问题解决了，单Zookeeper本身的单点问题出现了，所以就有了Zookeeper集群来达到Zookeeper本省的高可用性，
那么Zookeeper集群节点间的数据同步该如何解决呢？

## Zookeeper安装

### 单机模式
1. Zookeeper[下载](https://www-eu.apache.org/dist/zookeeper/zookeeper-3.4.14/zookeeper-3.4.14.tar.gz)
2. 解压`tar -zxvf zookeeper-3.4.14.tar.gz`
3. 将config文件夹下的zoo_simple.conf修改为zoo.conf
4. 启动Zookeeper `sh bin/zkServer.sh start`

### 集群模式
1. 在Zookeeper工作目录（zoo.conf配置文件汇总的dataDir指定的目录）下创建myid文件，配置当前集群id
2. 在zoo.conf配置文件中加入server.id=ip port1 port2，其中id为Zookeeper id，ip为Zookeeper IP，port1为数据同步通信所使用的端口，port2为leader选举所使用的端口，例如：`server.1 192.168.3.207 2888 3888`
注意：第一次启动一个节点时，会报错，原因是没有其他的节点存在

## Zookeeper配置项解释
* tickTime：Zookeeper服务器和客户端之间的心跳间隔，单位毫秒
* initTime：集群模式下服务器接受客户端（这里的客户端并不是客户端连接Zookeeper的客户端，而是follower节点和leader节点中的follower节点）连接的最长心跳次数
* dataDir：Zookeeper的数据存储目录
* clientPort：对外暴露的客户端连接端口，默认是2181
* syncLimit：leader节点和follower节点之间的数据同步超时时间长度，最长不能超过多少个心跳次数
* server：集群模式下所有节点信息配置

### 节点属性
通过get命令可以看到Zookeeper的节点属性
```shell
[zk: localhost:2181(CONNECTED) 2] get /zookeeper

cZxid = 0x0
ctime = Thu Jan 01 08:00:00 CST 1970
mZxid = 0x0
mtime = Thu Jan 01 08:00:00 CST 1970
pZxid = 0x0
cversion = -1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1
```
#### Zxid
* cZxid：该节点创建时间所对应的格式时间戳
* mZxid：该节点修改时间所对应的格式时间戳，若该节点没有经过修改，则mZxid和cZxid对应。
* pZxid：该节点和该节点的子节点（不包含孙子节点）的创建和删除时间所对应的格式时间戳
Zxid是一个64位的数字，高32位标注epoch，低32位自增（每一个事务操作递增）

#### version版本号
* cversion：子节点版本号
* dataVersion：节点数据版本号
* aclVersion：节点所拥有的权限版本号

#### time时间
* ctime：当前节点创建时间
* mtime：当前节点修改时间

#### 其他
* ephemeralOwner：临时节点sessionId，如果该节点是临时节点，则该值为客户端和服务器之间的会话ID，如果该节点不是临时节点，该值为0
* dataLength：当前节点数据长度
* numChildren：子节点数量

## Zookeeper节点特性
* 持久化节点
* 临时节点
* 有序节点（有序节点可以是持久化的也可以是临时的）
* 同级节点下名称必须统一
* 临时节点下不允许有子节点

## 集群模式下的数据同步
Zookeeper集群中各个节点都是可以接受客户端请求的，Zookeeper会在每次事务请求提交完毕过后将数据同步到所有follower或者observer节点上，当客户端发送一个读的请求时，Zookeeper直接从当前节点返回数据给客户端。
当客户端发送的是写请求时，Zookeeper会将写的请求转发给leader节点，进行事务性操作，完成后将数据同步到所有follower和observer节点上。

### 集群角色
* leader：整个集群中唯一调度和处理者，保证集群事务处理顺序性。
* follower：处理客户端非事务请求和转发事务请求，参与事务请求投票，参与leader选举投票
* observer：可提升集群性能，不参与任何形式的投票（包括事务请求和leader选举投票）

## Zookeeper改进版2PC提交
### 2PC
在分布式系统中，处理分布式事务有很多种方式，最常见的就是两阶段提交，那么什么是两阶段提交呢？其实两阶段提交就是将一个事务性操作拆分成两个阶段
提交事务请求和执行事务提交
* 提交事务请求：TM（事务管理器）将事务操作转发给当前业务的所有事务参与者，然后等待其相应，AP（事务参与者）在收到请求过后去执行事务操作，并将Undo和Redo写入到事务日志中，最后反馈给TM是否能执行该事务。
* 执行事务提交：TM收到各个事务参与者的返回过后，通过反馈来决定提交或者回滚事务，如果所有参与者返回的ACK为可执行，则提交该事务，如果有一个参与者返回ACK为不能执行，则回滚该事务。

## ZAB协议
ZAB(Zookeeper Atomic Broadcast，Zookeeper原子广播)协议是Zookeeper专门设计用于处理崩溃恢复的原子广播协议，在Zookeeper中，主要依赖ZAB来实现数据一致性问题。
ZAB协议包含两种基本模式：
* 崩溃恢复
* 原子广播
当整个的leader节点出现宕机或者说集群启动的时候，Zookeeper会进入崩溃恢复模式，在重新选举出新的leader过后，有超过半数以上的节点完成数据通过过后，Zookeeper会进入原子广播模式进行数据同步。

### 崩溃恢复原理
集群中一旦leader几点出现宕机或者由于网络原因leader节点和其他follower节点失去联系已经超过半数以上，那么Zookeeper会认为该leader已经不合法了，就会重新选举出一个新的leader，
为了使leader挂了过后系统能过正常工作，ZAB协议需要解决一下两个问题
1. 已经处理的消息不能丢失
当leader在收到所有follower的事务反馈ACK后发送commit指令时宕机，假如向follower1发送commit成功过后宕机，这个时候follower1已经执行了该事务请求，而follower2并没有收到，这个时候ZAB需要重新选举新的leader
并且保证该事务请求需要被所有节点所执行。
2. 被丢弃的消息不能再次出现
当leader在收到客户端的事务请求生成proposal过后出现宕机，这个时候这个事务性消息并没有被广播出去，所有的follower节点均没有收到来自leader的事务请求，这个时候ZAB需要保证新选举的leader需要将该事务请求要被丢弃，
就算是原来的leader重启注册完成过后也需要将该proposal丢弃，这样就能和集群保持数据一致。
针对以上两个要求，我作出如下假设：
如果ZAB的leader选举算法能够保证leader在出现宕机过后选举出目前ZXID最大的follower节点作为leader节点，那么应该就能保证之前已经处理的消息不被丢失，同时，由于每次leader选举，epoch会在原来的基础上自增1（epoch += 1）
这样就算旧的leader重启它也不会再被选举成为leader。新的leader会将所有的旧的epoch没有被commit的消息全部清除掉。

### 原子广播原理
当leader在收到消息过后，会给当前消息赋予一个全局唯一的64位自增id(zxid)，后续通过zxid可以实现因果有序的特征。然后将消息广播给所有follower节点，follower将消息写入到磁盘过后给leader返回一个ACK，
当集群节点中超过半数以上的节点返回ACK过后，leader会像所有节点发送commit请求。

## Leader选举
Zookeeper选举方式有多种，默认是fast选举，leader选举有两个场景
* 集群启动时
* leader出现宕机时

### 服务启动时的leader选举
1. 集群各个节点在启动时，所有节点状态都为LOOKING（节点状态有4中，分别是LOOKING，LEADING，FOLLOWING，OBSERVER）
2. 每个server发出一个投票给集群中的其他节点，例如当前有三个节点，各个节点的基本信息如下：
    * server1:zxid=0,myid=1
    * server2:zxid=0,myid=2
    * server3:zxid=0,myid=3
    server1将投票信息（01）发送给其他两个节点
3. 各个节点在收到投票过后，首先进行校验，如状态校验等
4. 各个节点将自己投票和别人的投票进行PK
    * 首先进行zxid比较，zxid最大的将作为当前集群中的leader
    * 如果zxid一样，则进行myid比较，myid最大的将作为leader
    对于server1而言，当前自己的投票为01，收到的投票为02，这个时候server1将更新自己的投票数据为02，而对于server2而言，自己的投票为02，收到的投票为01，这个时候server2不需要更新自己的投票。
5. 统计投票，每次投票过后，都会统计投票，只要超过半数以上的节点投票一致，则被投票对象将被选举成为leader
6. 最后改变节点状态，leader节点状态为LEADING，其他节点均为FOLLOWING

### 运行时Leader选举
当集群中的leader宕机或不可用时，这个时候集群已经不能对外提供服务，而是进入新一轮的leader选举，运行时Leader选举和服务启动时的Leader选举过程基本一致，首先会将epoch自增1，然后将所有节点的状态改为LOOKING，进行新一轮的leader选举。