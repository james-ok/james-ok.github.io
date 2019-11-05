---
title: Spring原理分析
date: 2019-11-05 10:18:35
categories: Spring
tags:
- Spring
- 源码
---

提到Spring，脑海中的第一概念就是IOC（控制反转）、DI（依赖注入）、AOP（面向切面编程），但是在日常编码中，一般的同学都并没有深入了解过这几个概念的实现原理，那么今天就通过分析源码的方式来了解一下。

## Spring IOC实现思路
### 什么是IOC
将创建对象的控制权交给Spring管理就叫控制反转，说的有点抽象，其实就是让Spring为我们创建管理对象，Spring实现IOC的大致思路如下：
* 加载配置文件
* 解析配置文件
* 注册BeanDefinition
### 加载配置文件
将我们编写的xml配置文件通过类加载、文件、URL等机制加载到内存中
### 解析配置文件
将加载到内存中的配置文件解析成程序能够理解的对象，这里指的是BeanDefinition，Spring支持多种配置文件格式，例如：XML、Properties
### 注册BeanDefinition
将各个BeanDefinition对象注册到IOC容器中，Spring中核心容器是一个Map，使用beanName作为key，BeanDefinition作为value存储

## DI基本概念
### 什么是DI
从容器中获得某个对象时，当前对象依赖的所有对象一并赋值给当前对象这叫依赖注入

Spring中依赖注入是从getBean开始的，getBean方法大致作用在于将BeanDefinition配置描述对象实例化，然后将依赖的对象递归创建，然后自动注入

### 循环注入
Spring中依赖注入常见循环注入问题，那么什么是循环注入呢？例如：对象A依赖对象B，对象B依赖对象C，对象C依赖对象A，这样就陷入了死循环。那么循环注入该如何避免，Spring中是如何处理循环注入的呢。
首先，循环注入常见于两种情况
* 构造函数循环注入
* setter方法循环注入

#### 构造函数循环注入
这种方式Spring会抛出BeanCurrentlyInCreationException异常，原因是，当创建对象A的时候，发现构造函数需要依赖B，这个时候A还没有被创建完成，Spring将正在创建中的A对象放到一个正在创建的map中，然后继续创建对象B，
以此类推，当创建对象C的时候，C对象依赖对象A，这个时候又去创建A，发现map中存在对象A，这个时候Spring就会抛出BeanCurrentlyInCreationException异常。那么为什么Spring要设计一个正在创建的map，试想一下，
如果Spring不做任何创建，只是单纯的递归创建对象并且自动注入依赖对象，当C对象创建的时候，去注入对象A，这个时候A对象其实并没有完成创建，然后继续创建对象A，如此就陷入了死循环，最终对导致内存溢出。

#### setter方法循环注入
此种方式可以正常完成依赖注入，因为这种方式是先创建对象，再注入依赖对象，拿之前的例子来说，首先创建对象A，发现对象A依赖对象B，这个时候创建对象B，继而创建对象C，在创建完对象C过后，发现对象C依赖对象A，
这个时候将已经创建的对象A注入给对象C，所以，这种方式的循环注入并不会出现问题。

