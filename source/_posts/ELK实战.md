---
title: ELK实战
date: 2020-01-09 19:39:35
categories: ELK
tags:
- ElasticSearch
- Kibana
- LogStash
---

## ElasticSearch安装
* [下载](https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.5.1-linux-x86_64.tar.gz) 
* 解压：`tar -zxvf elasticsearch-7.5.1-linux-x86_64.tar.gz`
* 进入ElasticSearch目录：`cd elasticsearch-7.5.1`
* 启动：`sh bin/elasticsearch`

### 启动遇到的问题
#### 问题一
由于ElasticSearch处于安全性考虑，ElasticSearch禁止使用root用户启动，需要新建一个用户和组，并且将ElasticSearch交给该用户和组管理。
1. 创建组：`groupadd elk`
2. 创建用户并分配组：`useradd -g elk elk`
3. 将ElasticSearch分配给新建的用户：`chown -R elk:elk ./elasticsearch-7.5.1`
4. 切换用户：`su elk`
5. 启动：`sh bin/elasticsearch`

#### 问题二
```shell
ERROR: [3] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65535]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[3]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```
切换回root用户
1. 编辑`vi /etc/security/limits.conf`文件，在倒数第二行添加
```shell
*       soft    nofile  65536
*       hard    nofile  65536
```
2. 编辑`vi /etc/sysctl.conf`文件，添加`vm.max_map_count=655360`
3. 执行`sysctl -p`

#### 问题四
```shell
ERROR: [1] bootstrap checks failed
[1]: the default discovery settings are unsuitable for production use; at least one of [discovery.seed_hosts, discovery.seed_providers, cluster.initial_master_nodes] must be configured
```
1. 修改配置文件`vim config/elasticsearch.yml`，添加内容`cluster.initial_master_nodes: ["node-1"]`

## Kibana安装
* [下载](https://artifacts.elastic.co/downloads/kibana/kibana-7.5.1-linux-x86_64.tar.gz) 
* 解压：`tar -zxvf kibana-7.5.1-linux-x86_64.tar.gz`
* 将Kibana分配给新建的用户：`chown -R elk:elk ./kibana-7.5.1-linux-x86_64`
* 修改配置如下
```yaml
server.port: 5601
server.host: "192.168.1.6"
elasticsearch.hosts: ["http://192.168.1.6:9200"]
i18n.locale: "zh-CN"
```

## LogStash安装
* [下载](https://artifacts.elastic.co/downloads/logstash/logstash-7.5.1.tar.gz) 
* 解压：`tar -zxvf LogStash-7.5.1.tar.gz`
* 将LogStash分配给新建的用户：`chown -R elk:elk ./LogStash-7.5.1`
* 创建LogStash配置文件
* 启动LogStash：`sh sh/logstash -f config/logstash.conf`

### LogStash配置
LogStash可以简单分为三个部分，分别是input、filter、output。input用于输入数据来源，filter用于处理数据，output用于数据数据，一般将output输出到ES中。