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
这种方式Spring会抛出`BeanCurrentlyInCreationException`异常，原因是，当创建对象A的时候，发现构造函数需要依赖B，这个时候A还没有被创建完成，Spring将正在创建中的A对象放到一个正在创建的`prototypesCurrentlyInCreation`(`prototypesCurrentlyInCreation`是一个`NamedThreadLocal`)中，然后继续创建对象B，
以此类推，当创建对象C的时候，C对象依赖对象A，这个时候又去创建A，发现`prototypesCurrentlyInCreation`中存在对象A，这个时候Spring就会抛出`BeanCurrentlyInCreationException`异常。那么为什么Spring要设计一个正在创建的`prototypesCurrentlyInCreation`，试想一下，
如果Spring不做任何措施，只是单纯的递归创建对象并且自动注入依赖对象，当C对象创建的时候，去注入对象A，这个时候A对象其实并没有完成创建，然后继续创建对象A，如此就陷入了死循环，最终对导致内存溢出。对象在创建完成后从`prototypesCurrentlyInCreation`中删除。

#### setter方法循环注入
此种方式可以正常完成依赖注入，因为这种方式是先创建对象，再注入依赖对象，拿之前的例子来说，首先创建对象A，发现对象A依赖对象B，这个时候创建对象B，继而创建对象C，在创建完对象C过后，发现对象C依赖对象A，
这个时候将已经创建的对象A注入给对象C，所以，这种方式的循环注入并不会出现问题。

## SpringMVC实现原理
SpringMVC的实现原理可以分为三个阶段，分别是：
* 配置阶段
* 初始化阶段
* 请求执行阶段

### 配置阶段
该阶段主要是配置SpringMVC的核心Servlet以及指定Spring的配置文件位置，配置监听等。

### 初始化阶段
初始化SpringMVC九大组件
```java
this.initMultipartResolver(context);
this.initLocaleResolver(context);
this.initThemeResolver(context);
this.initHandlerMappings(context);
this.initHandlerAdapters(context);
this.initHandlerExceptionResolvers(context);
this.initRequestToViewNameTranslator(context);
this.initViewResolvers(context);
this.initFlashMapManager(context);
```
以上组件的作用分别是
`MultipartResolver`：文件处理器
`LocaleResolver`：语言处理器，用于国际化
`ThemeResolver`：主题处理器
`HandlerMappings`：SpringMVC核心请求处理器，简称处理器映射器
`HandlerAdapters`：处理器适配器，用于适配处理器参数，从request中获取参数，自动转型并适配形式参数
`HandlerExceptionResolvers`：异常处理器
`RequestToViewNameTranslator`：视图名称翻译器
`ViewResolvers`：页面渲染处理器
`FlashMapManager`：参数传递管理器

以上最核心的是`HandlerMappings`、`HandlerAdapters`、`ViewResolvers`，`HandlerMapping`用于管理URI对应的处理器Method，`HandlerAdapters`用于适配处理器参数，`ViewResolvers`用于试图渲染

### 请求执行阶段
当SpringMVC应用启动完毕，用户像服务器发送一个请求，首先会通过用户请求的URI进行匹配`HandlerMapping`，得到`HandlerMapping`过后，进行参数封装，最后执行方法，最后通过返回的`ModelAndView`交给`ViewResolvers`渲染试图

### 小结
SpringMVC的运行流程大致是，配置核心入口DispatcherServlet，初始化`HandlerMappings`、`HandlerAdapters`、`ViewResolvers`，当用户请求时，通过用户请求的URI查找处理器，封装请求参数，执行返回结果，渲染试图。

## Spring AOP实现原理
Spring中AOP分为两个阶段，第一个阶段为加载和解析配置阶段，第二个阶段为创建代理对象阶段
### 加载解析配置阶段
该阶段主要对AOP的配置进行提起加载解析，在IOC的解析成`BeanDefinition`的时候进行，其实Spring所有的配置加载都在这一阶段完成，在解析XML配置文件的时候，Spring默认只解析Bean Namespace，当Spring发现配置文件中有引入其他Spring扩展Namespace的时候，
Spring会根据配置文件的NamespaceURI进行确定使用哪一个NamespaceHandler来解析当前配置文件中的扩展配置，最后都将封装成一个`BeanDefinition`然后注册到IOC容器中。值得注意的是，在解析AOP配置的过程中，
Spring向容器注册了一个`AspectJAwareAdvisorAutoProxyCreator`，改类用于创建代理对象。
### 创建代理对象阶段
Spring中不管是创建对象还是依赖注入都是从getBean开始的，通过探究里面真正干活的是`AbstractAutowireCapableBeanFactory`中的`createBean`，该方法最终会调到`initializeBean`，该方法又会调用`applyBeanPostProcessorsAfterInitialization`方法，
在该方法中，会循环调用之前初始化时注册到容器中的所有`BeanPostProcessor`的`postProcessAfterInitialization`方法，而AOP在初始化的时候，注册了一个`AspectJAwareAdvisorAutoProxyCreator`，该类是`BeanPostProcessor`的子类，所以最终创建代理类的是
`AspectJAwareAdvisorAutoProxyCreator`的父类AbstractAutoProxyCreator的`postProcessAfterInitialization`方法，在该方法中调用`wrapIfNecessary`判断是否需要被代理（里面的判断逻辑就是切入点表达式，满足条件的表示该方法被代理），
并且应该使用哪种代理方式取决于目标对象是代理接口还是代理类，如果是代理接口则使用JDK动态代理，代理类则使用CGLIB动态代理。代码如下：
```java
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {
	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			return new ObjenesisCglibAopProxy(config);
		}
		else {
			return new JdkDynamicAopProxy(config);
		}
	}
}
```
