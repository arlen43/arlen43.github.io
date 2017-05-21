---
layout: post
title: No mapping found for HTTP request with URI [..]
tags: Exception
source: virgin
---

一场DispatcherServlet url-pattern引发的血案！`/` 和 `/*`到底有什么区别

## 日志：
```java
DEBUG o.s.web.servlet.DispatcherServlet:865 - DispatcherServlet with name 'http' processing GET request for [/product/product/test]
DEBUG o.s.w.s.m.m.a.RequestMappingHandlerMapping:310 - Looking up handler method for path /product/test
DEBUG o.s.w.s.m.m.a.RequestMappingHandlerMapping:317 - Returning handler method [public org.springframework.web.servlet.ModelAndView com.hongkun.product.controller.ProductTest.getDictIndex()]
DEBUG o.s.b.f.s.DefaultListableBeanFactory:251 - Returning cached instance of singleton bean 'productTest'
DEBUG o.s.web.servlet.DispatcherServlet:951 - Last-Modified value for [/product/product/test] is: -1
DEBUG o.s.web.servlet.DispatcherServlet:1251 - Rendering view [org.springframework.web.servlet.view.JstlView: name '1'; URL [/WEB-INF/1.jsp]] in DispatcherServlet with name 'http'
DEBUG o.s.web.servlet.view.JstlView:166 - Forwarding to resource [/WEB-INF/1.jsp] in InternalResourceView '1'
DEBUG o.s.web.servlet.DispatcherServlet:865 - DispatcherServlet with name 'http' processing GET request for [/product/WEB-INF/1.jsp]
DEBUG o.s.w.s.m.m.a.RequestMappingHandlerMapping:310 - Looking up handler method for path /WEB-INF/1.jsp
DEBUG o.s.w.s.m.m.a.RequestMappingHandlerMapping:320 - Did not find handler method for [/WEB-INF/1.jsp]
WARN  o.s.web.servlet.PageNotFound:1147 - No mapping found for HTTP request with URI [/product/WEB-INF/1.jsp] in DispatcherServlet with name 'http'
DEBUG o.s.web.servlet.DispatcherServlet:1000 - Successfully completed request
DEBUG o.s.web.servlet.DispatcherServlet:1000 - Successfully completed request
```

## 我的配置

我的web.xml配置

```xml
<servlet>
    <servlet-name>http</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:config/spring/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>http</servlet-name>
    <url-pattern>/*</url-pattern>
</servlet-mapping>
```

dispatcher-servlet.xml配置

```xml
<mvc:resources mapping="/common/**" location="/common/" cache-period="3600" />
<mvc:resources mapping="/static/**" location="/static/" cache-period="3600" />

<bean id="internalResourceViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
   <property name="prefix" value="/WEB-INF/" />
   <property name="suffix" value=".jsp" />
</bean>
```

java方法

```java
@Controller
@RequestMapping("/product")
public class ProductTest {
    @Resource
    private SqlSessionFactoryBean bean;
 
    @RequestMapping("/test")
    public ModelAndView getDictIndex() {
        return new ModelAndView("1");
    }
}
```

在WEB-INF中我有1.jsp

## 问题解决
根据上边的日志，`[/product/product/test]`经由DispatcherServlet处理完之后，返回1，然后经由InternalResourceViewResolver解析为` [/WEB-INF/1.jsp]`资源，然后Forward该资源。这个资源又被当做Servlet请求处理了，也即又经过了DispatcherServlet，又去找Handler了，结果没找到就报错了。

### 解决方法一
dispatcher-servlet.xml配置中增加一行
```xml
<mvc:default-servlet-handler />
```
其会对*.jsp的请求做处理，返回对应路径的值，但是不做任何渲染，是以纯文本的方式返回前台的，你在浏览器中看到的是jsp的源代码。

### 解决方法二
在web.xml中添加jsp Servlet配置
```xml
<servlet-mapping>
    <servlet-name>jsp</servlet-name>
    <url-pattern>*.jsp</url-pattern>
</servlet-mapping>
```
这样就是jsp请求交给了jsp Servlet处理了，没有给到Spring，然后包括各种资源等，都可以通过配置default servlet来处理
```xml
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.jpg</url-pattern>
</servlet-mapping>
<!-- ... -->
```
这样，web.xml中就会有一堆堆的Servlet配置，看着很烦。

### 解决方法三
DispatcherServlet配置修改，url-pattern随便改一下，比如只匹配*.do的文件
```xml
<servlet-mapping>
    <servlet-name>http</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>
```
但这样看着会很不爽

### 解决方法四
DispatcherServlet配置修改，url-pattern改成 /，而不是 /*
```xml
<servlet-mapping>
    <servlet-name>http</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>
```

/ 只匹配 `host/servlet`，*.jsp不会去匹配，其他的还是会匹配的

/* 匹配`host/servlet`之下的所有内容，包括*.jpg，*.html，更包含*.jsp。

这个看着是最清爽的一种解决方案。