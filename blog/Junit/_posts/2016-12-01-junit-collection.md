---
layout: post
title: Junit Spring整合问题收集
tags: JUnit
source: virgin
---

**注**： 内容会慢慢添加

1. 测试方法上加了注解`@Transactional`之后，**Junit默认为了不让测试数据污染数据库，从而会将所有的操作回滚**。不同于Activiti自己支持的Junit，再没有`@Transactional`注解的情况下，也会删除相关数据

