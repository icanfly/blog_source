title: Mybatis Invalid bound statement (not found)问题分析
date: 2016-03-04

tags:
 - Mybatis
 - ORM
categories:
 - 原创文章
thumbnail: /images/mybatis.png
---

今天又因为精心大意犯一个错，而且以前也已经遇到过，但是没有进行总结
```java
Caused by: org.apache.ibatis.binding.BindingException: Invalid bound statement (not found): com.xxx.xxx.xxx.monitor.mapper.XXXXMapper.loadAllServices
    at org.apache.ibatis.binding.MapperMethod$SqlCommand.<init>(MapperMethod.java:189) ~[mybatis-3.2.7.jar:3.2.7]
    at org.apache.ibatis.binding.MapperMethod.<init>(MapperMethod.java:43) ~[mybatis-3.2.7.jar:3.2.7]
    at org.apache.ibatis.binding.MapperProxy.cachedMapperMethod(MapperProxy.java:58) ~[mybatis-3.2.7.jar:3.2.7]
    at org.apache.ibatis.binding.MapperProxy.invoke(MapperProxy.java:51) ~[mybatis-3.2.7.jar:3.2.7]
    at com.sun.proxy.$Proxy35.loadAllServices(Unknown Source) ~[na:na]
    ...
    ...
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.7.0_79]
    at java.lang.reflect.Method.invoke(Method.java:606) ~[na:1.7.0_79]
    at org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor$LifecycleElement.invoke(InitDestroyAnnotationBeanPostProcessor.java:354) ~[spring-beans-4.2.4.RELEASE.jar:4.2.4.RELEASE]
    at org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor$LifecycleMetadata.invokeInitMethods(InitDestroyAnnotationBeanPostProcessor.java:305) ~[spring-beans-4.2.4.RELEASE.jar:4.2.4.RELEASE]
    at org.springframework.beans.factory.annotation.InitDestroyAnnotationBeanPostProcessor.postProcessBeforeInitialization(InitDestroyAnnotationBeanPostProcessor.java:133) ~[spring-beans-4.2.4.RELEASE.jar:4.2.4.RELEASE]
    ... 74 common frames omitted
```

查了好多网上资料，其实都没有说到我这个问题的根本上。

后来分析了好一大阵后才发现是maven编译时的配置出问题，加上下面这个配置就好了。

<!--more-->

```xml
<build>
...
<resources>
    <resource>
        <directory>src/main/resources</directory>
        <excludes>
            <exclude>**/.svn/*</exclude>
        </excludes>
    </resource>
    <resource>
        <directory>src/main/java</directory>
        <excludes>
            <exclude>**/.svn/*</exclude>
        </excludes>
        <includes>
            <include>**/*.xml</include>
        </includes>
    </resource>
</resources>
        ...
</build>
```

原因在于如果你的资源文件在java包下面，则maven默认打包是不会认为这些资源文件需要打入包内，所以在启动的时候老是会报Invalid bound statement (not found)，而如果资源文件放在resources文件夹下面就不会有问题，这与maven的资源存放机制有关。如果要求maven打包的时候将java包下面的非*.java文件也打入包中，则需要上面这这个配置项。
