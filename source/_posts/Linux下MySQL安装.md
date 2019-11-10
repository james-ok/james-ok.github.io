---
title: Linux下MySQL安装
date: 2019-11-10 17:49:20
categories: MySQL
tags:
- Linux
- MySQL
---
## 准备
Linux CentOS 7 x64
mysql-5.7.27 [下载](http://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.16-1.el7.x86_64.rpm-bundle.tar)
## 卸载MariaDB
```shell
[root@MiWiFi-R3L-srv downloads]# rpm -qa|grep mariadb
mariadb-libs-5.5.52-1.el7.x86_64
[root@MiWiFi-R3L-srv downloads]# rpm -e mariadb-libs-5.5.52-1.el7.x86_64 --nodeps
```
## 安装MySQL
解压MySQL
```shell
[root@MiWiFi-R3L-srv developer]# tar -xvf ../downloads/mysql-5.7.27-1.el7.x86_64.rpm-bundle.tar -C /env/developer/mysql
mysql-community-libs-5.7.27-1.el7.x86_64.rpm
mysql-community-embedded-devel-5.7.27-1.el7.x86_64.rpm
mysql-community-libs-compat-5.7.27-1.el7.x86_64.rpm
mysql-community-devel-5.7.27-1.el7.x86_64.rpm
mysql-community-embedded-compat-5.7.27-1.el7.x86_64.rpm
mysql-community-common-5.7.27-1.el7.x86_64.rpm
mysql-community-client-5.7.27-1.el7.x86_64.rpm
mysql-community-server-5.7.27-1.el7.x86_64.rpm
mysql-community-test-5.7.27-1.el7.x86_64.rpm
mysql-community-embedded-5.7.27-1.el7.x86_64.rpm
```
安装顺序一定是`common`、`libs`、`client`、`server`，接下来，开始吧。
```shell
[root@MiWiFi-R3L-srv mysql]# rpm -ivh mysql-community-common-5.7.27-1.el7.x86_64.rpm
警告：mysql-community-common-5.7.27-1.el7.x86_64.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
准备中...                          ################################# [100%]
正在升级/安装...
   1:mysql-community-common-5.7.27-1.e################################# [100%]
   
[root@MiWiFi-R3L-srv mysql]# rpm -ivh mysql-community-libs-5.7.27-1.el7.x86_64.rpm
警告：mysql-community-libs-5.7.27-1.el7.x86_64.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
准备中...                          ################################# [100%]
正在升级/安装...
   1:mysql-community-libs-5.7.27-1.el7################################# [100%]

[root@MiWiFi-R3L-srv mysql]# rpm -ivh mysql-community-client-5.7.27-1.el7.x86_64.rpm
警告：mysql-community-client-5.7.27-1.el7.x86_64.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
准备中...                          ################################# [100%]
正在升级/安装...
   1:mysql-community-client-5.7.27-1.e################################# [100%]
```
在安装server的时候，报错如下：
```shell
[root@MiWiFi-R3L-srv mysql]# rpm -ivh mysql-community-server-5.7.27-1.el7.x86_64.rpm
警告：mysql-community-server-5.7.27-1.el7.x86_64.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
错误：依赖检测失败：
	/usr/bin/perl 被 mysql-community-server-5.7.27-1.el7.x86_64 需要
	net-tools 被 mysql-community-server-5.7.27-1.el7.x86_64 需要
	perl(Getopt::Long) 被 mysql-community-server-5.7.27-1.el7.x86_64 需要
	perl(strict) 被 mysql-community-server-5.7.27-1.el7.x86_64 需要
```
可见server安装需要依赖perl、net-tools，我们使用`yum -y install perl`、`yum -y install net-tools`安装完成后，我们再执行
```shell
[root@MiWiFi-R3L-srv mysql]# rpm -ivh mysql-community-server-5.7.27-1.el7.x86_64.rpm
警告：mysql-community-server-5.7.27-1.el7.x86_64.rpm: 头V3 DSA/SHA1 Signature, 密钥 ID 5072e1f5: NOKEY
准备中...                          ################################# [100%]
正在升级/安装...
   1:mysql-community-server-5.7.27-1.e################################# [100%]
```
到此，MySQL已经安装完成
## 配置MySQL
执行`mysqld --initialize --user=mysql`，其中`--initialize`参数表示以安全模式初始化，则会在日志文件中生成一个密码，用户在登录数据库过后需要将其修改为自己的密码，相反
`--initialize-insecure`表示非安全模式，则不会生成临时密码。`--user=mysql`是为了保证数据库目录为与文件的所有者为mysql登陆用户，如果是root身份运行mysql服务，则需要，如果是以mysql身份运行，则可以去掉--user选项。
```shell
[root@MiWiFi-R3L-srv /]# cat /var/log/mysqld.log 
2019-11-10T11:08:27.878852Z 0 [Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use --explicit_defaults_for_timestamp server option (see documentation for more details).
2019-11-10T11:08:29.654413Z 0 [Warning] InnoDB: New log files created, LSN=45790
2019-11-10T11:08:29.914439Z 0 [Warning] InnoDB: Creating foreign key constraint system tables.
2019-11-10T11:08:30.008176Z 0 [Warning] No existing UUID has been found, so we assume that this is the first time that this server has been started. Generating a new UUID: 6d3a480f-03aa-11ea-a2ef-00e0b41ce34d.
2019-11-10T11:08:30.012039Z 0 [Warning] Gtid table is not ready to be used. Table 'mysql.gtid_executed' cannot be opened.
2019-11-10T11:08:30.014895Z 1 [Note] A temporary password is generated for root@localhost: eFZcxogD#6.i
```
## 启动MySQL
现在启动mysql数据库`systemctl start mysqld.service`，也可以将MySQL设置为开机自动启动`systemctl enable mysqld.service`,然后连接到MySQL修改密码`mysql -uroot -p`
这里必须修改密码，否则将无法使用
```shell
mysql> show database;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'database' at line 1
```
更改密码
```shell
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'cjwan1314+';
Query OK, 0 rows affected (0.00 sec)
```
## 开启防火墙
查询端口是否开放`firewall-cmd --query-port=3306/tcp`
开放端口`firewall-cmd --zone=public --add-port=3306/tcp --permanent`
重启防火墙`systemctl restart firewalld.service`

MySQL安装到此完毕！！