---
title: Ceph集群部署及使用
date: 2020-03-18 20:53:35
categories: Ceph
tags:
- 私有云
- 分布式文件系统
---

现阶段随着互联网的快速发展，用户的数据（例如文件、图片、视频）等越来越多越来越大，传统的文件系统已经无法满足业务需求。所以才有了今天的分布式文件系统或者说对象存储系统。
今天给大家介绍的是市面上较为流行且用的较多的CEPH，那么在使用CEPH之前首先需要将CEPH部署到服务器上，CEPH部署相对比较繁琐，导致很多同学在部署CEPH时遇到的坑很多，所以我
希望通过这篇文章能帮助大家快速安装部署CEPH集群。

# Ceph部署
## 环境准备
CentOS7(这里准备4台服务器)
    192.168.10.20/192.168.20.20(cluster ip) admin-node  管理节点
    192.168.10.21/192.168.20.21(cluster ip)	ceph-node1  MON/OSD节点
    192.168.10.22/192.168.20.22(cluster ip) ceph-node2  MON/OSD节点
    192.168.10.23/192.168.20.23(cluster ip)	ceph-node3  MON/OSD节点
    
## Ceph版本
Mimic

## 部署

### 关闭防火墙或者开放防火墙端口
所有节点关闭防火墙
* 关闭SELinux
```
[root@ceph-node1 ~]# vi /etc/selinux/config
SELINUX=disabled
```
* 关闭firewalld
```
[root@admin-node ~]# systemctl stop firewalld
[root@admin-node ~]# systemctl disable firewalld
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
[root@admin-node ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
   Active: inactive (dead)
     Docs: man:firewalld(1)

3月 28 12:29:44 localhost.localdomain systemd[1]: Starting firewalld - dynamic firewall daemon...
3月 28 12:29:45 localhost.localdomain systemd[1]: Started firewalld - dynamic firewall daemon.
3月 28 16:38:29 admin-node systemd[1]: Stopping firewalld - dynamic firewall daemon...
3月 28 16:38:30 admin-node systemd[1]: Stopped firewalld - dynamic firewall daemon.
```
重启

### 配置hosts
将所有节点的ip地址和主机名配置到所有节点的`/etc/hosts`中
```
192.168.10.20   admin-node
192.168.10.21   ceph-node1
192.168.10.22   ceph-node2
192.168.10.23   ceph-node3
```

### 所有节点安装epel-release(Extra Packages for Enterprise Linux)
```
[root@admin-node ~]# yum -y install epel-release
```

### 配置yum源
```
[root@admin-node ~]# vi /etc/yum.repos.d/ceph.repo
```
内容如下：
```
[Ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[Ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://mirrors.aliyun.com/ceph/keys/release.asc
```
最后清理缓存并重新缓存
```
[root@admin-node ~]# yum clean all && yum makecache fast
```

### 创建ceph用户并配置免密登录
ceph建议不要使用root用户，所以我们需要创建一个用户，并且要求该用户能无密码使用sudo权限，且管理节点需要无密码登录。
在所有节点上创建用户cephuser，密码为123456
```
[root@admin-node ~]# useradd cephuser
[root@admin-node ~]# echo '123456' | passwd --stdin cephuser
更改用户 cephuser 的密码 。
passwd：所有的身份验证令牌已经成功更新。
[root@admin-node ~]# echo "cephuser ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephuser
cephuser ALL = (root) NOPASSWD:ALL
[root@admin-node ~]# chmod 0440 /etc/sudoers.d/cephuser

```
在管理节点上切换到刚刚创建的用户cephuser，生成秘钥并copy到其他所有节点上。
```
[root@admin-node ~]# su cephuser
[cephuser@admin-node root]$ cd 
[cephuser@admin-node ~]$ ssh-keygen -t rsa -P ''
[cephuser@admin-node ~]$ ssh-copy-id cephuser@ceph-node1
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/cephuser/.ssh/id_rsa.pub"
The authenticity of host 'ceph-node1 (192.168.10.21)' can't be established.
ECDSA key fingerprint is SHA256:iacjrRPzH8fq4HpRtSz/UZ7Q4V9EV98MzQImvLrV/Gs.
ECDSA key fingerprint is MD5:38:b7:a4:ea:d0:e8:56:9f:30:72:b1:ee:5c:db:82:a1.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
cephuser@ceph-node1's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'cephuser@ceph-node1'"
and check to make sure that only the key(s) you wanted were added.

[cephuser@admin-node ~]$ ssh-copy-id cephuser@ceph-node2
...
[cephuser@admin-node ~]$ ssh-copy-id cephuser@ceph-node3
...
[cephuser@admin-node ~]$ ssh-copy-id cephuser@ceph-node4
...
```

### 在管理节点安装必要软件
```
[cephuser@admin-node ~]$ sudo yum install ceph-deploy python-setuptools python2-subprocess32 ceph-common 
```

### 在管理节点下创建一个集群目录，用于存放ceph集群的配置文件
```
[cephuser@admin-node ~]$ mkdir ceph-cluster
[cephuser@admin-node ~]$ cd ceph-cluster/
```

### 卸载ceph
如果安装中出现问题，可以通过一下命令卸载ceph
```
ceph-deploy purge [node]
```

### 部署MON节点
在管理节点通过ceph-deploy部署mon到ceph-node1上
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy new ceph-node1 --cluster-network 192.168.20.0/24  --public-network 192.168.10.0/24
```

### 安装ceph
一下两种方式选择一种即可
1. 在所有节点安装ceph（管理节点可以不装）
```
[cephuser@ceph-node1 ~]$ sudo yum install ceph ceph-radosgw -y
```
2. 在管理节点运行（这里可以不指定管理节点）
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy install --no-adjust-repos ceph-node1 ceph-node2 ceph-node3
```

### 在远程主机上部署MON监控节点
在管理节点上执行，生成keyring文件
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy mon create-initial
```

### 将配置和client.admin秘钥环推送到远程主机
每次更改ceph的配置文件，都可以用这个命令推送到所有节点上
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy admin ceph-node1 ceph-node2 ceph-node3
```

### 在所有节点以root的身份运行
不管哪个节点，只要需要使用cephadm用户执行命令行工具，这个文件就必须要让cephadm用户拥有访问权限，就必须执行这一步
```
[root@ceph-node1 ~]# setfacl -m u:cephuser:r /etc/ceph/ceph.client.admin.keyring
```

### 部署MGR
在管理节点的集群目录下执行
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy mgr create ceph-node1
```

### 创建OSD
查看所有OSD节点上的磁盘
```
[root@ceph-node1 ~]# fdisk -l
磁盘 /dev/sda：10.7 GB, 10737418240 字节，20971520 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节
磁盘标签类型：dos
磁盘标识符：0x000b0681

   设备 Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048     2099199     1048576   83  Linux
/dev/sda2         2099200    20971519     9436160   8e  Linux LVM

磁盘 /dev/sdb：5368 MB, 5368709120 字节，10485760 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节


磁盘 /dev/mapper/centos-root：8585 MB, 8585740288 字节，16769024 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节


磁盘 /dev/mapper/centos-swap：1073 MB, 1073741824 字节，2097152 个扇区
Units = 扇区 of 1 * 512 = 512 bytes
扇区大小(逻辑/物理)：512 字节 / 512 字节
I/O 大小(最小/最佳)：512 字节 / 512 字节

```
清空osd节点上用来作为osd设备的磁盘
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy disk zap ceph-node1 /dev/sdb
[cephuser@admin-node ceph-cluster]$ ceph-deploy disk zap ceph-node2 /dev/sdb
[cephuser@admin-node ceph-cluster]$ ceph-deploy disk zap ceph-node3 /dev/sdb

```
创建OSD
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy osd create ceph-node1 --data /dev/sdb
[cephuser@admin-node ceph-cluster]$ ceph-deploy osd create ceph-node2 --data /dev/sdb
[cephuser@admin-node ceph-cluster]$ ceph-deploy osd create ceph-node3 --data /dev/sdb
```

## 部署对象网关

### 配置和s3cmd客户端的使用
ceph-radowgw在前面已经默认安装了，所以直接使用`ceph-deploy rgw create ceph-node1`创建一个网关节点，这个时候就可以在浏览器[访问](http://192.168.10.21:7480/)了
创建一个用户，在网关节点上执行
```
[root@ceph-node1 ~]# radosgw-admin user create --uid=radosgw --display-name="radosgw"
{
    "user_id": "radosgw",
    "display_name": "radosgw",
    "email": "",
    "suspended": 0,
    "max_buckets": 1000,
    "auid": 0,
    "subusers": [],
    "keys": [
        {
            "user": "radosgw",
            "access_key": "FWAL5HI8BT298JIRK730",
            "secret_key": "E1JZCCyi40IUXRnBWpYp4N0a3eflxsD7r6OWBBwL"
        }
    ],
    "swift_keys": [],
    "caps": [],
    "op_mask": "read, write, delete",
    "default_placement": "",
    "placement_tags": [],
    "bucket_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "user_quota": {
        "enabled": false,
        "check_on_raw": false,
        "max_size": -1,
        "max_size_kb": 0,
        "max_objects": -1
    },
    "temp_url_keys": [],
    "type": "rgw",
    "mfa_ids": []
}

```
我们在管理节点安装一个s3cmd用于测试
```
[root@admin-node ~]# yum install -y s3cmd
```
配置s3cmd客户端
```
[root@admin-node ~]# s3cmd --configure

Enter new values or accept defaults in brackets with Enter.
Refer to user manual for detailed description of all options.

Access key and Secret key are your identifiers for Amazon S3. Leave them empty for using the env variables.
Access Key: FWAL5HI8BT298JIRK730
Secret Key: E1JZCCyi40IUXRnBWpYp4N0a3eflxsD7r6OWBBwL
Default Region [US]: US

Use "s3.amazonaws.com" for S3 Endpoint and not modify it to the target Amazon S3.
S3 Endpoint [s3.amazonaws.com]: 

Use "%(bucket)s.s3.amazonaws.com" to the target Amazon S3. "%(bucket)s" and "%(location)s" vars can be used
if the target S3 system supports dns based buckets.
DNS-style bucket+hostname:port template for accessing a bucket [%(bucket)s.s3.amazonaws.com]: 

Encryption password is used to protect your files from reading
by unauthorized persons while in transfer to S3
Encryption password: 
Path to GPG program [/usr/bin/gpg]: 

When using secure HTTPS protocol all communication with Amazon S3
servers is protected from 3rd party eavesdropping. This method is
slower than plain HTTP, and can only be proxied with Python 2.7 or newer
Use HTTPS protocol [Yes]: 

On some networks all internet access must go through a HTTP proxy.
Try setting it here if you can't connect to S3 directly
HTTP Proxy server name: 

New settings:
  Access Key: FWAL5HI8BT298JIRK730
  Secret Key: E1JZCCyi40IUXRnBWpYp4N0a3eflxsD7r6OWBBwL
  Default Region: US
  S3 Endpoint: s3.amazonaws.com
  DNS-style bucket+hostname:port template for accessing a bucket: %(bucket)s.s3.amazonaws.com
  Encryption password: 
  Path to GPG program: /usr/bin/gpg
  Use HTTPS protocol: True
  HTTP Proxy server name: 
  HTTP Proxy server port: 0

Test access with supplied credentials? [Y/n] n

Save settings? [y/N] y
Configuration saved to '/root/.s3cfg'

```
配置的不完全，需要手动编辑配置文件，主要有如下配置，其他的配置可以先不管
```
[root@admin-node ~]# vi .s3cfg
[default]
access_key = FWAL5HI8BT298JIRK730
secret_key = E1JZCCyi40IUXRnBWpYp4N0a3eflxsD7r6OWBBwL
host_base = 192.168.10.21:7480
host_bucket = 192.168.10.21:7480/%(bucket)
use_https = False
```
基本操作
```
[root@admin-node ~]# s3cmd mb s3://my-bucket-name
Bucket 's3://my-bucket-name/' created
[root@admin-node ~]# s3cmd ls
2020-03-29 08:31  s3://my-bucket-name
[root@admin-node ~]# s3cmd rb s3://my-bucket-name
Bucket 's3://my-bucket-name/' removed

```

### Java S3客户端的使用
添加maven依赖
```xml
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-s3</artifactId>
    <version>1.11.534</version>
</dependency>
```
Java代码
```java
package xyz.easyjava;

import com.amazonaws.ClientConfiguration;
import com.amazonaws.Protocol;
import com.amazonaws.auth.AWSCredentials;
import com.amazonaws.auth.AWSStaticCredentialsProvider;
import com.amazonaws.auth.BasicAWSCredentials;
import com.amazonaws.client.builder.AwsClientBuilder;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3Client;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;
import com.amazonaws.services.s3.model.*;
import com.amazonaws.util.StringUtils;

import java.io.File;
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.net.URL;
import java.util.Date;
import java.util.List;

public class CephS3Demo {

    public static void main(String[] args) throws FileNotFoundException {
        String accessKey = "FWAL5HI8BT298JIRK730";
        String secretKey = "E1JZCCyi40IUXRnBWpYp4N0a3eflxsD7r6OWBBwL";

        AWSCredentials credentials = new BasicAWSCredentials(accessKey, secretKey);

        ClientConfiguration clientConfiguration = new ClientConfiguration();
        clientConfiguration.setProtocol(Protocol.HTTP);
        AmazonS3 amazonS3 = AmazonS3ClientBuilder.standard()
                .withCredentials(new AWSStaticCredentialsProvider(credentials))
                .withEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration("192.168.10.21:7480",""))
                .withClientConfiguration(clientConfiguration)
                .build();

        //Bucket b = amazonS3.createBucket("two-bucket");
        //System.out.println(b);

        List<Bucket> buckets = amazonS3.listBuckets();
        for (Bucket bucket : buckets) {
            System.out.println(bucket.getName() + "\t" +
                    StringUtils.fromDate(bucket.getCreationDate()));
        }

        //amazonS3.deleteBucket("two-bucket");

        /*FileInputStream inputStream = new FileInputStream(new File("C:\\Users\\vipdu\\Desktop\\新建文本文档.txt"));
        PutObjectResult putObject = amazonS3.putObject("two-bucket", "newFile.txt", new File("C:\\Users\\vipdu\\Desktop\\新建文本文档.txt"));
        System.out.println(putObject);

        PutObjectResult putObject1 = amazonS3.putObject("two-bucket", "newFile2.txt", inputStream, new ObjectMetadata());
        System.out.println(putObject1);*/

        //将"newFile.txt"设置为公开读，"newFile2.txt"设置为私有
        amazonS3.setObjectAcl("two-bucket","newFile.txt", CannedAccessControlList.PublicRead);
        amazonS3.setObjectAcl("two-bucket","newFile2.txt", CannedAccessControlList.Private);

        GeneratePresignedUrlRequest request = new GeneratePresignedUrlRequest("two-bucket","newFile.txt");
        request.setExpiration(new Date(System.currentTimeMillis()+(60*1000)));
        URL url = amazonS3.generatePresignedUrl(request);
        System.out.println(url);

        GeneratePresignedUrlRequest request2 = new GeneratePresignedUrlRequest("two-bucket","newFile2.txt");
        URL url2 = amazonS3.generatePresignedUrl(request2);
        System.out.println(url2);


    }
}

```

## 扩展集群

### 扩展MON节点
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy mon add ceph-node2
[cephuser@admin-node ceph-cluster]$ ceph-deploy mon add ceph-node3

```
### 扩展MGR
```
[cephuser@admin-node ceph-cluster]$ ceph-deploy mgr create ceph-node2 ceph-node3
```