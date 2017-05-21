---
layout: post
title: Maven如何打sources包
tags: Maven
source: virgin
---

    项目中自己经常会写一些公共包，放到自己的maven仓库中，供团队起他成员使用。但是使用过程中，很多成员想看看实现源码，那就得打源码包

## 方式一：通过Maven-Plugin
```xml
<!-- 编译插件，打jar包时用 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <version>3.3</version>
    <configuration>
        <source>1.7</source>
        <target>1.7</target>
        <encoding>utf-8</encoding>
    </configuration>
</plugin>
<!-- sources包插件，打sources包时用 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-source-plugin</artifactId>
    <version>2.4</version>
    <executions>
        <execution>
            <id>attach-sources</id>
            <goals>
                <goal>jar-no-fork</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
配置好Plugin之后，用命令正常打就行了
```shell
mvn clean install -Dmaven.test.skip=true -Pprd
```

## 方式二：通过命令
```shell
mvn source:jar
``` 
该命令只单独打sources包

打完包后，就可以deploy到自己的私服了。

