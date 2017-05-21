---
layout: post
title: MySql Innodb 事务相关参数
tags: MySql
source: virgin
---


```sql
-- 显示加Innodb锁的两种方式
select ... lock in share mode：加 S 锁 
select ... for update：加 X 锁

mysql> select @@global.tx_isolation;
+-----------------------+
| @@global.tx_isolation |
+-----------------------+
| REPEATABLE-READ       |
+-----------------------+
--设定事务隔离级别

mysql> select @@global.autocommit;
+---------------------+
| @@global.autocommit |
+---------------------+
|                   1 |
+---------------------+
--自动提交


mysql> select @@global.innodb_table_locks;
+-----------------------------+
| @@global.innodb_table_locks |
+-----------------------------+
|                           1 |
+-----------------------------+
--InnoDB 内部锁定一个表

mysql> select @@global.innodb_lock_wait_timeout;
+-----------------------------------+
| @@global.innodb_lock_wait_timeout |
+-----------------------------------+
|                                50 |
+-----------------------------------+
--控制等待时间


mysql> select @@global.innodb_locks_unsafe_for_binlog;
+-----------------------------------------+
| @@global.innodb_locks_unsafe_for_binlog |
+-----------------------------------------+
|                                       0 |
+-----------------------------------------+
--insert into ...select 是否锁定源表
```
