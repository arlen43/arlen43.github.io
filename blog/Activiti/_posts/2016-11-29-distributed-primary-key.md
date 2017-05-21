---
layout: post
title: Activiti5.10解决分布式部署的主键问题[转]
tags: Activiti
source: reprint
source_url: http://blog.csdn.net/kongqz/article/details/8027295
---

转自：http://blog.csdn.net/kongqz/article/details/8027295

## 一、概要综述
activiti5是jbpm4升级上来的一款最新版工作流引擎，已经将自己的表划分为4类：运行时、通用数据、历史数据、流程相关数据，但是有一个核心问题就是是否支持集群部署，经过我对源码的初步分析发现，他的默认主键策略是全局获取一个通用表中的字段来做增加，在大并发量的情况下会出现主键重复的问题

### 1、activiti5的默认主键策略分析：
（1）、每次需要主键的时候从act_ge_property表中的next.dbid中获取下一个主键值，但是主键增长步长是100，也就是说每次从这里获取下一个值的时候，上次是100，下次的值是200.

（2）、他们所有需要主键的表都从这个表中获取下一个值

（3）、但是他们针对性能做了一个取巧处理，就是每次步长100，将这个步长cache在本地用sychronize方法调用，也就是说一段时间内只需要在本地获取主键即可，不需要访问数据实时更新，一定程度的缓解了数据库调用压力

（4）、但是对于高并发来说，这个只能局部缓解，并发写入压力，还是有造成主键重复的概率

### 2、解决方案分析

（1）、因为activiti的主键是统一管理，直接通过集中替换主键策略就可以完成主键策略的替换

（2）、经过分析源码发现activiti的流程引擎的主键引用采用的方式是先看spring配置的 idGenerator 属性是否有外部注入，如果没有，才使用默认的主键策略生成主键，所以我们只需要针对配置文件进行主键策略的替换即可

（3）、寻找一种能产生唯一主键的生成策略，这里uuid可能是一个比较通用的解决方案，不过悲剧的是uuid对于人去识别可能不是很友好，但是对于程序来说无所谓了，而activiti本身的id字段也支持uuid算法来生成主键。

## 二、实施方案

为了快速见效，直接用activiti官方提供的activiti-explorer项目来更换主键策略来验证效果

### 1、直接找到部署的war包，更新它的配置文件，来指定主键策略。这里我们要找的就是那个applicationContext.xml文件
```java
<bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration"> 
    <property name="dataSource" ref="dataSource" /> 
    <property name="transactionManager" ref="transactionManager" /> 
    <property name="databaseSchemaUpdate" value="true" /> 
    <property name="jobExecutorActivate" value="true" /> 
  <property name="customFormTypes"> 
    <list> 
      <ref bean="userFormType"/> 
    </list> 
  </property> 
<property name="idGenerator"><bean class="org.activiti.engine.impl.persistence.StrongUuidGenerator" /></property>
</bean>  
```
### 2、将主键策略要引用的uuid这个jar包引入，这里我使用java-uuid-generator-3.1.2.jar

### 3、重新启动tomcat来做相关流程部署，发现数据库中的主键策略真的变为uuid了（这里别忘记先清理库啊，这样看起来效果才不会被干扰，因为以前一直用的是默认主键生成策略，生成的int类型的主键）