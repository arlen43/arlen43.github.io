---
layout: post
title: Class path contains multiple SLF4J bindings
tags: Exception
source: virgin
---

## 异常堆栈

```java
java.lang.ClassCastException: org.slf4j.impl.Log4jLoggerFactory cannot be cast to ch.qos.logback.classic.LoggerContext
    at ch.qos.logback.ext.spring.LogbackConfigurer.initLogging(Unknown Source)
    at ch.qos.logback.ext.spring.web.WebLogbackConfigurer.initLogging(Unknown Source)
    at ch.qos.logback.ext.spring.web.LogbackConfigListener.contextInitialized(Unknown Source)
    at org.apache.catalina.core.StandardContext.listenerStart(StandardContext.java:5003)
    at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5517)
    at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:150)
    at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1574)
    at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1564)
    at java.util.concurrent.FutureTask.run(FutureTask.java:262)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:745)
```

## 异常分析

项目中加入ActiveMQ Spring集成后，启动便报这个错，初步估计是包冲突引起。查看日志，发下下面一行日志

```html
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/D:/Program%20Files/Apache/apache-tomcat-7.0.64-01/wtpwebapps/oss-web/WEB-INF/lib/activemq-all-5.13.3.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/D:/Program%20Files/Apache/apache-tomcat-7.0.64-01/wtpwebapps/oss-web/WEB-INF/lib/logback-classic-1.0.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
```

也即SLF4J有多个实现，所以得去掉一个，将ActiveMQ包中的SLF4J依赖去掉

## 异常解决

正常情况下，根据maven官网（http://maven.apache.org/guides/introduction/introduction-to-optional-and-excludes-dependencies.html）指示，要去掉依赖项目中的依赖包，只需要用exclusion即可。
如下做法：去掉ActiveMQ包中的SLF4J依赖
```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-all</artifactId>
    <version>5.13.3</version>
    <exclusions>
      <exclusion>
        <groupId>org.slf4j</groupId>
              <artifactId>slf4j-api</artifactId>
      </exclusion>
      <exclusion>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
      </exclusion>
      <exclusion>
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
      </exclusion>
    </exclusions>
</dependency>
```

但是ActiveMQ-all的pom文件构建有了maven的插件maven-shade-plugin，插件中包含了一堆的依赖包，无法使用maven的exclusion来排除，所以此方法不可行，故此，不用activemq-all，改用明细的包，见官网地址：http://activemq.apache.org/version-5-initial-configuration.html

最后的maven配置文件如下：
```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-broker</artifactId>
    <version>5.13.3</version>
</dependency>
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-client</artifactId>
    <version>5.13.3</version>
</dependency>
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-kahadb-store</artifactId>
    <version>5.13.3</version>
</dependency>
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-spring</artifactId>
    <version>5.13.3</version>
</dependency>
<dependency>
    <groupId>org.fusesource.hawtbuf</groupId>
    <artifactId>hawtbuf</artifactId>
    <version>1.11</version>
</dependency>
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-kahadb-store</artifactId>
    <version>5.13.3</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jms</artifactId>
    <version>${springframework-version}</version>
</dependency>
```