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

### String(字符串)：
字符串是Redis中最基本的数据类型，它能存储任何字符数据，例如JSON，Base64编码的图片等，String最大支持存储512M的字符数据。

#### 内部数据结构
String支持三种数据类型，分别是int、浮点数据和字符数据，int数据类型使用int存储，浮点数据和字符数据使用SDS（Simple Dynamic String）存储，SDS是在C的标准字符串结构上作了封装，Redis3.2
有五种sdshdr类型，目的是根据存储的字符串长度选择不同的sdshdr，不同的sdshdr占用的内存大小各有不同，这样就达到了节省内存开销的目的。

### List(列表)：
List列表是一个有序的字符串列表，由于List底层采用的是双向链表的数据结构，所以不管List列表中的数据有多大，向列表的两端存取数据都是很快的，常用操作也是向列表的两端存取数据。

#### 内部数据结构
在3.2之前，List中元素个数较少或者单个元素长度较小的时候，采用ZipList数据接口存储数据，当List中元素个数或者单个元素长度较大的时候，就会采用LinkedList存储。这两种数据结构各有
优缺点，LinkedList在两端数据的存储复杂度较低，但是内存开销比较大；ZipList内存开销比较小，但是插入和删除需要频繁申请内存。
在3.2之后，Redis在数据存储结构上做了优化，采用QuickList数据结构，QuickList其实是LinkedList和ZipList数据结构的整合，QuickList任然是一个LinkedList，只是每个元素都是一个ZipList，
每个元素都能存储多个数据元素；即QuickList是多个ZipList组成的LinkedList

### Hash(可以认为是Java中的Map)：
Hash可以看成是Java中的Map，由一个Sting类型的key和多个String类型的field和value组成。适合存储对象。

#### 内部数据结构
Hash底层数据结构可以使用ZipList和HashTable，当Hash中field和value的字符串长度都小于64字节，一个Hash的field和value的个数小于512个时，使用ZipList数据结构存储

### Set(集合)：
Set存储一个无序不能重复的元素集合，最多可以存储232-1个元素

### SortSet(有序集合)：
