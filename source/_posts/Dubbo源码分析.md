---
title: Dubbo源码分析
date: 2019-11-21 21:18:50
categories: Dubbo
tags:
- SPI
---

## Dubbo SPI
### Dubbo SPI使用以及规范
* 创建接口并且加上@SPI注解表示该接口是一个Dubbo扩展点，将该扩展点打成一个jar包发布
```java
@SPI
public interface IHelloService {
    String sayHello(String msg);
}
```
* 在需要实现的扩展插件项目中依赖以上接口扩展点，并且实现该接口扩展点
```java
public class HelloServiceImpl implements IHelloService {
    @Override
    public String sayHello(String msg) {
        return "Dubbo SPI Hello:" + msg;
    }
}
```
并且在resources目录先创建META-INF/dubbo/(META-INF/dubbo/;META-INF/dubbo/internal/;META-INF/services/;任选一个)目录，并且在该目录下创建以扩展点接口为名称的文件：xyz.easyjava.dubbo.spi.extend.service.IHelloService
在该文件中填写该扩展点实现的名称以及实现类的全路径，例如：
```java
hello=xyz.easyjava.dubbo.spi.achieve.service.HelloServiceImpl
hello2=xyz.easyjava.dubbo.spi.achieve.service.HelloServiceImpl2
```
* 现在就可以在需要使用该扩展的地方使用了，方式如下：
```java
public class DubboSpiTest {
    public static void main(String[] args) {
        IHelloService extension = ExtensionLoader.getExtensionLoader(IHelloService.class).getExtension("hello2");
        System.out.println(extension.sayHello("MrAToo"));
        IHelloService adaptiveExtension = ExtensionLoader.getExtensionLoader(IHelloService.class).getAdaptiveExtension();
        System.out.println(adaptiveExtension.sayHello("MrAToo2"));
    }
}
```

### 源码分析
先从`ExtensionLoader.getExtensionLoader(IHelloService.class).getAdaptiveExtension();`开始
1. 首先是调用了`ExtensionLoader`类的静态方法`getExtensionLoader(IHelloService.class)`，在该方法中除了校验，主要是实例化了一个`ExtensionLoader`实例，
并且在`ExtensionLoader`的构造方法中通过`objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());`
创建了一个`objectFactory`对象，该对象是一个`ExtensionFactory`
2. 得到`ExtensionLoader`实例对象过后，调用了该对象的`getAdaptiveExtension()`方法，在该方法中调用`createAdaptiveExtension()`创建实例，在`createAdaptiveExtension()`里面调用
`injectExtension((T) getAdaptiveExtensionClass().newInstance());`，该方法是一个注入的方法，先不看`injectExtension()`方法是如何注入的，我们先看实例是如何创建的，很显然，实例作为`injectExtension()`
方法的参数传入，那么`getAdaptiveExtensionClass().newInstance()`这句代码中的`getAdaptiveExtensionClass()`方法返回的Class<T>中T是如何确定的。在`getAdaptiveExtensionClass()`方法中，首先调用了`getExtensionClasses();`
然后判断`cachedAdaptiveClass`是否为null，如果不为null，则直接返回`cachedAdaptiveClass`，那么再看看`getExtensionClasses();`方法中又干了什么，在该方法中又调用了`loadExtensionClasses()`，下面来看看该方法的代码：
```java
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length > 1) {
                throw new IllegalStateException("more than 1 default extension name on extension " + type.getName()
                        + ": " + Arrays.toString(names));
            }
            if (names.length == 1) {
                cachedDefaultName = names[0];
            }
        }
    }

    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName());
    loadDirectory(extensionClasses, SERVICES_DIRECTORY, type.getName().replace("org.apache", "com.alibaba"));
    return extensionClasses;
}
```
首先是判断`type`的类对象是否包含@SPI注解，如果包含该注解，则将该注解的value值放到`cachedDefaultName`属性中（该属性在`createAdaptiveExtensionClassCode`方法中使用到，可以通过`getDefaultExtensionName`方法获取默认扩展点，如果自适应扩展点中URL协议为空该值可以作为默认协议），
最后调用`loadDirectory(extensionClasses, DUBBO_DIRECTORY, type.getName());`分别加载`META-INF/dubbo/;META-INF/dubbo/internal/;META-INF/services/;`三个目录下的文件，最终会调用一下方法
```java
/**
 * extensionClasses: 扩展类集合
 * resourceURL: 资源URL
 * clazz: 扩展类Class（实现了扩展接口的类，配置在接口文件中的Class）
 * name: 名称
 */
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error when load extension class(interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + "is not subtype of interface.");
    }
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException("More than 1 adaptive class found: "
                    + cachedAdaptiveClass.getClass().getName()
                    + ", " + clazz.getClass().getName());
        }
    } else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
    } else {
        clazz.getConstructor();
        if (name == null || name.length() == 0) {
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
                cachedActivates.put(names[0], activate);
            } else {
                // support com.alibaba.dubbo.common.extension.Activate
                com.alibaba.dubbo.common.extension.Activate oldActivate = clazz.getAnnotation(com.alibaba.dubbo.common.extension.Activate.class);
                if (oldActivate != null) {
                    cachedActivates.put(names[0], oldActivate);
                }
            }
            for (String n : names) {
                if (!cachedNames.containsKey(clazz)) {
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                }
            }
        }
    }
}
```
1. 先看第一个判断`clazz.isAnnotationPresent(Adaptive.class)`，如果扩展类包含`@Adaptive`注解，则将该扩展作为自定义适配扩展点，赋值给`cachedAdaptiveClass`，前面提到在`getExtensionClasses`方法中，如果`cachedAdaptiveClass`值不为null，则直接返回，
所以，当实现接口的类上有`@Adaptive`注解，则`getAdaptiveExtension();`返回的实例就是当前实例，即自定义适配扩展点。
2. 再看第二个判断`isWrapperClass(clazz)`,判断当前这个扩展是不是一个wrapper，如果是，则将该扩展类放入到`cachedWrapperClasses`中，该变量在`getExtension`方法调用链`createExtension`方法中被使用，大概源码内容为，如果`cachedWrapperClasses`变量有值，则需要包装原始对象。
3. 如果以上两个条件都不成立，则走else逻辑，在该逻辑中，首先判断当前扩展类中是否包含`@Activate`注解，如果包含，则put到`cachedActivates`中。
在判断`cachedNames`中是否包含当前扩展类的类对象，如果不存在，则将class放到`cachedNames`里面，最后循环将name作为key，class作为value放到`extensionClasses`中。

我们在回过头来看，在`getAdaptiveExtensionClass`方法中，如果`cachedAdaptiveClass`为null，则会调用`createAdaptiveExtensionClass`方法，并且将该方法的返回值放入到`cachedAdaptiveClass`中，然后返回。
在`createAdaptiveExtensionClass`方法中，通过调用`createAdaptiveExtensionClassCode`方法返回一串代码，然后动态编译生成Class对象，然后返回。
那么在`createAdaptiveExtensionClassCode`（该方法主要是生成一个代理类）中，这串代码到底是什么，又是如何生成的呢？
首先，dubbo只会为该接口中带有`@Adaptive`注解的方法进行代理，如果该接口中没有带`@Adaptive`注解的方法，则会抛出异常，并且，Dubbo是一个基于URL驱动的RPC框架，方法中标注有`@Adaptive`注解的方法参数上必须
带有`java.net.URL`参数，否则，会抛出异常。
生成的代理类代码如下：
```java
package xyz.easyjava.dubbo.spi.achieve.service;

import org.apache.dubbo.common.extension.ExtensionLoader;
public class Protocol$Adaptive implements org.apache.dubbo.rpc.Protocol {
    public void destroy() {throw new UnsupportedOperationException("method public abstract void org.apache.dubbo.rpc.Protocol.destroy() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }
    public int getDefaultPort() {throw new UnsupportedOperationException("method public abstract int org.apache.dubbo.rpc.Protocol.getDefaultPort() of interface org.apache.dubbo.rpc.Protocol is not adaptive method!");
    }
    public org.apache.dubbo.rpc.Exporter export(org.apache.dubbo.rpc.Invoker arg0) throws org.apache.dubbo.rpc.RpcException {
        if (arg0 == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument == null");
        if (arg0.getUrl() == null) throw new IllegalArgumentException("org.apache.dubbo.rpc.Invoker argument getUrl() == null");org.apache.dubbo.common.URL url = arg0.getUrl();
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.export(arg0);
    }
    public org.apache.dubbo.rpc.Invoker refer(java.lang.Class arg0, org.apache.dubbo.common.URL arg1) throws org.apache.dubbo.rpc.RpcException {
        if (arg1 == null) throw new IllegalArgumentException("url == null");
        org.apache.dubbo.common.URL url = arg1;
        String extName = ( url.getProtocol() == null ? "dubbo" : url.getProtocol() );
        if(extName == null) throw new IllegalStateException("Fail to get extension(org.apache.dubbo.rpc.Protocol) name from url(" + url.toString() + ") use keys([protocol])");
        org.apache.dubbo.rpc.Protocol extension = (org.apache.dubbo.rpc.Protocol)ExtensionLoader.getExtensionLoader(org.apache.dubbo.rpc.Protocol.class).getExtension(extName);
        return extension.refer(arg0, arg1);
    }
}
```
从上面生成的类来看，dubbo对扩展接口上带有`@Adaptive`注解的方法进行了代理，没有标注`@Adaptive`的方法，直接抛出`UnsupportedOperationException`异常。被代理的方法通过URL协议来获取一个扩展点。

接下来，分析`injectExtension()`方法，也就是Dubbo中的依赖注入，Duboo支持Spring的依赖注入以及Dubbo自己的SPI自适应扩展点。
在该方法中可以看出，当前自适应扩展点中是否包含一个setter方法，有且只有一个参数的public的方法，并且该方法没有标注`@DisableInject`注解，那么Dubbo会为该自适应扩展点依赖注入。
被注入的对象核心代码在`objectFactory.getExtension(pt, property)`通过`objectFactory`的`getExtension`方法获得被注入的对象，然后放到当前自适应扩展点，实现依赖注入。
那么`objectFactory`对象是什么，在什么时候被实例化的？
还记得在最开始通过`getExtensionLoader(IHelloService.class)`获得一个`ExtensionLoader`对象的时候，由于`ExtensionLoader`类的构造方法是私有化的，所以在`getExtensionLoader`方法中
创建了`ExtensionLoader`对象，然而就在这个私有化的构造方法中，有这样一句代码
```java
objectFactory = (type == ExtensionFactory.class ? null : ExtensionLoader.getExtensionLoader(ExtensionFactory.class).getAdaptiveExtension());
```
这就是`objectFactory`对象实例化的地方。
这里有创建了一个`ExtensionFactory`类的`ExtensionLoader`对象，并且将这个对象缓存在了`EXTENSION_LOADERS`（所以，整个运行过程中`ExtensionFactory`只有一个，并且都是`AdaptiveExtensionFactory`），然后调用了该对象的`getAdaptiveExtension`方法，该方法前面已经分析过来，
返回一个自适应扩展点。然后我们看`ExtensionFactory`的实现类，有一个名为`AdaptiveExtensionFactory`的自适应扩展点（因为该类上面标注了`@Adaptive`注解），所以我们可以发现`objectFactory`
对象的实例其实就是`AdaptiveExtensionFactory`类的实例对象。回到`injectExtension`方法的`objectFactory.getExtension(pt, property)`代码上，这里实际上调用的就是`AdaptiveExtensionFactory`
里面的`getExtension`方法。该方法如下：
```java
public <T> T getExtension(Class<T> type, String name) {
        for (ExtensionFactory factory : factories) {
            T extension = factory.getExtension(type, name);
            if (extension != null) {
                return extension;
            }
        }
        return null;
    }
```
其中`factories`就是所有的`ExtensionFactory`类的扩展，从所有的扩展点中任意返回一个null的实例返回，dubbo默认有`SpringExtensionFactory`、`SPIExtensionFactory`两个，`SpringExtensionFactory`的实现
就是从Spring的IOC容器中拿到对象注入。如果被注入对象类上标注了`@SPI`注解，那么最终还是交给`SPIExtensionFactory`对象去处理，该类里面有是通过`ExtensionLoader`得到一个自适应扩展点。到此，Dubbo的依赖注入完成。

## 服务发布流程
Dubbo是阿里巴巴依赖Spring开源的RPC框架，至于为什么要依赖Spring我们不去深究，大概是因为Spring优秀的IOC，又或者是AOP，介于Spring的高度抽象，灵活的设计模式，便于去扩展，所以，Dubbo基于Spring的扩展区实现
Dubbo基于Spring扩展的`NameSpaceHandler`，Spring容器在启动的时候会调用`DubboNamespaceHandler`的`init()`方法，该方法主要是解析Spring配置文件中的Dubbo扩展标签，将其转换成`BeanDefinition`，以便Spring容器进行管理。
Dubbo服务的发布流程是从ServiceBean开始的，因为该类实现了接口`InitializingBean`，该接口会在依赖注入完成过后调用`afterPropertiesSet`方法，而`afterPropertiesSet`方法就是Dubbo启动的关键。
