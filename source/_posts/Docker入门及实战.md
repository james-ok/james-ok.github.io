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