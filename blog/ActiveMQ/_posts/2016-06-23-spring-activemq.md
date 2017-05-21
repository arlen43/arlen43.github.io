---
layout: post
title: Spring集成ActiveMQ
tags: ActiveMQ
source: virgin
---

## 配置JMSFactory、JmsTemplate
Apache官网链接：http://activemq.apache.org/spring-support.html

### 0. 添加POM依赖
```xml
<dependency>
<groupId>org.apache.activemq</groupId>
<artifactId>activemq-all</artifactId>
<version>5.13.3</version>
</dependency>
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jms</artifactId>
    <version>${springframework-version}</version>
</dependency>
```

### 1. 配置JMSFactory
配置JMSFactory（ActiveMQ JMS Client）就跟配置Spring的其他简单Bean一样，就是配置一个ActiveMQConnectionFactory的工厂实例。从连接工厂中获取JMSTemplate。

*eg. 通过指定ip，端口配置JMS Client*

```xml
<bean id="jmsFactory" class="org.apache.activemq.ActiveMQConnectionFactory">
    <property name="brokerURL">
      <value>tcp://localhost:61616</value>
    </property>
</bean>
```

*使用Spring命名空间简化配置*

```xml
<beans
  xmlns="http://www.springframework.org/schema/beans"
  xmlns:amq="http://activemq.apache.org/schema/core"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-2.0.xsd
  http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core.xsd">
 
  <amq:connectionFactory id="jmsFactory" brokerURL="vm://localhost"/>
</beans>
```
### 2. 配置Spring JmsTemplate

spring提供了一个虚处理类，JMSTemplate，抽象模板类，让底层实现透明化。可以使用其发送消息等。
```xml
<bean id="myJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory">
      <!-- 或者 <constructor-arg ref="jmsFactory" /> -->
      <ref local="jmsFactory"/>
    </property>
</bean>
```
```java
@Component
public class JmsQueueSender {
 
    @Resource
    private JmsTemplate jmsTemplate;
    @Resource(name = "destination")
    private Queue queue;
 
    public void send(final String msg) {
        this.jmsTemplate.send(queue, new MessageCreator() {
            @Override
            public Message createMessage(Session session) throws JMSException {
                return session.createTextMessage(msg);
            }
        });
    }
 
    public void sendWithConvert(String msg) {
        this.jmsTemplate.convertAndSend(queue, msg, new MessagePostProcessor() {
            @Override
            public Message postProcessMessage(Message message) throws JMSException {
                message.setIntProperty("id", 1);
                return message;
            }
        });
    }
}
```
需要注意的是，默认JMSTemplate会为每一次的消息发送创建一个新的connection、session、producer，很耗资源和性能，为了解决此问题

* Apache提供了JSM连接池PooledConnectionFactory（activemq-pool包）。
  它可以缓存connection、session、producer，但是不会缓存consumer，所以只适合生产者发送消息。

  *官方解释：因consumer一般都是异步的，也就是说broker代理会把producer的消息放在一个消费者预取缓存中，当消费者准备好就可以随时去拿。*但是，消费者拿也要建立连接，也会耗性能，这部分也可以做缓存。

  如下配置：
  ```xml
  <!-- a pooling based JMS provider -->
  <bean id="jmsFactory" class="org.apache.activemq.pool.PooledConnectionFactory" destroy-method="stop">
    <property name="connectionFactory">
      <bean class="org.apache.activemq.ActiveMQConnectionFactory">
        <property name="brokerURL">
          <value>tcp://localhost:61616</value>
        </property>
      </bean>
    </property>
  </bean>
 
  <!-- Spring JMS Template -->
  <bean id="myJmsTemplate" class="org.springframework.jms.core.JmsTemplate">
    <property name="connectionFactory">
      <ref local="jmsFactory"/>
    </property>
  </bean>
  ```

* Spring提供了缓存池CachingConnectionFactory（spring-jms包）
  它会缓存session、MessageProducer、MessageConsumer，它继承了SingleConnectionFactory，在所有createConnection()方法中，均返回同一个共享连接，并且忽略connection.close/stop方法，当该连接异常会自动创建一个新连接。
  如下配置：
  ```xml
  <!-- creates an activemq connection factory using the amq namespace -->
  <amq:connectionFactory id="amqConnectionFactory" 
   brokerURL="${jms.url}" userName="${jms.username}" password="${jms.password}" />
   
  <!-- CachingConnectionFactory Definition, sessionCacheSize property is the number of sessions to cache -->
  <bean id="connectionFactory" class="org.springframework.jms.connection.CachingConnectionFactory">
	<constructor-arg ref="amqConnectionFactory" />
        <!-- exception 处理类 -->
	<property name="exceptionListener" ref="jmsExceptionListener" />
	<property name="sessionCacheSize" value="100" />
  </bean>
  <!-- JmsTemplate Definition -->
  <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
     <constructor-arg ref="connectionFactory"/>
  </bean>
  ```

如上配置中，connectionFactory中可以配置异常处理器，一个简单的异常处理器代码如下：

```java
@Component("jmsExceptionListener")
public class JmsExceptionListener implements ExceptionListener {
    @Override
    public void onException(JMSException e) {
        e.printStackTrace();
    }
}
```

## 配置Destination、MessageListener、ListenerContainer

spring官网链接：http://docs.spring.io/spring/docs/2.5.x/reference/jms.html#jms-mdp

### 普通bean配置法
#### Destination配置
```xml
<!-- queue -->
<bean id="destination" class="org.apache.activemq.command.ActiveMQQueue">
   <constructor-arg>
       <value>test.foo</value>
   </constructor-arg>
</bean>
<!-- topic -->
<bean id="topicDestination" class="org.apache.activemq.command.ActiveMQTopic"> 
    <constructor-arg value="topic1" />
</bean>
```
#### MessageListener配置

因为MessageListener全是自己实现，使用注解或者xml-bean均可。目前Spring支持3种MessageListener，分别为MessageListener、SessionAwareMessageListener、MessageListenerAdapter
##### 1. MessageListener
```java
@Component
public class JmsMessageListener implements MessageListener {
    @Override
    public void onMessage(Message message) {
        if (message instanceof TextMessage) {
            try {
                System.out.println(((TextMessage)message).getText());
            } catch (JMSException e) {
                e.printStackTrace();
            }
        } else {
            throw new IllegalArgumentException("Message must be type of TextMessage");
        }
    }
}
```
```xml
<bean id="messageListener" class="com.hongkun.oss.common.util.jms.JmsMessageListener" />
```
##### 2. SessionAwareMessageListener

onMessage方法中会将session传进来，以便可以发送回执消息，如下面的例子：
```java
@Component
public class JmsSessionAwareMessageListener implements SessionAwareMessageListener<TextMessage> {
    @Resource
    private Destination destination;
    @Override
    public void onMessage(TextMessage message, Session session) throws JMSException {
        System.out.println(message.getText());
        MessageProducer producer = session.createProducer(destination);
        Message textMessage = session.createTextMessage("已成功收到，并处理消息");
        producer.send(textMessage);
    }
}
```
```xm
<bean id="sessionMessageListener" class="com.hongkun.oss.common.util.jms.JmsSessionAwareMessageListener" />
```
##### 3. MessageListenerAdapter

其实现为一个纯粹的java bean，不依赖JMS的任何东西，如下：
```java
public interface IJmsMessageDelegate {
    void handleMessage(String message);
    void handleMessage(Map message);
    void handleMessage(byte[] message);
    void handleMessage(Serializable message);
}
```
```java
@Component
public class JmsMessageDelegate implements IJmsMessageDelegate {
    @Override
    public void handleMessage(String message) {
        // TODO Auto-generated method stub
    }
    @Override
    public void handleMessage(Map message) {
        // TODO Auto-generated method stub
    }
    @Override
    public void handleMessage(byte[] message) {
        // TODO Auto-generated method stub
    }
    @Override
    public void handleMessage(Serializable message) {
        // TODO Auto-generated method stub
    }
}
```
```xml
<bean id="messageListenerAdapter" class="org.springframework.jms.listener.adapter.MessageListenerAdapter">
    <constructor-arg>
        <bean class="com.hongkun.oss.common.util.jms.JmsMessageDelegate"/>
    </constructor-arg>
    <!-- or <constructor-arg ref="jmsMessageDelegate" /> -->
</bean>
```
#### ListenerContainer配置

Spring提供了三种Container，SimpleMessageListenerContainer、DefaultMessageListenerContainer、ServerSessionMessageListenerContainer，一般用Default就行了，至于区别，在Spring官网有详细解释，地址见本章最开始。
```xml
<bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="destination"/>
    <!-- 具体的messageListener -->
    <property name="messageListener" ref="messageListener" />
</bean>
```
### 用jms schema配置法

使用jms schema可以很大程度上简化配置，但是却让人云里雾里，不知道具体配的哪个类，可以查看schema的xsd，来确定默认用的哪个类，和每个配置中有哪些属性、哪些元素等。
```xml
<!-- listener-container有很多属性，都是默认的，如connection-factory如果不配，默认值就是connectionFactory，具体可查看xsd -->
<jms:listener-container concurrency="10" connection-factory="connectionFactory">
    <!-- listener有很多属性，具体查看xsd -->
    <jms:listener id="QueueListener" destination="test.foo" ref="messageListener" />
</jms:listener-container>
```

一个属性多点的配置实例：
```xml
<jms:listener-container connection-factory="myConnectionFactory"
                        task-executor="myTaskExecutor"
                        destination-resolver="myDestinationResolver"
                        transaction-manager="myTransactionManager"
                        concurrency="10">
    <jms:listener destination="queue.orders" ref="orderService" method="placeOrder"/>
    <jms:listener destination="queue.confirmations" ref="confirmationLogger" method="log"/>
</jms:listener-container>
```

listener-container元素的属性只是类`AbstractMessageListenerContainer`中的属性，上边讲的三种实现类的特殊属性均无法配置，故此，要用到上边三种实现类的特殊属性的话，这种配置方式并不是最好的。比如下面要讲的事务支持。

## 事务支持
参照帖子：http://haohaoxuexi.iteye.com/blog/1983532

ActiveMQ支持本地事务和分布式事务，本地事务很简单，在配置ListenerContainer时指定sessionTransacted属性即可。分布式事务需要容器支持JTA，如果支持，只需配置transactionManager属性，当该属性配置后，之前的sessionTransacted属性就不起作用。

### 本地事务

举个栗子：比如在Listener类的onMessage方法中，接收到消息后出错并抛出异常，那么刚刚接收的消息还会处于队列中，等到下次消费又会拿到。实际的测试当中，如果某条消息处理失败，则会再次消费，也即再次处理，一直会重试5次，具体配置

listener onMessage方法：
```java
@Override
public void onMessage(Message message) {
    if (message instanceof TextMessage) {
        String text = null;
        try {
            text = ((TextMessage)message).getText();
            System.out.println("=\n=\n=\n"+text+"\n=\n=\n=");
        } catch (JMSException e) {
            e.printStackTrace();
        }
        if ("".equals(text)) {
            throw new IllegalArgumentException("消息为空，无法处理");
        }
    } else {
        throw new IllegalArgumentException("Message must be type of TextMessage");
    }
}
```

```xml
<bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
    <property name="connectionFactory" ref="connectionFactory"/>
    <property name="destination" ref="destination"/>
    <property name="messageListener" ref="messageListener" />
    <!-- 指定该属性即可 -->
    <property name="sessionTransacted" value="true" />
</bean>
```

如上配置，控制台部分日志(debug级别)：
```java
15:01:52.492 [jmsContainer-1] DEBUG org.apache.activemq.ActiveMQSession:580 - ID:DESKTOP-A1D5JRD-51491-1467702079756-1:1:1 Transaction Commit :null
15:01:53.492 [jmsContainer-1] DEBUG org.apache.activemq.ActiveMQSession:580 - ID:DESKTOP-A1D5JRD-51491-1467702079756-1:1:1 Transaction Commit :null
// 如上日志一直重复，客户端监听消息就会一直去拿，每次拿都是一次事务
15:01:54.395 [jmsContainer-1] DEBUG o.apache.activemq.TransactionContext:250 - Begin:TX:ID:DESKTOP-A1D5JRD-51491-1467702079756-1:1:1
15:01:54.396 [jmsContainer-1] DEBUG o.s.j.l.DefaultMessageListenerContainer:313 - Received message of type [class org.apache.activemq.command.ActiveMQTextMessage] from consumer [Cached JMS MessageConsumer: ActiveMQMessageConsumer { value=ID:DESKTOP-A1D5JRD-51491-1467702079756-1:1:1:1, started=true }] of session [Cached JMS Session: ActiveMQSession {id=ID:DESKTOP-A1D5JRD-51491-1467702079756-1:1:1,started=true} java.lang.Object@3e0adffd]
=
 
=
// 事务回滚
15:01:54.401 [jmsContainer-1] DEBUG o.s.j.l.DefaultMessageListenerContainer:608 - Initiating transaction rollback on application exception
java.lang.IllegalArgumentException: 消息为空，无法处理
// 异常堆栈
```

或者直接使用JmsTransaction，配置如下（当然用这种方式可以直接用jms schema简化配置）：
```xml
    <bean id="jmsTransactionManage" class="org.springframework.jms.connection.JmsTransactionManager">
       <property name="connectionFactory" ref="connectionFactory" />
    </bean>
    <!-- and this is the message listener container -->
    <bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="destination" ref="destination"/>
        <property name="messageListener" ref="messageListener" />
        <!-- <property name="sessionTransacted" value="true" /> -->
        <property name="transactionManager" ref="jmsTransactionManage" />
    </bean>
```

如上配置，控制台部分日志(debug级别)：
```java
14:31:16.493 [jmsContainer-1] DEBUG o.s.j.c.JmsTransactionManager:366 - Creating new transaction with name [jmsContainer]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
14:31:16.493 [jmsContainer-1] DEBUG o.s.j.c.JmsTransactionManager:183 - Created JMS transaction on Session [Cached JMS Session: ActiveMQSession {id=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:1,started=true} java.lang.Object@494d4f59] from Connection [Shared JMS Connection: ActiveMQConnection {id=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1,clientId=ID:DESKTOP-A1D5JRD-51289-1467700267086-0:1,started=true}]
14:31:17.493 [jmsContainer-1] DEBUG o.s.j.c.JmsTransactionManager:753 - Initiating transaction commit
14:31:17.493 [jmsContainer-1] DEBUG o.s.j.c.JmsTransactionManager:223 - Committing JMS transaction on Session [Cached JMS Session: ActiveMQSession {id=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:1,started=true} java.lang.Object@494d4f59]
14:31:17.493 [jmsContainer-1] DEBUG org.apache.activemq.ActiveMQSession:580 - ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:1 Transaction Commit :null
// ... 一直循环上面的日志，也就是客户端监听消息，一直去请求broker ...
14:31:19.498 [jmsContainer-1] DEBUG o.s.j.c.JmsTransactionManager:366 - Creating new transaction with name [jmsContainer]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT
14:31:19.498 [http-bio-9080-exec-3] DEBUG o.s.j.c.CachingConnectionFactory:366 - Creating cached JMS MessageProducer for destination [queue://test.foo]: ActiveMQMessageProducer { value=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:2:1 }
14:31:19.499 [jmsContainer-1] DEBUG o.s.j.c.JmsTransactionManager:183 - Created JMS transaction on Session [Cached JMS Session: ActiveMQSession {id=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:1,started=true} java.lang.Object@494d4f59] from Connection [Shared JMS Connection: ActiveMQConnection {id=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1,clientId=ID:DESKTOP-A1D5JRD-51289-1467700267086-0:1,started=true}]
14:31:19.511 [http-bio-9080-exec-3] DEBUG o.s.jms.core.JmsTemplate:567 - Sending created message: ActiveMQTextMessage {commandId = 0, responseRequired = false, messageId = null, originalDestination = null, originalTransactionId = null, producerId = null, destination = null, transactionId = null, expiration = 0, timestamp = 0, arrival = 0, brokerInTime = 0, brokerOutTime = 0, correlationId = null, replyTo = null, persistent = false, type = null, priority = 0, groupID = null, groupSequence = 0, targetConsumerId = null, compressed = false, userID = null, content = null, marshalledProperties = null, dataStructure = null, redeliveryCounter = 0, size = 0, properties = null, readOnlyProperties = false, readOnlyBody = false, droppable = false, jmsXGroupFirstForConsumer = false, text = }
14:31:19.560 [jmsContainer-1] DEBUG o.apache.activemq.TransactionContext:250 - Begin:TX:ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:1
14:31:19.561 [jmsContainer-1] DEBUG o.s.j.l.DefaultMessageListenerContainer:313 - Received message of type [class org.apache.activemq.command.ActiveMQTextMessage] from consumer [Cached JMS MessageConsumer: ActiveMQMessageConsumer { value=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:1:1, started=true }] of transactional session [Cached JMS Session: ActiveMQSession {id=ID:DESKTOP-A1D5JRD-51289-1467700267086-1:1:1,started=true} java.lang.Object@494d4f59]
// 以下为打出的消息内容，消息为空则抛出异常
=

=
// 抛出异常后，事务回滚
14:31:19.561 [jmsContainer-1] DEBUG o.s.j.l.DefaultMessageListenerContainer:330 - Rolling back transaction because of listener exception thrown: java.lang.IllegalArgumentException: 消息为空，无法处理
// 因配置了异常handler，仅仅是打印错误，下面是错误堆栈打印
14:31:19.562 [jmsContainer-1] WARN  o.s.j.l.DefaultMessageListenerContainer:696 - Execution of JMS message listener failed, and no ErrorHandler has been set.
java.lang.IllegalArgumentException: 消息为空，无法处理
// 堆栈信息略
// 以上异常消息日志会打印多次，因为consumer第一次消费失败，又去拿一次，又失败了，又去拿，总共5-7次吧
```

### 分布式事务
需要JTA支持，具体配置参照本章所参照的帖子！

## 一个完整的配置示例
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:amq="http://activemq.apache.org/schema/core"
    xmlns:jms="http://www.springframework.org/schema/jms"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
    http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
    http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.1.xsd
    http://activemq.apache.org/schema/core http://activemq.apache.org/schema/core/activemq-core-5.13.3.xsd
    http://www.springframework.org/schema/jms http://www.springframework.org/schema/jms/spring-jms-3.2.xsd">
 
    <!-- creates an activemq connection factory using the amq namespace -->
    <amq:connectionFactory id="amqConnectionFactory"
       brokerURL="${jms.url}" userName="${jms.username}" password="${jms.password}" />
 
    <!-- CachingConnectionFactory Definition, sessionCacheSize property is the number of sessions to cache -->
    <bean id="connectionFactory" class="org.springframework.jms.connection.CachingConnectionFactory">
        <constructor-arg ref="amqConnectionFactory" />
        <!-- <property name="exceptionListener" ref="jmsExceptionListener" /> -->
        <property name="sessionCacheSize" value="100" />
    </bean>
 
    <!-- JmsTemplate Definition -->
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
       <constructor-arg ref="connectionFactory"/>
    </bean>
 
    <!-- transaction -->
    <bean id="jmsTransactionManage" class="org.springframework.jms.connection.JmsTransactionManager">
       <property name="connectionFactory" ref="connectionFactory" />
    </bean>
 
    <!-- listener container definition using the jms namespace, concurrency is the max number of concurrent listeners that can be started -->
    <jms:listener-container concurrency="10" transaction-manager="jmsTransactionManage">
        <jms:listener id="QueueListener" destination="test.foo" ref="jmsMessageListener" />
    </jms:listener-container>
 
    <!-- destination -->
    <bean id="destination" class="org.apache.activemq.command.ActiveMQQueue">
       <constructor-arg>
           <value>test.foo</value>
       </constructor-arg>
    </bean>
    <bean id="topicDestination" class="org.apache.activemq.command.ActiveMQTopic"> 
        <constructor-arg value="topic1" />
    </bean>
 
    <!-- default listener -->
    <!-- this is the Message Driven POJO (MDP) -->
    <bean id="messageListener" class="com.hongkun.oss.common.util.jms.JmsMessageListener" />
 
    <!-- session listener -->
    <bean id="sessionMessageListener" class="com.hongkun.oss.common.util.jms.JmsSessionAwareMessageListener" />
 
    <!-- adapter listener -->
    <bean id="messageListenerAdapter" class="org.springframework.jms.listener.adapter.MessageListenerAdapter">
        <constructor-arg>
            <bean class="com.hongkun.oss.common.util.jms.JmsMessageDelegate"/>
        </constructor-arg>
    </bean>
 
    <!-- and this is the message listener container -->
    <bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="destination" ref="destination"/>
        <property name="messageListener" ref="messageListener" />
        <property name="sessionTransacted" value="true" />
        <!-- <property name="transactionManager" ref="jmsTransactionManage" /> -->
    </bean>
 
</beans>
```

## 具体使用
官方示例代码，包含通过URI来配置ActiveMQ的各种参数、消息的等待及处理、消息的发送等示例

https://svn.apache.org/repos/asf/activemq/trunk/activemq-unit-tests/src/test/java/org/apache/activemq/spring/
