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

## Docker-Compose安装
```
#下载docket-compose
curl -L https://github.com/docker/compose/releases/download/1.17.1/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose

#修改权限
chmod +x /usr/local/bin/docker-compose
```