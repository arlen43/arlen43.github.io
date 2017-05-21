---
layout: post
title: 三种持久化方式六种恢复策略
tags: ActiveMQ
source: virgin
---

# 三种持久化方式
ActiveMQ支持三种持久化方式，默认是KahaDB（本地文件系统，5.3之前推荐使用）、JDBC（支持各种数据库，长久持久化推荐使用）、LevelDB（本地文件系统，5.9之后推荐使用）

## KahaDB
是ActiveMQ工程的一部分，它很好的提供了各种使用方式（读、写、丢弃）消息的支持，可以特别快速的持久化它们。

ActiveMQ5.0及以上版本配置：
```xml
<broker brokerName="broker" persistent="true" useShutdownHook="false">
   <transportConnectors>
     <transportConnector uri="tcp://localhost:61616"/>
   </transportConnectors>
   <persistenceAdapter>
     <!-- ActiveMQ4.1以及更早版本配置 -->
     <!-- <kahaPersistenceAdapter dir="activemq-data" maxDataFileLength="33554432"/> -->
     <kahaDB directory="activemq-data" />
   </persistenceAdapter>
</broker>
```

## JDBC--MySql
1. 将MySql的驱动包，放入ActiveMQ的lib下
```shell
cd [activemq_install_dir]
# 拷贝 mysql-connector-java-*.*.*.jar，到当前目录
```

2. 修改ActiveMQ的配置文件
```xml
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">

  <!-- //... -->
  <broker useJmx="true" brokerName="jdbcBroker" xmlns="http://activemq.apache.org/schema/core">
    <!-- ... -->
    <persistenceAdapter>
       <!--  for mysql-ds below, add attribute: dataSource="#mysql-ds" -->
       <jdbcPersistenceAdapter dataDirectory="${activemq.data}" cleanupPeriod="0" dataSource="#mysql-ds" />
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

3. 在数据库服务器中创建数据库
```sql
create database activemq;
```

4. 重启ActiveMQ
你会发现多了下面几张表
```sql
activemq_acks
activemq_lock
activemq_msgs
```

## LevelDB
可重复的LevelDB，使用ZooKeeper从LevelDB上配置的多个broker节点中选取一个master节点，提供broker主服务 ，然后从master节点同步所有salve节点的LevelDB，使slave节点跟master节点保持一致；

        ZooKeeper：分布式应用程序协调服务！相当于负载调节的东西。

官方介绍：http://activemq.apache.org/replicated-leveldb-store.html

默认配置：
```xml
< persistenceAdapter >
       < levelDBdirectory = "activemq-data" />
</ persistenceAdapter >
```

# 六种恢复策略——消息订阅者模式
恢复策略可以让你订阅了一个主题后恢复消息，比如，你从集群中brokerA中订阅消费，此时有人将brokerA杀死，当你重新连到集群中的brokerB时，可能会丢失了一些消息。

所以，ActiveMQ支持配置“一次或者某些合适数量”的消息恢复，当你重新连接到集群中其他broker时，broker会在下一个新消息到来之前，推送之前你所配置的量的消息给你。

http://activemq.apache.org/subscription-recovery-policy.html
|Policy Name|Sample Configuration|Description|
|---|---|---|
|FixedSizedSubscriptionRecoveryPolicy|`<fixedSizedSubscriptionRecoveryPolicy maximumSize="1024"/>`|Keep a fixed amount of memory in RAM for message history which is evicted in time order.|
|FixedCountSubscriptionRecoveryPolicy|`<fixedCountSubscriptionRecoveryPolicy maximumSize="100"/>`|Keep a fixed count of last messages.
|LastImageSubscriptionRecoveryPolicy|`<lastImageSubscriptionRecoveryPolicy/>`|Keep only the last message.|
|NoSubscriptionRecoveryPolicy|`<noSubscriptionRecoveryPolicy/>`|Disables message recovery.|
|QueryBasedSubscriptionRecoveryPolicy|`<queryBasedSubscriptionRecoveryPolicy query="JMSType = 'car' AND color = 'blue'"/>`|Perform a user specific query mechanism to load any message they may have missed. Details on message selectors are available here: http://java.sun.com/j2ee/1.4/docs/api/javax/jms/Message.html|
|TimedSubscriptionRecoveryPolicy|`<timedSubscriptionRecoveryPolicy recoverDuration="60000" />`|Keep a timed buffer of messages around in memory and use that to recover new subscriptions. Recovery time is in milliseconds.|
|RetainedMessageSubscriptionRecoveryPolicy|`<retainedMessageSubscriptionRecoveryPolicy/>`|Keep the last message with ActiveMQ.Retain property set to true|


