---
layout: post
title: Spring 事务 —— 嵌套调用
tags: Spring Transaction
source: virgin
---

**引语**： Spring声明式事务，将业务代码和事务分离，让我们开发人员专注于业务代码开发，带来了极大的便利，然而，有时候不知道其实现原理，随便乱用会导致各种各样的问题。

懒得翻代码，做了如下测试，来反推结论：
## 同一个类内部嵌套调用
1. **测试条件**：

   类无事务；类内override public方法A不加注解；override public方法B加Transactional注解。

   **测试过程**：

   通过接口调用A，再由A去调B

   **测试结果**：

   通过接口调用A，有代理类

   A内操作数据库，没有事务

   A内调用B，没有代理类，B内操作数据库，没有事务

2. **测试条件**：

   类无事务；类内override public方法A加Transactional注解；override public方法B加Transactional注解。

   **测试过程**：

   通过接口调用A，再由A去调B

   **测试结果**：

   通过接口调用A，有代理类

   A内操作数据库，有事务

   A内调用B，没有代理类，B内操作数据库，有事务，事务从当前事务（A所创建）获取


3. **测试条件**：

   类无事务；类内override public方法A加Transactional注解；override public方法B不加注解。

   **测试过程**：

   通过接口调用A，再由A去调B

   **测试结果**：

   通过接口调用A，有代理类

   A内操作数据库，有事务

   A内调用B，没有代理类，B内操作数据库，有事务，事务从当前事务（A所创建）获取

4. **测试条件**：

   类无事务；类内override public方法A不加注解；override public方法B不加注解。

   **测试过程**：

   通过接口调用A，再由A去调B

   **测试结果**：

   通过接口调用A，无代理类

   A内操作数据库，无事务

   A内调用B，没有代理类，B内操作数据库，无事务

5. **测试条件**：

   类无事务；类内override public方法A加Transactional注解；override public方法B加Transactional(propagation = Propagation.NOT_SUPPORTED)注解。
   **测试过程**：

   通过接口调用A，再由A去调B

   **测试结果**：

   通过接口调用A，有代理类

   A内操作数据库，有事务

   A内调用B，没有代理类，B内操作数据库，有事务，事务从当前事务（A所创建）获取

   B抛出异常，整个事务（A所创建）回滚

6. **测试条件**：

   类有事务Transactional；类内override public方法A加Transactional注解；override public方法B加Transactional(propagation = Propagation.NOT_SUPPORTED)注解。

   **测试过程**：

   通过接口调用A，再由A去调B

   **测试结果**：

   通过接口调用A，有代理类

   A内操作数据库，有事务

   A内调用B，没有代理类，B内操作数据库，有事务，事务从当前事务（A所创建）获取

   B抛出异常，整个事务（A所创建）回滚

   **结论**：也就是说propagation = Propagation.NOT_SUPPORTED没有什么卵用！**只是内部调用不管啥事务都没卵用，都用的是父级方法的；**

    **结论**：不管是类上加事务，还是方法上加事务，外部调用该类时，均会使用代理来执行；类内调用，不管方法上有无加事务，均不会起作用，因为内部调用不会产生任何代理类，是直接执行；所以平时的代码中，尤其是非public方法加注解都是一厢情愿！

## 多个类通过服务嵌套调用

1. **测试条件**：

   类C1无事务；类C1内override public方法C不加注解；类C2有事务；类C2内override public方法D不加注解

   **测试过程**：

   通过接口调用C，再由C去调D

   **测试结果**：

   通过接口调用C，无代理类

   C内操作数据库，无事务

   C内调用D，有代理类

   D内操作数据库，有事务，新建事务

2. **测试条件**：

   类C1无事务；类C1内override public方法C不加注解；类C2无事务；类C2内override public方法D加Transactional注解

   **测试过程**：

   通过接口调用C，再由C去调D

   **测试结果**：

   通过接口调用C，无代理类

   C内操作数据库，无事务

   C内调用D，有代理类，刚一进D方法，就会创建事务，然后等到有数据库操作，创建资源（session等）

   D内再次操作数据库，有事务，从当前事务拿资源（session等）

3. **测试条件**：

   类C1有事务；类C1内override public方法C不加注解；类C2无事务；类C2内override public方法D不加注解

   **测试过程**：

   通过接口调用C，再由C去调D

   **测试结果**：

   通过接口调用C，无代理类

   C内操作数据库，无事务

   C内调用D，无代理类

   D内操作数据库，有事务，从当前事务拿资源（session等）

4. **测试条件**：

   类C1有事务；类C1内override public方法C不加注解；类C2无事务；类C2内override public方法D加Transactional(propagation = Propagation.NOT_SUPPORTED)注解

   **测试过程**：

   通过接口调用C，再由C去调D

   **测试结果**：

   通过接口调用C，有代理类

   C内操作数据库，有事务

   C内调用D，有代理类，刚一进D方法，挂起C所创建事务，创建新sqlSession，并在Spring的TransactionSynchronizationManager中注册该资源，无事务

   D内再次操作数据库，无事务，还用上步创建的sqlSession来执行

   D方法跳出时，关闭sqlSession，返还数据库连接

   D方法跳出后，恢复C事务，提交C事务

5. **测试条件**：

   类C1有事务；类C1内override public方法C不加注解；类C2无事务；类C2内override public方法D加Transactional(propagation = Propagation.REQUIRES_NEW)注解

   **测试过程**：

   通过接口调用C，再由C去调D

   **测试结果**：

   通过接口调用C，有代理类

   C内操作数据库，有事务

   C内调用D，有代理类，刚一进D方法，挂起C所创建事务，重新获取数据库连接，创建新事务，创建新sqlSession，并在Spring的TransactionSynchronizationManager中注册该资源，无事务

   D内再次操作数据库，无事务，还用上步创建的sqlSession来执行

   D方法跳出时，提交D事务

   D方法跳出后，恢复C事务，提交C事务

    **结论**：所有事务调用都是针对外部调用的，内部调用均是没有代理类的，也即没有任何事务处理！

    但是外部调用的话意味着本来内部的方法得下移一层，新建一个类；也有个不好的解决方案，就是本来引用本类的bean，然后由该bean调用本类的方法


### 关于NOT_SUPPORTED、REQUIRES_NEW测试日志如下
    没有看详细的日志，以为两个结果一样，以为NOT_SUPPORTED也是有事务的结果又是自己年轻了。请看下边日志！

***---------------PROPAGATION_REQUIRED------------- 开始***
Creating new transaction with name [com.xx.oss.order.business.impl.TransactionTestBusinessImpl.otherTransServiceCall]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT; ''
Acquired Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] for JDBC transaction
Switching JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] to manual commit
Creating a new SqlSession
Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] will be managed by Spring
ooo Using Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java]
==>  Preparing: select * from order_info WHERE order_number = ? and del_flag = 0 
==> Parameters: ORD2016051315333433342051714(String)
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
Returning cached instance of singleton bean 'transactionManager'
**Suspending current transaction**

**-- 进入updateOrderInfo方法**
**Creating a new SqlSession**
***-- 这里就是迷惑我的地方，其实这并未创建事务，仅仅是从Spring的TransactionSynchronizationManager中注册一下刚刚新建的资源sqlSession***
***Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@825e33]***
Fetching JDBC Connection from DataSource
Registering transaction synchronization for JDBC Connection
JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] will be managed by Spring
ooo Using Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java]
==>  Preparing: update order_info SET add_time = ?, update_time = ?, del_flag = ? where id = ? 
==> Parameters: 2016-05-19 11:50:54.0(Timestamp), 2016-05-19 12:49:14.134(Timestamp), 0(Byte), 1545(Integer)
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@825e33]
Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@825e33]
Returning JDBC Connection to DataSource

**Resuming suspended transaction after completion of inner transaction**
Initiating transaction commit
Committing JDBC transaction on Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java]
Transaction synchronization committing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
Releasing JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] after transaction
Returning JDBC Connection to DataSource



***---------------REQUIRES_NEW开始  ---------------***
Creating new transaction with name [com.xx.oss.order.business.impl.TransactionTestBusinessImpl.otherTransServiceCall]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT; ''
Acquired Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] for JDBC transaction
Switching JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] to manual commit
Creating a new SqlSession
Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java] will be managed by Spring
ooo Using Connection [jdbc:mysql://127.0.0.1:3306/oss?someparam, UserName=root@localhost, MySQL Connector Java]
==>  Preparing: select * from order_info WHERE order_number = ? and del_flag = 0 
==> Parameters: ORD2016051315333433342051714(String)
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
Returning cached instance of singleton bean 'transactionManager'
**Suspending current transaction**

***-- 进入updateOrderInfo方法，在上边的日志中，没有再去获取新链接，直接从当前链接中创建sqlSession的***
**Acquired Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java] for JDBC transaction
Switching JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java] to manual commit**
Creating a new SqlSession
Registering transaction synchronization for SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@741f396c]
JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java] will be managed by Spring
ooo Using Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java]
==>  Preparing: update add_time = ?, update_time = ?, del_flag = ? where id = ? 
==> Parameters: 2016-05-19 12:49:53.0(Timestamp), 2016-05-19 13:16:38.693(Timestamp), 0(Byte), 1545(Integer)
Releasing transactional SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@741f396c]
***-- 在上边的日志中，这边直接close sqlSession，并回收数据库连接***
**Initiating transaction commit
Committing JDBC transaction on Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java]
Transaction synchronization committing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@741f396c]
Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@741f396c]
Releasing JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java] after transaction
Returning JDBC Connection to DataSource**

Resuming suspended transaction after completion of inner transaction
Initiating transaction commit
Committing JDBC transaction on Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java]
Transaction synchronization committing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
Transaction synchronization closing SqlSession [org.apache.ibatis.session.defaults.DefaultSqlSession@602adfd8]
Releasing JDBC Connection [jdbc:mysql://127.0.0.1:3306/oss?somparam, UserName=root@localhost, MySQL Connector Java] after transaction
Returning JDBC Connection to DataSource