---
layout: post
title: Mybatis generator 生成的 Example 类详解
tags: Mybatis Plugin
source: virgin
---
    Mybatis generator 插件会生成一堆Example类，之前一直没有在意，直到最近写generator的插件时，才去探了一下究竟。然后发现，其用编程式实现SQL 的动态拼接，很是方便，比自己一直去在SqlMap中写sql方便太多了。

## 结构解析
生成的文件中，包含了Example类和SqlMap中的Example_Where_Clause。
### Example类
![Example相关类图]({{site.url}}/assets/img-blog/Mybatis/ExampleClass.png)

生成的Example类图如上。大体理解下，Example将其看成一个查询的Where条件集合（List<Criteria>），其依赖的Criteria类没有任何方法，是供我们自定义自己的方法的，该类的关键实现都在其父类GeneratedCriteria。GeneratedCriteria包含有一个Where条件中的子语句集合（List<Criterion>），比如 id=2；number like '%22%'。他所依赖的类Criterion就是Where子语句集的数据结构。
 
需要注意的地方：
1. Example类中的Criteri*的关系
   
   ![Criteria结构图]({{site.url}}/assets/img-blog/Mybatis/CreteriaStruct.png)

    一次查询只有一个oredCriteria，一个oredCriteria中含有多个Criteria，**多个Criteria之间全是用OR连接**。一个Criteria中含有多个Criterion，**多个Criterion之间用AND连接**。
2. Example类创建Criteria的方法
    * or() 和 or(Criteria)：均是为Example创建**多个**Criteria，前者是创建一个新的Criteria，后者是将已有的Criteria加入到现有的oredCriteria中；
    * createCriteria：为Example创建**单个**Criteria；
    * 注意点：or() 和 createCriteria() 如果是第一次调用，具有相同的功能，往oredCriteria中放入一个Criteria。如果是第二次或者多次调用，or()还是往oredCriteria中放入一个新的Criteria，而createCriteria()则是直接返回一个新的Criteria，不对oredCriteria做任何操作。
3. GeneratedCriteria类isValid方法
    isValid方法是供SqlMap文件中的WhereClause用的，其只是简单指定了Criteria中是否含有一个或多个Criterion。

### Example_Where_Clause
```xml
<sql id="Example_Where_Clause">
<where>
  <foreach collection="oredCriteria" item="criteria" separator="or">
	<if test="criteria.valid">
	  <trim prefix="(" prefixOverrides="and" suffix=")">
		<foreach collection="criteria.criteria" item="criterion">
		  <choose>
			<when test="criterion.noValue">
			  and ${criterion.condition}
			</when>
			<when test="criterion.singleValue">
			  and ${criterion.condition} #{criterion.value}
			</when>
			<when test="criterion.betweenValue">
			  and ${criterion.condition} #{criterion.value} and #{criterion.secondValue}
			</when>
			<when test="criterion.listValue">
			  and ${criterion.condition}
			  <foreach close=")" collection="criterion.value" item="listItem" open="(" separator=",">
				#{listItem}
			  </foreach>
			</when>
		  </choose>
		</foreach>
	  </trim>
	</if>
  </foreach>
</where>
</sql>
```
对照着上边的Example相关类解析，可以看出`oredCriteria`中多个Criteria中用OR分割，然后如果isValid（Criteria中有多个Criterion），则遍历Criterion，并且用And拼接。

## 几个例子
下面总共三个例子，基本能说明所有问题了。
```sql
SELECT * FROM order_extra_fee WHERE id > 3 AND id < 10 AND fee_code = 502;
SELECT * FROM order_extra_fee WHERE (id >3 AND id < 10) OR (add_time > '2017-02-10 18:42:27' AND add_time < '2017-02-12 18:42:27');
SELECT * FROM order_extra_fee WHERE ((id >3 AND id < 10) OR (add_time > '2017-02-10 18:42:27' AND add_time < '2017-02-12 18:42:27')) AND fee_amount > 0;
SELECT * FROM order_extra_fee WHERE ((id >3 AND id < 10 AND fee_amount > 0) OR (add_time > '2017-02-10 18:42:27' AND add_time < '2017-02-12 18:42:27')AND fee_amount > 0);
```
下面是java实现，注意我的Example类的命名都改了，改成Query了。
```java
OrderExtraFeeQuery query = new OrderExtraFeeQuery();
// 第一种
query.createCriteria().andIdGreaterThan(3).andIdLessThan(10);
 
// 第二种，or用来拼接另一个Criteria
query.or().andAddTimeGreaterThan(new Date(1486723347000l)).andAddTimeLessThan(new Date(1486896147000l));
 
// 第三种，没法实现啊，(￣︶￣)> ，换种思路，也即第四种
query.clear();
query.createCriteria().andIdGreaterThan(3).andIdLessThan(10).andFeeAmountGreaterThan(new BigDecimal("0"));
query.or().andAddTimeGreaterThan(new Date(1486723347000l)).andAddTimeLessThan(new Date(1486896147000l)).andFeeAmountGreaterThan(new BigDecimal("0"));
```

## 与前台页面统一封装
前台增删改查想必是每个偏业务程序员的痛，JQGrid插件功能还是很强大的，基本可以实现增删改查的所有功能，样式也可以选择多种的，那么通过支撑其的增删改查，考虑到我们前台参数统一封装也可以借鉴其思路。
