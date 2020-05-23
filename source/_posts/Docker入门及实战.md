---
title: Docker入门及实战
date: 2019-10-11 19:46:01
categories: Docker
tags:
- Docker
---

我们常常在部署项目的过程中遇到一些莫名其妙的问题，为什么在本地就能跑的项目放到服务器上就跑不起来，
这里面的原因有很多，或许是环境的差异导致的，也或许是一些中间件版本不一致导致的，
总之，程序本身是没有问题的，这些问题或许在今天能得以解决。

## Docker是什么
Docker是一个虚拟化的容器，可以看成是一个虚拟化的服务器，但他和服务器的区别是，
容器中实际上并没有服务器操作系统，它其实是调用的宿主机的系统内核。

## Docker安装
这里使用yum安装Docker，默认yum远程仓库中没有docker，所以需要添加repository，添加repository需要用到yum-config-manager，但是在使用yum-config-manager时报找不到该命令，所以需要安装yum-utils
步骤如下：
1. 安装yum-utils
```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
```
2. 添加docker yum源
```shell
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```
3. 安装Docker CE和客户端
```shell
yum install docker-ce docker-ce-cli containerd.io
```
4. 启动Docker
```shell
systemctl start docker
```

## Docker常用命令及使用技巧

### 阿里云镜像加速
1.登录阿里云控制台，[传送门](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)
2.copy配置到服务器执行即可，类似以下
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://gqjyyepn.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 将镜像push到hub.docker.com
1.在docker客户端登录到docker hub，使用`docker login`默认使用的是hub.docker.com
2.打一个tag，这个tag必须是以docker hub的用户名开头，例如`docker tag hello-world:latest soeasydocker/hello:v1.0`，这个时候在image列表就多了一个soeasydocker/hello镜像
3.推送到hub.docker.com，使用命令`docker push soeasydocker/hello`

### 构建image的几种方式
* 容器构建镜像：docker container commit containerId imageName:tag
* Dockerfile构建镜像

### Dockerfile详解
* FROM：基础镜像，指定当前Dockerfile构建的镜像是在这个镜像之上。scratch表示没有基础镜像
* MAINTAINER：镜像维护中以及联系方式，官方推荐使用LABEL MAINTAINER来指定。
* LABEL：例如 LABEL maintainer="abc@163.com"
* RUN：构建镜像时运行的命令，一般用于构建应用程序运行所需环境，该命令会创建新的镜像层，所以使用RUN时，能将多个命令合并成一个的尽量合并成一个
* CMD：容器启动后默认指定的命令，CMD指定的命令可以被docker run后面的命令覆盖，该命令通常用于容器启动的默认环境参数。
* ENTRYPOINT：容器启动时执行的命令，用ENTRYPOINT指定的命令一定会被执行，不会被覆盖，该命令一般用于启动一个服务或引用程序。
* ADD：拷贝文件或目录到镜像中，如果拷贝的是一个url或者压缩包，会自动下载并且自动解压，例如：ADD http://aaa.com/bbb.tar.gz /html，会自动将bbb.tar.gz下载到镜像的/html中并解压
* COPY：拷贝文件或目录到镜像中，用法和ADD相同，区别在于不支持下载和解压
* ENV：设置镜像环境变量
* WORKDIR：工作目录，相当于cd到某个目录
* EXPOSE：生命容器的端口
* VOLUME：指定容器挂载点到宿主机
* ARG：构建容器时指定参数，用法：ARG arg，构建时通过build-arg指定参数名及参数值 docker build --build-arg arg=value
* USER：指定执行RUN、CMD、ENTRYPOINT命令时的用户
* HEALTHCHECK：健康检查，告诉docker容器如何检查应用程序是否还在正常运行

## Docker网络
我们知道在同一台宿主机上的多个docker容器，各个容器之间是可以互相通信的，那么为什么可以互相通信，这个就要从Docker的网络说起。
想要搞清楚Docker的网络，必须要先搞明白Linux的网络命名空间。
在Linux的网络中，各个网络的命名空间是互相隔离的，也就是互不干扰，我们可以为每个命名空间分配多个虚拟网卡（veth），veth是成对出现的，就像一根管道，管道的两端可以互相
通信，假如我们现在创建两个互不干扰的网络命名空间test1/test2，分别为两个命名空间分配虚拟网卡，由于虚拟网卡是成对出现的，我们将一对中的一个分配给test1，另一个分配给
test2，这样就想到与将两个命名空间用管道连接起来了，命令如下：
```
[root@node ~]# ip netns add test1
[root@node ~]# ip netns add test2
[root@node ~]# ip netns list
test1
test2
[root@node ~]# ip link add veth0 type veth peer name veth1
[root@node ~]# ip link list
[root@node ~]# ip link set veth0 netns test1 
[root@node ~]# ip link set veth1 netns test2

[root@node ~]# ip netns exec test1 ip link list
[root@node ~]# ip netns exec test2 ip link list
```
那么这样就能互相通信了吗，不能，因为这两个命名空间目前还没有运行并且我们并没有为他们分配ip，

## Docker-Compose安装
```
#下载docket-compose
curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

#修改权限
chmod +x /usr/local/bin/docker-compose
```


## Docker Harbor搭建
Harbor依赖docker和docker-compose，所以，在安装harbor之前必须先安装docker和compose。Harbor的安装有两种方式，在线安装和离线安装，这里使用在线安装。

### 在线安装Harbor
1.从github.com上下载harbor，这里使用1.10.2版本，[下载地址](https://github.com/goharbor/harbor/releases/download/v1.10.2/harbor-online-installer-v1.10.2.tgz)
```
wget https://github.com/goharbor/harbor/releases/download/v1.10.2/harbor-online-installer-v1.10.2.tgz
```
2.解压
```
tar -zxvf harbor-online-installer-v1.10.2.tgz
```
3.修改配置`harbor/harbor.cfg`(重点关注一下配置内容)
```
hostname = #你的域名或IP #harbor域名或IP
ui_url_protocol = http #默认使用的protocol
db_password = dbpwd #harbor数据库ROOT用户链接的密码
max_job_workers = 3
self_registration = on #允许注册用户
customize_crt = on
project_creation_restriction = adminonly #设置只有管理员可以创建项目
harbor_admin_password = harborpwd #admin用户登录密码
```
4.运行安装脚本`sh harbor/install.sh`
5.启动`docker-compose start`
### Harbor维护
Harbor本身是通过docker来运行的，日常维护依赖于docker-compose来维护，harbor的多个服务也是运行在docker中，我们可以通过`docker ps`或者`docekr-compose ps`查看
