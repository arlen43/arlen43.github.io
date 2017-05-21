---
layout: post
title: Spring MVC 集成 SLF4J+LogBack
tags: Spring LogBack
source: virgin
---

## 添加依赖
```xml
<dependency>
	<groupId>ch.qos.logback</groupId>
	<artifactId>logback-classic</artifactId>
	<version>1.1.3</version>
</dependency>
<dependency>
	<groupId>org.logback-extensions</groupId>
	<artifactId>logback-ext-spring</artifactId>
	<version>0.1.2</version>
</dependency>
<dependency>
	<groupId>org.slf4j</groupId>
	<artifactId>jcl-over-slf4j</artifactId>
	<version>1.7.12</version>
</dependency>
```
如上所示是集成所需要的依赖，其中：

第一个logback-classic包含了logback本身所需的slf4j-api.jar、logback-core.jar及logback-classsic.jar

第二个logback-ext-spring是由官方提供的对Spring的支持，它的作用就相当于log4j中的Log4jConfigListener；这个listener，网上大多都是用的自己实现的，原因在于这个插件似乎并没有出现在官方文档的显要位置导致大多数人并不知道它的存在

第三个**jcl-over-slf4j**是用来把Spring源代码中大量使用到的commons-logging替换成slf4j，只有在添加了这个依赖之后**才能看到Spring框架本身打印的日志**，否则只能看到开发者自己打印的日志

## 编写logback.xml
官网地址如下：
http://logback.qos.ch/translator/

logback的详细用法及其xml文件的相关语法，可参见它的用户向导，地址如下：
http://logback.qos.ch/manual/introduction.html

## 配置方式一——web.xml
```xml
<context-param>
         <param-name>logbackConfigLocation</param-name>
         <param-value>classpath:logback.xml</param-value>
</context-param>
<listener>
         <listener-class>ch.qos.logback.ext.spring.web.LogbackConfigListener</listener-class>
</listener>
```
与log4j类似，logback集成到Spring MVC项目中，也需要在web.xml中进行配置，同样也是配置一个config location和一个config listener，其中LogbackConfigListener由前述的logback-ext-spring依赖提供，若不依赖它则找不到这个listener类

优势：

* 配置简单

劣势：

* 当有多套环境时，因web.xml不在class目录下，无法用maven进行过滤，加载不同的日志配置文件
* 当用JUnit执行单元测试时，无法使用自己编写的logback.xml配置logback

## 配置方式二——spring-log.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:p="http://cxf.apache.org/policy"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd">
	<bean id="loggingInitialization"
		class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
		<property name="targetClass"
			value="com.*.LogbackIniter" />
		<property name="targetMethod" value="init" />
		<property name="arguments">
			<list>
				<value>classpath:config/log/logback_@{envName}.xml</value>
			</list>
		</property>
	</bean>
</beans>
```

需要用java写一个LogBack的初始构造器
```java
public class LogbackIniter {
    public static void init(String logbackConfigLocation) throws FileNotFoundException, JoranException {
        String resolvedLocation = SystemPropertyUtils.resolvePlaceholders(logbackConfigLocation);
        if (StringUtils.isEmpty(resolvedLocation) || !resolvedLocation.toLowerCase().endsWith(".xml")) {
            throw new IllegalArgumentException("Log config file is not valid.");
        }
 
        URL logbackConfUrl = ResourceUtils.getURL(resolvedLocation);
        LoggerContext loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory(); 
        loggerContext.reset();
        JoranConfigurator configurator = new JoranConfigurator(); 
        configurator.setContext(loggerContext); 
        configurator.doConfigure(logbackConfUrl);
        StatusPrinter.printInCaseOfErrorsOrWarnings(loggerContext);
    }
}
```

优势：

* 多环境支持，通过maven替换即可
* junit测试方便，可以讲日志打入文件，慢慢翻看

## 其他
使用slf4j-api的时候，需要注意的是：slf4j采用了单例模式，项目中创建的每一个Logger实例都会按你传入的name（传入的Class<?>实例也会被转换成String型的name）保存到一个静态的ConcurrentHashMap中；因此只要name（或Class<?>实例）相同，每次返回的实际上都是同一个Logger实例。因此完全没必要把Logger实例作为常量或静态成员，随用随取即可。实际上，其作者也不建议那么做

## reference
完全参照如下网页：http://blog.csdn.net/sadfishsc/article/details/47160213

