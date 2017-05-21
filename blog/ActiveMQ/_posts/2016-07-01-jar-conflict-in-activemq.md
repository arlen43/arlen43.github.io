---
layout: post
title: Pom 引入activemq-all的时候 slf4j-log4j12 包冲突
tags: ActiveMQ
source: virgin
---

配置activeMQ时，官网推荐配置如下：

```xml
<dependency>
  <groupId>org.apache.activemq</groupId>
  <artifactId>activemq-all</artifactId>
  <version>5.14.4</version>
</dependency>
```

然后一般项目中都有引自己的slf4j-**相关包，然后就会报冲突
```java
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/E:/work/myeclipse-tomcat-7.0.54/webapps/ssmDemo-web/WEB-INF/lib/activemq-all-5.9.0.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/E:/work/myeclipse-tomcat-7.0.54/webapps/ssmDemo-web/WEB-INF/lib/slf4j-log4j12-1.7.7.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
```

## 1 利用Maven的Exclude
例如像下面这样
```xml
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-all</artifactId>
    <version>5.14.4</version>
    <exclusions>
        <exclusion>
            <artifactId>log4j</artifactId>
            <groupId>log4j</groupId>
        </exclusion>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

然后发现不起作用，根据 [Maven官网](http://maven.apache.org/guides/introduction/introduction-to-optional-and-excludes-dependencies.html)的介绍，exclusion就是用来排除依赖的，但是为什么不起作用呢？

查看activemq-all的pom文件，发现pom中使用了maven-plugin：maven-shade-plugin来构建jar包，导致exclusion不生效。解决方案如下：

配置shade-plugin时
```xml
<configuration>
    <shadedArtifactAttached>false</shadedArtifactAttached>
    <shadedClassifierName>jar-with-dependencies</shadedClassifierName>
    <artifactSet>
        <excludes>
            <exclude>org.slf4j:log4j-over-slf4j</exclude>
            <exclude>org.slf4j:jcl-over-slf4j</exclude>
            <exclude>org.slf4j:slf4j-log4j12</exclude>
            <exclude>io.netty:netty:3.2.5.Final</exclude>
        </excludes>
    </artifactSet>
</configuration>
```
来自 [网上](http://bbs.csdn.net/topics/391959528)，还没空验证。

## 2. 不要依赖activemq-all，而是依赖需要的子包
参照 [官网](http://activemq.apache.org/version-5-initial-configuration.html)
```xml
<!-- activemq -->
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
```
