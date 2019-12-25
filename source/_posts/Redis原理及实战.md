---
title: Redis原理及实战
date: 2019-12-23 19:14:14
categories: Redis
tags:
- Redis
- Jedis
---
相信大家对Redis并不陌生，我们在做数据缓存、分布式锁、分布式事务等业务中经常会见到Redis的身影，今天带大家来深入了解一下Redis的原理以及在实际应用场景中的使用。

## 初识Redis
Redis可以看成是NoSQL数据库，也可以看成是缓存中间件，Redis缺省有16（分别是0-15）个库，每个库中包含多个key，每个key对应的数据类型有五种，分别是String、List、Hash、Set、SortSet

### 安装
* [下载](http://download.redis.io/releases/redis-5.0.7.tar.gz) 注意：第二位数字为偶数则代表稳定版，奇数为非稳定版
* 解压：`tar -zxvf redis-5.0.7.tar.gz`
* 进入Redis目录：`cd redis-5.0.7`
* 编译：`make` 注意：这里或许会报错，根据提示安装依赖的库即可解决，例如gcc
* 测试编译：`make test`
* 安装：`make install {PREFIX=/path}`

## Redis数据类型
Redis支持五种数据类型，分别是String、List、Hash、Set、SortSet，五种类型的特性如下：

### String(字符串)
字符串是Redis中最基本的数据类型，它能存储任何字符数据，例如JSON，Base64编码的图片等，String最大支持存储512M的字符数据。

#### 内部数据结构
String支持三种数据类型，分别是int、浮点数据和字符数据，int数据类型使用int存储，浮点数据和字符数据使用SDS（Simple Dynamic String）存储，SDS是在C的标准字符串结构上作了封装，Redis3.2
有五种sdshdr类型，目的是根据存储的字符串长度选择不同的sdshdr，不同的sdshdr占用的内存大小各有不同，这样就达到了节省内存开销的目的。

### List(列表)
List列表是一个有序的字符串列表，由于List底层采用的是双向链表的数据结构，所以不管List列表中的数据有多大，向列表的两端存取数据都是很快的，常用操作也是向列表的两端存取数据。

#### 内部数据结构
在3.2之前，List中元素个数较少或者单个元素长度较小的时候，采用ZipList数据接口存储数据，当List中元素个数或者单个元素长度较大的时候，就会采用LinkedList存储。这两种数据结构各有
优缺点，LinkedList在两端数据的存储复杂度较低，但是内存开销比较大；ZipList内存开销比较小，但是插入和删除需要频繁申请内存。
在3.2之后，Redis在数据存储结构上做了优化，采用QuickList数据结构，QuickList其实是LinkedList和ZipList数据结构的整合，QuickList任然是一个LinkedList，只是每个元素都是一个ZipList，
每个元素都能存储多个数据元素；即QuickList是多个ZipList组成的LinkedList

### Hash(可以认为是Java中的Map)
Hash可以看成是Java中的Map，由一个Sting类型的key和多个String类型的field和value组成。适合存储对象。

#### 内部数据结构
Hash底层数据结构可以使用ZipList和HashTable，当Hash中field和value的字符串长度都小于64字节，一个Hash的field和value的个数小于512个时，使用ZipList数据结构存储

### Set(集合)
Set存储一个无序不能重复的元素集合，最多可以存储232-1个元素，集合和列表的最大区别就是唯一性和有序性。

#### 内部数据结构
Set底层数据结构有IntSet和HashTable，当所有元素是int类型时，这使用IntSet，否则使用HashTable（只用Key，Value为null）

### SortSet(有序集合)
SortSet和Set的区别就是增加了排序功能，在集合的基础上，有序集合为集合中的每个元素绑定了一个score（分数）。有序集合中的元素和集合一样是唯一的，但是元素的score是可以重复的。
我们可以通过score进行排序，查找，删除等操作。

#### 内部数据结构
SortSet采用ZipList或者SkipList+HashTable数据结构存储数据。

## Redis过期时间
Redis中可以为一个key设置一个过期时间，当设置了过期时间的key到期过后会被删除。
在Redis中，为某个key设置过期时间有三种方式：
1. `EXPIRE key seconds`:为key设置过期时间，单位为妙，返回值1表示设置成功，0表示失败（例如：键不存在）。
2. `PEXPIRE key millis`:为key设置过期时间，单位为毫秒
3. `setex key seconds value`:该方式为字符串独有，设置key的过期时间，单位为秒
查看一个key的有效期使用`TTL key`或者`PTTL key`，两种方式分别对应`EXPIRE`和`PEXPIRE`两种方式设置的过期时间。如果`TTL`或者`PTTL`返回-2，则表示键不存在，-1则表示没有设置过期时间，其他数字
则表示过期剩余时间。
如果想让某个设置了过期时间的key恢复成持久的key，可以使用`PERSIST key`，成功返回1，失败返回0.

### 过期删除原理
在Redis中，对于已经过期的key的删除有两种方式，如下：
1. 积极方式：采用随机抽取算法，周期性的对已经设置了过期时间的key随机抽取一批key，将已经过期的key进行删除，该方式有一个缺陷，并不能确保所有过期的key被删除。具体流程如下：
    1. 随机抽取20个带有timeout的key
    2. 将已经过期的key进行删除
    3. 如果被删除（已过期）的key超过抽取总数的25%（5个），则重复执行该操作
2. 消极方式：当key被访问的时候判断是否过期，如果过期则删除它，该方式有一个缺陷，对于没有被查询到的已经过期的key，会常住内存。
Redis采用以上两种过期删除方式来互补，达到过期key的删除逻辑。

## 发布/订阅（publish/subscribe）
Redis提供发布/订阅的功能，可以在多个进程之间进行消息通信。`PUBLISH channel.1 message`表示向channel.1发送了一条消息，内容为message，该命令返回一个数值，表示订阅了当前channel
的订阅者数量，当返回0的时候表示该channel没有订阅者；订阅者使用`SUBSCRIBE channel.1 channel.2 ...`订阅channel.1，一个channel可以有多个订阅者，一个消费者也可以订阅多个消息，当发送者发送一条
详细到一个channel，该channel中的所有订阅者都会受到该条消息。需要注意的是：发送到channel中的消息不会持久化，也就是说，订阅者只能收到订阅过后的消息，订阅之前该channel所产生的消息不能收到。
channel可以分为两类，普通的channel和Pattern Channel（规则匹配），例如：现在有两个channel，分别是普通channel `abc`和Pattern Channel `*bc`，发送者向abc中发送一条消息`PUBLISH abc hello`,
首先`abc`这个channel能收到一条消息hello，`*bc`也能匹配到abc这个channel，所以`*bc`也能收到这条消息hello。

## Redis数据的持久化
在Redis中，数据的持久化有两种方式
* RDB：Fork一个子进程根据配置的规则定时的将内存中的数据写入到磁盘中。
* AOF(Append Only File)：每次执行过后将命令追加到AOF文件中，类似于MySQL的binlog。

### RDB
当符合RDB持久化条件时，Redis会Fork和主进程一样的子进程，先将内存中的所有数据写入到一个临时文件中，当内存中的所有数据都写入完毕过后，再将之前的备份文件替换。该方式的缺点是最后一次持久化
过后的数据有可能会丢失，也就是说，两次数据的持久化间隔产生的数据有可能丢失。
什么叫符合RDB持久化条件呢？
* 当满足配置文件的规则时：在redis.conf文件中配置`save 900 1`,`save 300 10`,`save 60 10000`，以上配置，save后面的第一个参数表示时间（单位秒），第二个表示键的个数，并且满足以上任意
一个配置都会执行，以上配置表示的意思就是：当900秒内有一个键发送变动或者300秒内有10个键发生变动或者60秒内有10000个键发生变动都会触发RDB快照。
* 客户端发送了SAVE或者BGSAVE命令：当我们需要对Redis服务进行重启的时候，我们可以操作SAVE或者BGSAVE命令手动执行RDB快照，SAVE和BGSAVE命令的区别在于，一个是同步执行，一个是异步执行，
同步执行会阻塞其他客户端的请求，而BGSAVE则不会阻塞。我们还可以通过LASTSAVE命令来查看最后一次执行RDB快照的时间。
* 客户端发送了FLUSHALL命令：该操作依赖配置规则，如果没有配置RDB的执行规则，该命令也不会触发RDB快照的执行。
* 执行复制（Replication）：该操作一般指在主从模式下，Redis会在复制初始时执行RDB快照。

### AOF
当我们的业务需求对Redis的使用不限于缓存时，可能会使用Redis存储一些比较重要的数据，这个时候我们可以开启AOF来降低RDB持久化方式对内存数据的丢失，当然，开启AOF对Redis对外提供服务的性能
是有一定的影响的，但是这种影响一般能接受，解决办法可以使用一些写入性能较高的磁盘。默认情况下，Redis并没有开启AOF持久化方式，我们可以在配置文件中配置AOF是否开启。`appendonly yes`将
appendonly属性值改为yes，则表示开启AOF持久化。还可以指定AOF持久化到磁盘的文件名称`appendfilename "appendonly.aof"`。
查看appendonly.aof文件，我们可以发现，里面保存了客户端操作Redis的所有事务操作（增删改）命令，但其实有的时候我们对Redis的操作是针对同一个key的，也就是说，其实真正有用的数据是最新
存在于内存中的数据，而AOF持久化文件则保存了各个key的变动轨迹，有很多命令轨迹是没有用的，所以这个时候需要对这样一个问题进行优化。
Redis也考虑到了这一点，我们可以通过配置的方式来解决这一问题，在redis.conf配置文件中配置`auto-aof-rewrite-percentage 100`和`auto-aof-rewrite-min-size 64mb`。
* `auto-aof-rewrite-percentage 100`:表示当前AOF文件的大小超过上一次重写AOF文件大小的百分之多少时会再次执行重写，如果之前没有重写过，则以启动时AOF文件的大小为准。
* `auto-aof-rewrite-min-size 64mb`:表示限制允许重写最小AOF文件的大小。
当然我们也可以通过手动执行`BGREWRITEAOF`命令的方式让AOF文件重写。
AOF方式的数据恢复会一个一个将AOF文件中的命令在Redis服务器上执行，性能上会比RDB方式慢。

#### AOF重写原理
同样的，为了不影响对外提供服务，AOF重写时主进程会Fork一个子进程来执行，该操作并不是和之前AOF追加的方式，而是类似于RDB的方式，将内存中的数据遍历出来，然后解析成set命令保存到AOF文件中，
在这期间，由于Redis还持续对外提供服务，那么在期间客户端发送的操作执行该如何保证数据同步呢，Redis的解决方案是在执行AOF重写的过程中，主进程接受到的所有客户端的事务操作会缓存到
`aof_rewrite_buf`缓存（单独开辟一块内存空间来存储执行AOF重写期间收到的客户端命令）中，当重写操作执行完成过后，在将`aof_rewrite_buf`缓存中将所有命令追加到重写过后的文件中，
当然这个文件也是一个临时文件，当以上操作都执行完毕过后，Redis会把之前旧的AOF文件替换，这样做的好处在于，就算在AOF重写时失败了，也不会影响之前已经持久化的AOF文件。

## Redis的内存回收策略
内存是有限且昂贵的，Redis作为一个内存缓存中间件，必须要考虑如何合理有效的使用内存空间。例如：当内存不足时，如何保证Redis程序的正常运行。