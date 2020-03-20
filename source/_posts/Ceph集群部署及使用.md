---
title: Ceph集群部署及使用
date: 2020-03-18 20:53:35
categories: Ceph
tags:
- 私有云
- 分布式文件系统
---

# Ceph部署
## 环境准备
CentOS7(这里准备四台服务器)
    192.168.1.11    node1
    192.168.1.12	node2
    192.168.1.13	node3
    192.168.1.14	node4
1. 将所有节点的hosts文件添加如上信息
```shell
[root@node1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.1.11	node1
192.168.1.12	node2
192.168.1.13	node3
192.168.1.14	node4
```
2. 所有节点关闭防火墙或者添加防火墙规则（线上务必使用添加规则的方式）
```shell
[root@node1 ~]# systemctl stop firewalld.service
[root@node1 ~]# systemctl disable firewalld.service
Removed symlink /etc/systemd/system/multi-user.target.wants/firewalld.service.
Removed symlink /etc/systemd/system/dbus-org.fedoraproject.FirewallD1.service.
```
3. 所有节点关闭selinux（将SELINUX的值改为disabled，需要重启生效）
```shell
[root@node1 ~]# vim /etc/selinux/config 
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
4. 所有节点安装NTP服务并将时间同步
```shell
[root@node1 ~]# yum install ntp ntpdate ntp-doc
[root@node1 ~]# systemctl start ntpd.service
[root@node1 ~]# ntpdate ntp1.aliyun.com
18 Mar 21:16:07 ntpdate[11092]: the NTP socket is in use, exiting
```
5. 各个节点创建普通用户，且无需密码使用sudo权限，切换到cephuser用户然后生成秘钥
```shell
[root@node1 ~]# useradd -d /home/cephuser -m cephuser
[root@node1 ~]# passwd cephuser //这里密码是：userpwd
[root@node1 ~]# echo "cephuser ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/cephuser
cephuser ALL = (root) NOPASSWD:ALL
[root@node1 ~]# chmod 0440 /etc/sudoers.d/cephuser
```
在管理节点上生成ssh秘钥，并且将公钥复制给其他节点
```shell
[root@node1 ~]# su cephuser
[cephuser@node1 root]$ ssh-keygen
[cephuser@node1 .ssh]$ ssh-copy-id cephuser@node1
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/cephuser/.ssh/id_rsa.pub"
The authenticity of host 'node1 (192.168.1.11)' can't be established.
ECDSA key fingerprint is SHA256:be2NmeaJDTwI5Jgz7c8mWuj3FZztfpgicKZU8tdY2XI.
ECDSA key fingerprint is MD5:18:ff:4c:83:6b:89:3c:8e:7b:8c:52:dd:d5:ca:de:42.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
cephuser@node1's password: 

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'cephuser@node1'"
and check to make sure that only the key(s) you wanted were added.
[cephuser@node1 .ssh]$ ssh-copy-id cephuser@node2
...
[cephuser@node1 .ssh]$ ssh-copy-id cephuser@node3
...
[cephuser@node1 .ssh]$ ssh-copy-id cephuser@node4
...
```
将所有节点的用户名和hostname都配置到管理节点的.ssh/config文件中，并且赋予该config文件权限600，这样就不用每次ceph-deploy都要使用--username指定用户名了，简化了ssh和scp的用法
```shell
cat > ~/.ssh/config <<EOF
Host node1
  Hostname node1
  User cephuser
Host node2
  Hostname node2
  User cephuser
Host node3
  Hostname node3
  User cephuser
Host node4
  Hostname node4
  User cephuser
EOF
[cephuser@node1 .ssh]$ chmod 600 ~/.ssh/config
```
6. 配置aliyun yum源
```shell
[cephuser@node1 ~]$ sudo yum install epel-release -y
[cephuser@node1 ~]$ vim /etc/yum.repos.d/ceph.repo

[ceph]
name=ceph
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/x86_64/
enabled=1
gpgcheck=0
priority=1

[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/noarch/
enabled=1
gpgcheck=0
priority=1

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-mimic/el7/SRPMS
enabled=1
gpgcheck=0
priority=1

```
7. 在node1上面安装ceph-deploy
```shell
[cephuser@node1 ~]$ sudo yum install ceph-deploy -y
```
8. 在node1上创建集群
1). 建立一个集群配置目录，并且cd到该目录下
```shell
[cephuser@node1 ~]$ sudo mkdir /etc/ceph
[cephuser@node1 ~]$ cd /etc/ceph
```
2). 将Monitor安装在node2节点上
```shell
[cephuser@node1 ceph]$ sudo ceph-deploy new node2
[cephuser@node1 ceph]$ ls
ceph.conf  ceph-deploy-ceph.log  ceph.mon.keyring
[cephuser@node1 ceph]$ sudo vim ceph.conf 
```
修改当前目录下的ceph.conf文件，加入`osd pool default size = 2`，该配置旨在把Ceph 配置文件里的默认副本数从 3 改成 2
注意：如果这里报错`ImportError: No module named pkg_resources`，是因为Python-pip被破坏了，可以执行以下命令修复：
```shell
[cephuser@node1 ceph]$ sudo yum install python-pkg-resources python-setuptools -y
```
9. 在各节点安装ceph
```shell
[cephuser@node1 ceph]$ sudo ceph-deploy install node1 node2 node3 node4
```
10. 配置初始 monitor(s)、并收集所有密钥：
```shell
[cephuser@node1 ceph]$ sudo ceph-deploy mon create-initial
[cephuser@node1 ceph]$ ll
总用量 544
-rw------- 1 root root    113 3月  19 21:50 ceph.bootstrap-mds.keyring
-rw------- 1 root root    113 3月  19 21:50 ceph.bootstrap-mgr.keyring
-rw------- 1 root root    113 3月  19 21:50 ceph.bootstrap-osd.keyring
-rw------- 1 root root    113 3月  19 21:50 ceph.bootstrap-rgw.keyring
-rw------- 1 root root    151 3月  19 21:50 ceph.client.admin.keyring
-rw-r--r-- 1 root root    220 3月  19 21:32 ceph.conf
-rw-r--r-- 1 root root 276082 3月  19 21:50 ceph-deploy-ceph.log
-rw------- 1 root root     73 3月  19 21:28 ceph.mon.keyring
-rw-r--r-- 1 root root     92 12月 13 06:01 rbdmap
```
11. 添加两个OSD
```shell
[cephuser@node3 ~]$ ssh node3
[cephuser@node3 ~]$ sudo mkdir /var/local/osd0
[cephuser@node3 ~]$ exit

[cephuser@node3 ~]$ ssh node4
[cephuser@node3 ~]$ sudo mkdir /var/local/osd1
[cephuser@node3 ~]$ exit
```
12. 从管理节点执行ceph-deploy来准备OSD，这里官网给的命令是`ceph-deploy osd prepare {ceph-node}:/path/to/directory`和`ceph-deploy osd activate {ceph-node}:/path/to/directory`
但其实该命令已经不再支持
```shell

```
注意，[这里使用prepare命令会报错](https://blog.csdn.net/redenval/article/details/79581686)
1). 新增磁盘
```shell
[cephuser@node1 ceph]$ sudo ceph-deploy osd create --data /dev/sdb node3

```
在node3和node4上安装mkfs.xfs文件，通过命令`sudo yum install xfs* -y`
注意，报错`[node3][ERROR ] RuntimeError: command returned non-zero exit status: 1`，解决办法，在各节点给磁盘添加权限`sudo chmod 777 /dev/sdb`