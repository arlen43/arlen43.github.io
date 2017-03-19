---
layout: post
title: ActiveMQ安装与验证
excerpt: ActiveMQ的基本介绍和安装配置
tags: ActiveMQ
source: virgin
---

## 基础信息 
1. 官网地址
http://activemq.apache.org/

2. .net客户端支持
首页：http://activemq.apache.org/nms/
下载：http://activemq.apache.org/nms/download.html

3. java客户端支持

使用maven直接支持

```xml
<dependency>
  <groupId>org.apache.activemq</groupId>
  <artifactId>activemq-all</artifactId>
  <version>5.13.3</version>
</dependency>
```

当然可以选择适用自己的子包， activemq-client, activemq-broker, activemq-xx-store etc.

## 基本概念

*ActiveMQ使用了Apache的Camel框架，Camel是一个轻量级的集成框架，它实现了所有的EIP（**Enterprise Integration Patterns 企业集成模式**）*

### 1. 消息模式

1. 点对点，product/consumer模式
2. 发布/订阅，pub/sub模式

具体使用见自己的业务场景

### 2. Broker节点

代表一个运行ActiveMQ的节点

### 3. Transport传输方式

前支持的有：VM Transport、TCP Transport、NIO Transport、SSL Transport、Peer Transport、UDP Transport、Multicast Transport、HTTP and HTTPS Transport、WebSockets Transport、Failover Transport、Fanout Transport、Discovery Transport、ZeroConf Transport等

1. VM  Transport：允许客户端和Broker直接在VM内部通信，采用的连接不是Socket连接，而是方法的直接调用，从而避免了网络开销，但是应用场景仅限于客户端和Broker在同一JVM环境下；

2. TCP Transport：客户端通过TCP Socket连接到远程Broker。配置语法：tcp://hostname:port?transportOptions

3. HTTP and HTTPS Transport：允许客户端使用REST或者Ajax的方式进行连接。这意味着可以直接使用Javascript向ActiveMQ发送消息。

4. WebSockets Transport：允许客户端通过HTML5标准的WebSockets方式连接到Broker。

5. Failover Transport：青龙系统MQ采用的就是这种连接方式。这种方式具备自动重新连接的机制，工作在其他Transport的上层，用于建立可靠的传输。允许配置任意多个的URI，该机制将会自动选择其中的一个URI来尝试连接。配置语法：failover:(tcp://localhost:61616,tcp://localhost:61617,.....)?transportOptions

6. Fanout Transport：主要适用于生产消息发向多个代理。如果多个代理出现环路，可能造成消费者接收重复的消息。所以，使用该协议时，最好将消息发送给多个不相连接的代理。

### 4. 持久化

默认是KahaDB（本地文件系统，5.3之前推荐使用）、JDBC（支持各种数据库，长久持久化推荐使用）、LevelDB（本地文件系统，5.9之后推荐使用） ，具体见三种持久化方式笔记

### 5. 集群方案

Shared File System Master Slave、JDBC Master Slave、Replicated LevelDB Store

官网地址：http://activemq.apache.org/masterslave.html

具体见笔记，配置及使用

## Linux下安装及配置使用

帮助文档：http://activemq.apache.org/using-activemq-4.html

我们还没有那么牛逼，去改源代码，然后重新编译安装。我们只安装人家编译好的二进制程序，具体如下。

### 1. 下载安装包

Download the activemq zipped tarball file to the Unix machine, using either a browser or a tool, i.e., wget, scp, ftp, etc. for example:
(see Download -> "The latest stable release")

```shell
wget http://activemq.apache.org/path/tofile/apache-activemq-x.x.x-bin.tar.gz
```

### 2. 解压安装包

Extract the files from the zipped tarball into a directory of your choice. For example:

```shell
cd [activemq_install_dir]
tar zxvf activemq-x.x.x-bin.tar.gz
```

### 3. 启动ActiveMQ

Starting ActiveMQ

From a command shell, change to the installation directory and run ActiveMQ as a **foregroud** process:

```shell
cd [activemq_install_dir]/bin
./activemq console
```

From a command shell, change to the installation directory and run ActiveMQ as a **daemon** process:

```shell
cd [activemq_install_dir]/bin
./activemq start
# 以下是输出
INFO: Loading '/usr/local/apache-activemq-5.13.3//bin/env'
INFO: Using java '/usr/local/jdk1.7.0_60/bin/java'
INFO: Starting - inspect logfiles specified in logging.properties and log4j.properties to get details
INFO: pidfile created : '/usr/local/apache-activemq-5.13.3//data/activemq.pid' (pid '1687')
```

更多启动方式，请看链接：

### 4. 验证启动是否成功

* 浏览器打开 `http://127.0.0.1:8161/admin/`，我的测试环境为`http://192.168.226.71:8161/admin/`，用户名密码为admin/admin
* 跳转到Queues页，输入一个队列名`test_001`，点击create，可以看到多了一个队列
* 点击Operations中的Send to，发送一条测试消息到队列，发送完毕后可以看到当前队列深度为1
* 查看mq日志`[activemq_install_dir]/data/activemq.log`，可以看到如下类似输出
  `2016-06-20 14:45:20,616 | INFO  | Apache ActiveMQ 5.13.3 (localhost, ID:localhost.localdomain-54957-1466405120141-0:1) started | org.apache.activemq.broker.BrokerService | main`
* ActiveMQ的默认端口是61616，如果进程正常启动，可以查看端口是否已处于监听状态
    ```shell
    # netstat -anpt | grep 61616
    tcp        0      0 :::61616                    :::*                        LISTEN      1687/java 
    ```
	
### 5. 监控ActiveMQ

* 可以直接访问ActiveMQ的控制台，来判断应用是否挂掉，地址： `http://127.0.0.1:8161/admin/`，用户名密码为admin/admin，可以在`conf/jetty-real.properties`中配置
* 使用JMX来监控ActiveMQ，具体查看文档`docs/WebConsole-README.txt`

### 6. 停止ActiveMQ

* 如果是前台进程，直接 Ctrl + C
* 如果是守护进程（后台进程）
    ```shell
    cd [activemq_install_dir]/bin
    ./activemq stop
    ```

### 7. 配置ActiveMQ

| 可选配置项 | 描述 | 使用方式 | 语法 |
| --- | --- | --- | --- |
|xbean|通过指定xbean-spring的xml配置文件来启动|activemq start xbean:sample.xml|http://activemq.apache.org/xml-configuration.html|
|broker|通过指定URI来启动|activemq start broker:***|broker:(transportURI,network:networkURI)/brokerName?brokerOptions，http://activemq.apache.org/broker-uri.html|
|properties|通过指定一个properties文件来启动|activemq start properties:sample.properties|http://activemq.apache.org/broker-properties-uri.html|
