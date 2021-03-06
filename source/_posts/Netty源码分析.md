---
title: Netty源码分析
date: 2020-03-25 12:08:54
categories: Netty
tags:
- NIO
- 网络编程
---

## Netty中Reactor模式的三种实现方式
* Reactor单线程模型：一个线程处理客户端连接并且处理客户端请求，在netty中表现为，使用一个NioEventLoopGroup并且只有一个线程
* Reactor多线程模型：多个线程并行处理客户端连接和请求，使用一个NioEventLoopGroup，该线程池中有多个线程
* 主从Reactor多线程模型：将客户端连接和客户端请求分为两组，用两组线程池去分别处理连接和请求，使用两个NioEventLoopGroup，并且每个NioEventLoopGroup中有多个线程

## TCP粘包半包及解决方案
* 问题
    半包现象：应用程序写入的数据大于套接字缓冲区，可能会出现半包现象
    粘包现象：应用程序写入的数据小于套接字缓冲区，可能会出现粘包现象
* 解决方案
    固定长度接收：基于固定长度消息进行处理
    分隔符：基于消息边界进行处理
    固定消息头长度，在消息头中指定正文长度：基于固定长度的消息头来处理消息，然后通过消息头中的指定内容长度来进行处理

## Netty二次解码
在解决了TCP数据的粘包和半包问题过后，这个时候得到的数据任然是二进制数据，那么如何将二进制数据装换成应用程序能使用的数据（例如java对象）这就需要使用二次解码，Netty支持的二次
解码器或者说序列化协议非常多，例如：protobuf、marshalling等等。