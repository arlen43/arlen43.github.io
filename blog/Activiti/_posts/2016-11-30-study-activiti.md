---
layout: post
title: Activiti 学习
tags: Activiti
source: virgin
---

关于流程 BPM2.0 的一本书《流程的永恒之道》
https://tkjohn.github.io/activiti-userguide/

仓库（Repository）：存放流程相关的所有静态数据，比如流程对应的xml、png
流程（Process）：即定义的流程，可以看成java中的.java文件
实例（Instance）：即流程实例，可以看成java中的new的一个对象。
活动（Activity）：即流程中的一个节点，比如一个方框
任务（Task）：即流程实例中的一个节点，比如一个方框对应的具体系统中的操作

RepositoryService/RuntimeService/TaskService/HistoryService/ManagementService/IdentityService/FormService

|表名|记录内容|
| --- | --- |
|act_re_deployment|已发布的流程，只简单记一个发布时间和id|
|act_re_procdef|已发布的流程对应的流程定义，包括版本、发布人、发布id等|
|act_ru_execution|记录还未结束的流程实例，包含当前活动的key|
|act_ru_task|记录还未结束的流程任务，也即流程实例中的当前任务|
|act_ru_identitylink|记录当前未结束的流程的当前任务对应的操作人|
|act_ru_variable|记录流程中的所有变量，也即流程上下文内容|

### 基本概念
1. Execution和ProcessInstance
在Activiti中Execution和ProcessInstance都用于获取当前流程实例的相关信息。
当流程中没有分支时，Execution等同于ProcessInstance，甚至连ID也相同；
当流程中存在分支(fork, parallel gateway)，则在分支口会形成子Execution，在下一个gateway才会合并(joined)

### 项目搭建

### 非Spring示例


### 包依赖
pom文件
```xml
<dependency>
   <groupId>org.activiti</groupId>
   <artifactId>activiti-engine</artifactId>
   <version>${activiti.version}</version>
</dependency>
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-spring</artifactId>
    <version>${activiti.version}</version>
</dependency>
```

包冲突：
activiti-engine依赖了spring-beans，5.22.0中依赖的是spring-framework的4.1.5.RELEASE，可能会与你的项目中包冲突，那么此时需要排除这些依赖，用自己项目的即可。最终配置如下：
```xml
<dependency>
   <groupId>org.activiti</groupId>
   <artifactId>activiti-engine</artifactId>
   <version>${activiti.version}</version>
   <exclusions>
      <exclusion>
          <groupId>org.springframework</groupId>
          <artifactId>spring-beans</artifactId>
      </exclusion>
   </exclusions>
</dependency>
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-spring</artifactId>
    <version>${activiti.version}</version>
    <exclusions>
            <exclusion>
                <groupId>org.springframework</groupId>
                <artifactId>spring-beans</artifactId>
            </exclusion>
         </exclusions>
</dependency>
```

### 与Spring集成
spring配置文件
全部配置都在一个文件中，包括DataSource、TransactionManager
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans   http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-2.5.xsd
                           http://www.springframework.org/schema/tx      http://www.springframework.org/schema/tx/spring-tx-3.0.xsd">
 
  <bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource" >
    <property name="driverClassName" value="com.mysql.jdbc.Driver" />
    <property name="url" value="jdbc:mysql://localhost:3306/activiti" />
    <property name="username" value="root" />
    <property name="password" value="123456" />
    <property name="defaultAutoCommit" value="false" />
  </bean>
 
  <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
  </bean>
 
  <tx:annotation-driven transaction-manager="transactionManager"/>
  <context:annotation-config />
 
  <bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
    <property name="dataSource" ref="dataSource" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="databaseSchemaUpdate" value="true" />
    <property name="jobExecutorActivate" value="false" />
 
    <property name="history" value="audit"/>
    <!-- 异步执行 -->
    <property name="asyncExecutorEnabled" value="true" />
    <property name="asyncExecutorActivate" value="false" />
    <!-- 流程定义缓存 -->
    <property name="processDefinitionCacheLimit" value="10" />
 
    <!-- 事件监听 -->
    <property name="eventListeners">
        <list>
            <bean class="com.arlen.activiti.demo.event.listener.EnventListenerDemo" />
        </list>
    </property>
    <!-- 特定类型事件监听，在事件监听器后执行 -->
    <property name="typedEventListeners">
        <map>
            <!-- job执行成功、失败 -->
            <entry key="JOB_EXECUTION_SUCCESS,JOB_EXECUTION_FAILURE" >
                <list>
                    <bean class="com.arlen.activiti.demo.event.listener.EnventListenerDemo" />
                </list>
            </entry>
        </map>
    </property>
    <!-- 还可在运行时添加事件监听器，或者给特定的流程添加，给流程添加详见流程定义xml中 -->
 
    <!-- 限制哪些bean可以被用于process的表达式中 -->
    <!-- 如果传一个空map，则任何bean都看不到；如果不指定该参数，则所有bean均可用 -->
    <property name="beans">
        <map>
            <!-- <entry key="" value-ref="" /> -->
        </map>
    </property>
 
    <property name="mailServerHost" value="mail.my-corp.com" />
    <property name="mailServerPort" value="5025" />
 
  </bean>
 
  <bean id="processEngine" class="org.activiti.spring.ProcessEngineFactoryBean">
    <property name="processEngineConfiguration" ref="processEngineConfiguration" />
  </bean>
 
  <bean id="activitiRule" class="org.activiti.engine.test.ActivitiRule">
    <property name="processEngine" ref="processEngine" />
  </bean>
 
  <bean id="repositoryService" factory-bean="processEngine" factory-method="getRepositoryService" />
  <bean id="runtimeService" factory-bean="processEngine" factory-method="getRuntimeService" />
  <bean id="taskService" factory-bean="processEngine" factory-method="getTaskService" />
  <bean id="historyService" factory-bean="processEngine" factory-method="getHistoryService" />
  <bean id="managementService" factory-bean="processEngine" factory-method="getManagementService" />
  <bean id="identityService" factory-bean="processEngine" factory-method="getIdentityService" />
 
</beans>
```

### 设计模式
所有的操作都用CMD即命令设计模式
一个命令的日志如下
```java
DEBUG o.a.e.i.interceptor.LogInterceptor:34 - --- starting GetProcessDefinitionInfoCmd --------------------------------------------------------
DEBUG o.a.s.SpringTransactionInterceptor:40 - Running command with propagation REQUIRED
DEBUG o.s.j.d.DataSourceTransactionManager:476 - Participating in existing transaction
DEBUG o.a.e.i.i.CommandContextInterceptor:48 - Valid context found. Reusing it for the current command 'org.activiti.engine.impl.cmd.GetProcessDefinitionInfoCmd'
DEBUG o.a.e.i.interceptor.LogInterceptor:40 - --- GetProcessDefinitionInfoCmd finished --------------------------------------------------------
DEBUG o.a.e.i.interceptor.LogInterceptor:41 - 
```

### 问题收集
如果报错 `SAXParseException; lineNumber: 933; columnNumber: 53; Element type "bind" must be declared.`，因为Mybatis的版本太老了，因activiti-engine中已经依赖了mybatis:3.0.0，如果你项目中有单独引，因优先原则，会优先用你项目中的，所以升级项目中的mybatis至mybatis:3.0.0即可，或者自己项目中别依赖。