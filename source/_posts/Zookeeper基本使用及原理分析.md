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
* tickTime:
* initTime:
## Zookeeper使用
