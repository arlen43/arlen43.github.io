---
layout: post
title: 配置及使用——高可用、可负载集群配置
tags: ActiveMQ
source: virgin
---

官网一些用户提交的示例配置：http://activemq.apache.org/user-submitted-configurations.html
完整配置及介绍：http://activemq.apache.org/features.html
Using ActiveMQ5：http://activemq.apache.org/using-activemq-5.html
参照此贴：http://www.open-open.com/lib/view/open1400126457817.html

默认配置
|类别|默认值|配置文件|具体配置项|
|---|---|---|---|
|控制台端口|8186|conf/jetty.xml|`<bean id="jettyPort" class="*"><property name="port" value="8161"/></bean>`|
|TCP服务端口|61616|conf/activemq.xml|`<transportConnector name="openwire" uri="tcp://localhost:61616" discoveryUri="multicast://default"/>`|
|Broker名称|localhost|conf/activemq.xml|`<broker xmlns="http://activemq.apache.org/schema/core" brokerName="localhost" dataDirectory="${activemq.data}">`|
以上默认配置均可修改。

**activemq.xml配置**
ActiveMQ的配置均在activemq.xml中配置，如下是采用spring schema简化的示例配置。

本来就是基于spring和jetty的项目，故此用spring schema简化配置
```xml
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:amq="http://activemq.apache.org/schema/core"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">
 
  <amq:broker useJmx="false" persistent="false">
    <amq:transportConnectors>
      <amq:transportConnector uri="tcp://localhost:0" />
    </amq:transportConnectors>
  </amq:broker>

</beans>
```
其他的配置详见官网，无非就是topic分发的策略、持久化、network等的配置，下面主要讲解高可用集群配置。

# 客户端使用集群
## Queue consumer clusters——消费者
ActiveMQ支持多消费者的高可用及负载均衡。即企业中所说的竞争消费者模式（competing consumers pattern）

![客户端集群]({{site.url}}/assets/img-blog/ActiveMQ/consumer-clusters.png)

这种模式，接收producers发送的消息，入队列，并将其分发给所有注册过的consumer。优势如下：

* 热加载，新注册一个consumer，不需要修改任何配置，即可加入到consumer集群
* 比使用系统负载均衡要好，系统负载均衡都是基于检测系统来确定哪些系统不可用，并不推送消息给这些不可用的系统。而竞争消费者模式，即使没有系统监控，一个竞争失败的消费者，也不会收到消息。
* 高可用性，如果一个consumer挂了，那么一些丢失的消息则会传给其他consumer；
当然，如果你让consumer的处理顺序化，这种模式并不是最理想的。但是为了保持上边的优势，可以将这种模式跟其他模式结合，比如exclusive consumers和message group模式

## Broker clusters —— 生产者
JMS最常用的模式是，有一个broker集合，jms client连接到其中一个，如果连接的broker挂了，则自动连接其他的。

具体使用就是在Jms client处，使用failover协议连接broker。e.g. `failover:(tcp://192.168.226.71:61616,tcp://192.168.226.71:61617,tcp://192.168.226.70:61616,tcp://192.168.226.70:61617)`

配置完failover之后，可以看到生产者所打印的日志，一直在定时去检查哪个URI可以使用

```java
WriteChecker: 10000ms elapsed since last write check.
WriteCheck[tcp://192.168.226.71:61616,tcp://192.168.226.71:61617,tcp://192.168.226.70:61616,tcp://192.168.226.70:61617]
```

所以，我们有多个broker，只需要在client中配置（静态指定或动态查找），即可使客户端连到其中一个broker并工作。但是，一个broker并不知道其他brokers上的consumers。所以，当一个broker上没有consumer时，所有的消息都会堆积。我们通过将brokers建立成一个网络，来存储或者传输消息，保证所有brokers同步（高可用），见下Networks of brokers

# 高可用、可负载集群搭建

如上，客户端生产者、消费者均可使用broker集群，那么具体的broker集群应该怎么配置呢？

ActiveMQ支持两种集群模式，Broker cluster（Networks of brokers）、Master/Slave。broker-cluster即搭建一个broker集群，所有brokers都处在同一个网络中，客户端访问其中任何一个都是相同的效果。Master/Slave即搭建主备，保证broker的高可用高可靠性，其中一个broker挂掉，那么会选择master/slave中的其他任意一个broker当做master。当然我们会发现，这两种模式，一个是可负载，一个是高可用，如果单独使用其中一个，从系统的稳定性、高可用等方面讲是不够的，所以尝试着将两种模式融合搭建应该是一个很好地选择，ActiveMQ在5.9之后也支持了这种模式。

## 1 Networks of brokers —— 可负载broker集群

如上所说的问题，为了防止多producer、多consumer情况下，一个独立的broker堆积消息，ActiveMQ提供了将broker构建网络，并使其在网络中传输同步消息。这样，不但保持了broker之间的同步，并且，我们可以知道一个network下，总共有多少个client，并且可以根据需要随时扩展broker；

优点：

* 可负载，对客户端来说随便用哪个broker都是一样的
* 一致性，各个broker之间都是同步的、互知的

缺点：

* 容灾性差，如果一个producerA连接到broker1，producer生产了很多消息到broker1，消息目前处于pending状态，此时broker1挂了，那么网络中的其他节点broker并没有同步到broker1中的消息。

官网配置连接：http://activemq.apache.org/networks-of-brokers.html

示例配置，broker1和broker2建立cluster：

### 1.1 static discovery 配置

具体可查看${activemq}/example/config/activemq-static-network-broker1/2.xml
```xml
<!-- broker1 -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://activemq.org/config/1.0">
    <!-- 第一步：修改broker的名字，一个cluster中的名字不能重复 --> 
    <broker brokerName="static-broker1" persistent="false" useJmx="false"> 
    <!-- 第二步：增加network节点 -->
    <!--
        The store and forward broker networks ActiveMQ will listen to.
        We'll leave it empty as duplex network will be configured by another broker
        Take a look at activemq-static_network-broker2.xml for example
    -->
    <networkConnectors>
    </networkConnectors>
     <!-- 第三步：更改自己的服务端口 -->
     <transportConnectors>
        <transportConnector name="openwire" uri="tcp://localhost:61616"/>
     </transportConnectors>
  </broker>
</beans>
```
```xml
<!-- broker2 -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://activemq.org/config/1.0">
    <!-- 第一步：修改broker的名字，一个cluster中的名字不能重复 --> 
    <broker brokerName="static-broker2" persistent="false" useJmx="false"> 
    <!-- 第二步：增加network节点 -->
    <!--
        The store and forward broker networks ActiveMQ will listen to
        Create a duplex connector to the first broker
    -->
    <networkConnectors>
        <networkConnector uri="static:(tcp://localhost:61616)" duplex="true"/>
    </networkConnectors>
     <!-- 第三步：更改自己的服务端口 -->
     <transportConnectors>
        <transportConnector name="openwire" uri="tcp://localhost:61618"/>
     </transportConnectors>
  </broker>
</beans>
```
### 1.2 dynamic discovery 配置
具体可查看${activemq}/example/config/activemq-dynamic-network-broker1/2.xml
```xml
<!-- broker1 -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://activemq.org/config/1.0">
    <!-- 第一步：修改broker的名字，一个cluster中的名字不能重复 --> 
    <broker brokerName="dynamic-broker1" persistent="false" useJmx="false"> 
    <!-- 第二步：增加network节点 -->
    <!--
        Configure network connector to use multicast protocol
        For more information, see
        http://activemq.apache.org/multicast-transport-reference.html
    -->
    <networkConnectors>
      <networkConnector uri="multicast://default"
        dynamicOnly="true"
        networkTTL="3"
        prefetchSize="1"
        decreaseNetworkConsumerPriority="true" />
    </networkConnectors>
     <!-- 第三步：更改自己的服务端口 -->
     <transportConnectors>
        <transportConnector name="openwire" uri="tcp://localhost:61616" discoveryUri="multicast://default" />
     </transportConnectors>
  </broker>
</beans>
```
```xml
<!-- broker2 -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://activemq.org/config/1.0">
    <!-- 第一步：修改broker的名字，一个cluster中的名字不能重复 --> 
    <broker brokerName="dynamic-broker2" persistent="false" useJmx="false"> 
    <!-- 第二步：增加network节点 -->
    <networkConnectors>
      <networkConnector uri="multicast://default"
        dynamicOnly="true"
        networkTTL="3"
        prefetchSize="1"
        decreaseNetworkConsumerPriority="true" />
    </networkConnectors>
     <!-- 第三步：更改自己的服务端口 -->
     <transportConnectors>
        <transportConnector name="openwire" uri="tcp://localhost:61618" discoveryUri="multicast://default" />
     </transportConnectors>
  </broker>
</beans>
```
### 1.3 MasterSlave Discovery配置
```xml
<networkConnectors>
  <networkConnector uri="masterslave:(tcp://host1:61616,tcp://host2:61616,tcp://..)"/>
</networkConnectors>
```
### 1.4 其他Transaction的配置详见官网。

## 2 Master slave —— 高可用broker主备集群
如Networks of brokers中所说的问题，万一Networks中的一个broker挂掉，那么正在生产的消息就会丢失，这样系统的可靠性、高可用性不高。所以，我们通过共享（消息）存储介质来构建主备部署，每个broker节点都在竞争存储目录的锁，保证一个boker节点挂了，可以让客户端透明的使用另一个选中为master的节点。

一个主备竞争示意图，先是broker1作为master，broker1挂了后，broker2竞争到锁，作为master。等到broker1重新启动，因为锁还为broker2所有，所以还是作为slave。

![主备竞争1]({{site.url}}/assets/img-blog/ActiveMQ/master-slave-compete-1.png)
![主备竞争2]({{site.url}}/assets/img-blog/ActiveMQ/master-slave-compete-2.png)
![主备竞争3]({{site.url}}/assets/img-blog/ActiveMQ/master-slave-compete-3.png)

优点：

* 可靠性、高可用——一个节点挂了，可以有其他节点竞争为master继续服务

缺点：

* 无负载——每次只有一个节点提供服务，其他节点仅仅作为备机（slave）等待master挂掉再服务。

官网配置连接：http://activemq.apache.org/masterslave.html

Master slave模式，官网总共提供了三种方案

|Master Slave Type|Requirements|Pros|条件|
|---|---|---|---|
|Shared File System Master Slave|A shared file system such as a SAN|Run as many slaves as required. Automatic recovery of old masters|需要一个共享的文件系统，如果系统功底不深或者没有运维，最好别尝试|
|JDBC Master Slave|A Shared database|Run as many slaves as required. Automatic recovery of old masters|Requires a shared database. Also relatively slow as it cannot use the high performance journal，数据库用的多，慢点也无所谓，毕竟熟悉|
|Replicated LevelDB Store|ZooKeeper Server|Run as many slaves as required. Automatic recovery of old masters. Very fast.|Requires a ZooKeeper server.逼格比较高的东西，但是很强大，让master/cluster也有负载功能|

### 2.1 共享文件系统模式

如下配置即可，多个broker配置相同的共享存储目录，broker会自动为该目录加锁，其他broker也即拿不到锁。
```xml
<persistenceAdapter>
    <!-- kahaDB -->
    <kahaDB directory="/sharedFileSystem/sharedBrokerData"/>
    <!-- levelDB -->
    <!-- <levelDB directory="/sharedFileSystem/sharedBrokerData"/> -->
    <!-- amqPersistenceAdapter -->
    <!-- <amqPersistenceAdapter directory="/sharedFileSystem/sharedBrokerData"/> -->
</persistenceAdapter>
```
MasterSlave模式，客户端使用，仍然使用failover协议，上边也讲了，谁获取到文件目录锁谁就是master并提供服务，slave就一直等待，并不提供服务。测试如下，分别启动共享文件目录的master、slave：

![启动master和slave]({{site.url}}/assets/img-blog/ActiveMQ/master-slave-has-start.png)

master 日志输出：

```java
2016-07-04 15:27:42,109 | INFO  | Refreshing org.apache.activemq.xbean.XBeanBrokerFactory$1@4bdc725b: startup date [Mon Jul 04 15:27:42 CST 2016]; root of context hierarchy | org.apache.activemq.xbean.XBeanBrokerFactory$1 | main
2016-07-04 15:27:43,462 | INFO  | Using Persistence Adapter: KahaDBPersistenceAdapter[/usr/local/activemq-share-data] | org.apache.activemq.broker.BrokerService | main
2016-07-04 15:27:43,791 | INFO  | PListStore:[/usr/local/activemq-master/data/master-broker-70/tmp_storage] started | org.apache.activemq.store.kahadb.plist.PListStoreImpl | main
2016-07-04 15:27:43,941 | INFO  | Apache ActiveMQ 5.13.3 (master-broker-70, ID:localhost.localdomain-43775-1467617263810-0:1) is starting | org.apache.activemq.broker.BrokerService | main
2016-07-04 15:27:43,967 | INFO  | Listening for connections at: tcp://localhost.localdomain:61616?maximumConnections=1000&wireFormat.maxFrameSize=104857600 | org.apache.activemq.transport.TransportServerThreadSupport | main
2016-07-04 15:27:43,973 | INFO  | Connector openwire started | org.apache.activemq.broker.TransportConnector | main
2016-07-04 15:27:43,978 | INFO  | Apache ActiveMQ 5.13.3 (master-broker-70, ID:localhost.localdomain-43775-1467617263810-0:1) started | org.apache.activemq.broker.BrokerService | main
```
slave 日志输出

```java
2016-07-04 15:29:25,471 | INFO  | Refreshing org.apache.activemq.xbean.XBeanBrokerFactory$1@63c3cd50: startup date [Mon Jul 04 15:29:25 CST 2016]; root of context hierarchy | org.apache.activemq.xbean.XBeanBrokerFactory$1 | main
2016-07-04 15:29:26,822 | INFO  | Using Persistence Adapter: KahaDBPersistenceAdapter[/usr/local/activemq-share-data] | org.apache.activemq.broker.BrokerService | main
2016-07-04 15:29:26,835 | INFO  | Database /usr/local/activemq-share-data/lock is locked by another server. This broker is now in slave mode waiting a lock to be acquired | org.apache.activemq.store.SharedFileLocker | main
```
可以看到slave一直没有启动，等停止master之后如下：

![停止master后slave启动]({{site.url}}/assets/img-blog/ActiveMQ/master-stop-and-cluster-start.png)

slave 日志输出

```java
2016-07-04 15:31:37,301 | INFO  | KahaDB is version 6 | org.apache.activemq.store.kahadb.MessageDatabase | main
2016-07-04 15:31:37,323 | INFO  | Recovering from the journal @1:503 | org.apache.activemq.store.kahadb.MessageDatabase | main
2016-07-04 15:31:37,328 | INFO  | Recovery replayed 46 operations from the journal in 0.017 seconds. | org.apache.activemq.store.kahadb.MessageDatabase | main
2016-07-04 15:31:37,355 | INFO  | PListStore:[/usr/local/activemq-slave/data/broker-slave-70/tmp_storage] started | org.apache.activemq.store.kahadb.plist.PListStoreImpl | main
2016-07-04 15:31:37,504 | INFO  | Apache ActiveMQ 5.13.3 (broker-slave-70, ID:localhost.localdomain-55790-1467617497373-0:1) is starting | org.apache.activemq.broker.BrokerService | main
2016-07-04 15:31:37,530 | INFO  | Listening for connections at: tcp://localhost.localdomain:61617?maximumConnections=1000&wireFormat.maxFrameSize=104857600 | org.apache.activemq.transport.TransportServerThreadSupport | main
2016-07-04 15:31:37,536 | INFO  | Connector openwire started | org.apache.activemq.broker.TransportConnector | main
2016-07-04 15:31:37,541 | INFO  | Apache ActiveMQ 5.13.3 (broker-slave-70, ID:localhost.localdomain-55790-1467617497373-0:1) started | org.apache.activemq.broker.BrokerService | main
```

### 2.2 共享数据库模式

只需要配置`<jdbcPersistenceAdapter />`即可，只需要配置一个或多个broker共用一个数据库即可。因为所有的broker都会去获取共享表的锁，数据库会保证只有一个broker获得锁。下面是一个完整配置，详见`${activemq}/example/conf/activemq-jdbc-performance.xml` 。记得得添加数据库连接jar包到${activemq}/lib目录下。

```xml
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">
 
  <!-- Allows us to use system properties as variables in this configuration file -->
  <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
      <property name="locations">
          <value>file:${activemq.conf}/credentials.properties</value>
      </property>
  </bean>
 
  <broker useJmx="true" brokerName="jdbcBroker" xmlns="http://activemq.apache.org/schema/core">
        <destinationPolicy>
            <policyMap>
              <policyEntries>
                <policyEntry topic=">" expireMessagesPeriod="0" prioritizedMessages="false">
                </policyEntry>
                <policyEntry queue=">" expireMessagesPeriod="0" prioritizedMessages="false">
                </policyEntry>
              </policyEntries>
            </policyMap>
        </destinationPolicy>
 
    <!--
        See more database locker options at http://activemq.apache.org/pluggable-storage-lockers.html
    -->
    <persistenceAdapter>
       <!--  for mysql-ds below, add attribute: dataSource="#mysql-ds" -->
       <jdbcPersistenceAdapter dataDirectory="${activemq.data}" cleanupPeriod="0" />
    </persistenceAdapter>
 
    <transportConnectors>
       <transportConnector name="openwire" uri="tcp://0.0.0.0:61616"/>
    </transportConnectors>
  </broker>
 
  <!-- MySql DataSource Sample Setup -->
  <bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost/activemq?relaxAutoCommit=true"/>
    <property name="username" value="root"/>
    <property name="password" value="root"/>
    <property name="maxTotal" value="200"/>
    <property name="poolPreparedStatements" value="true"/>
  </bean>
 
</beans>
```
### 2.3 zookeeper，参照官网。
### 2.4 客户端使用
客户端使用failover协议，比如`failover:(tcp://broker1:61616,tcp://broker2:61616,tcp://broker3:61616)`，所有的broker中，只有master节点会启动自己的服务端口，所以客户端只会连接到master一个节点！

## 3 Cluster - MasterSlave 模式结合
如本章最初所讲，和cluster、mastersalve两节所讲弊端，将两种模式结合似乎是一种完美的解决方案。举个例子配置，A、C建立可负载Cluster，A/B、C/D各自建立MasterSlave。假如有六台服务器供使用，分别为host1-6。以下仅仅是关键配置：
```xml
<!-- broker A, host1 -->
<broker brokerName="brokerA">
    <networkConnectors>
      <networkConnector uri="masterslave:(tcp://host3:61616,tcp://host4:61616)"/>
    </networkConnectors>
    <persistenceAdapter>
       <jdbcPersistenceAdapter dataDirectory="${activemq.data}" dataSource="#mysql-ds" />
    </persistenceAdapter>
    <transportConnectors>
       <transportConnector name="openwire" uri="tcp://localhost:61616" discoveryUri="multicast://default" />
    </transportConnectors>
</broker>
<bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <!-- host5 use for mysql1 -->
    <property name="url" value="jdbc:mysql://host5/activemq?relaxAutoCommit=true"/>
    <property name="username" value="activemq"/>
    <property name="password" value="activemq"/>
    <property name="maxTotal" value="200"/>
    <property name="poolPreparedStatements" value="true"/>
</bean>
```
```xml
<!-- broker B, host2 -->
<broker brokerName="brokerB">
    <networkConnectors>
      <networkConnector uri="masterslave:(tcp://host3:61616,tcp://host4:61616)"/>
    </networkConnectors>
    <persistenceAdapter>
       <jdbcPersistenceAdapter dataDirectory="${activemq.data}" dataSource="#mysql-ds" cleanupPeriod="0" />
    </persistenceAdapter>
    <transportConnectors>
       <transportConnector name="openwire" uri="tcp://localhost:61616" discoveryUri="multicast://default" />
    </transportConnectors>
</broker>
<bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://host5/activemq?relaxAutoCommit=true"/>
    <property name="username" value="activemq"/>
    <property name="password" value="activemq"/>
    <property name="maxTotal" value="200"/>
    <property name="poolPreparedStatements" value="true"/>
</bean>
```
```xml
<!-- broker C, host3-->
<broker brokerName="brokerC">
    <networkConnectors>
      <networkConnector uri="masterslave:(tcp://host1:61616,tcp://host2:61616)"/>
    </networkConnectors>
    <persistenceAdapter>
       <jdbcPersistenceAdapter dataDirectory="${activemq.data}" dataSource="#mysql-ds" cleanupPeriod="0" />
    </persistenceAdapter>
    <transportConnectors>
       <transportConnector name="openwire" uri="tcp://localhost:61616" discoveryUri="multicast://default" />
    </transportConnectors>
</broker>
<bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <!-- host6 use for mysql2 -->
    <property name="url" value="jdbc:mysql://host6/activemq?relaxAutoCommit=true"/>
    <property name="username" value="activemq"/>
    <property name="password" value="activemq"/>
    <property name="maxTotal" value="200"/>
    <property name="poolPreparedStatements" value="true"/>
</bean>
```
```xml
<!-- broker D, host4-->
<broker brokerName="brokerD">
    <networkConnectors>
      <networkConnector uri="masterslave:(tcp://host1:61616,tcp://host2:61616)"/>
    </networkConnectors>
    <persistenceAdapter>
       <jdbcPersistenceAdapter dataDirectory="${activemq.data}" dataSource="#mysql-ds" cleanupPeriod="0" />
    </persistenceAdapter>
    <transportConnectors>
       <transportConnector name="openwire" uri="tcp://localhost:61616" discoveryUri="multicast://default" />
    </transportConnectors>
</broker>
<bean id="mysql-ds" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://host6/activemq?relaxAutoCommit=true"/>
    <property name="username" value="activemq"/>
    <property name="password" value="activemq"/>
    <property name="maxTotal" value="200"/>
    <property name="poolPreparedStatements" value="true"/>
</bean>
```
至此，一个完美的ActiveMQ的集群搭建完毕，一个实际的例子，见笔记集群配置实战。