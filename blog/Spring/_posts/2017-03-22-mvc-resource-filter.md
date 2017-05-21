---
layout: post
title: DispatcherServlet 资源过滤那些事
tags: Spring AOP
source: virgin
---

    有了SpringMVC之后，Servlet那一套似乎不要我们操心了，然而当你在没有踩过一些坑之前，你是不能很好的伸展拳脚的。

## 1 DispatcherServlet配置之url-pattern

```xml
<servlet-mapping>
    <servlet-name>http</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>
```
1. *.do，匹配任何以.do结尾的请求；
2. /\*，匹配任何`host/context`下的请求，包括**\*.jsp**，*.jgp等一切的一切
3. / ，匹配任何`host/context`下**除了\*.jsp**的所有请求。

1就不用说了，2/3里边的坑真的很多很多。

### 坑1，资源的过滤
当你配置了2或3，就意味着你的所有静态资源请求都会经过DispatcherServlet处理，那肯定意味着404，找不到对应的Handler

#### 解决方式一：配置default servlet
```xml
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>*.jpg</url-pattern>
</servlet-mapping>
<!-- ... -->
```
可以在web.xml中配置一堆的default的servlet-mapping，但是这么一堆，看着爽么，我们还是用Spring来解决吧。

### 解决方式二：配置Resources
配置`Resources`。一般在dispatcher-servlet中配置
```xml
<mvc:resources mapping="/common/**" location="/common/" cache-period="3600" />
<mvc:resources mapping="/static/**" location="/static/" cache-period="3600" />
```
配置了resources之后，凡是mapping可以匹配的资源，DispatcherServlet找不到Handler之后，便会去资源里找，找到并返回。
**注意**：location，一定是一个完整路径，而不是通配符，千万不要配置成`location="/common/**"`，不知道谁的这种写法，花费了我5个小时找原因！

### 坑2，*.jsp的过滤
当你配置了2，也即 /*。假如我们有如下代码
dispatcher-servlet.xml
```xml
<bean id="internalResourceViewResolver" class="org.springframework.web.servlet.view.InternalResourceViewResolver">
   <property name="prefix" value="/WEB-INF/" />
   <property name="suffix" value=".jsp" />
</bean>
```
ProductController .java
```java
@Controller
@RequestMapping("/product")
public class ProductController {
    @Resource
    private SqlSessionFactoryBean bean;
 
    @RequestMapping("/test")
    public ModelAndView test() {
        return new ModelAndView("1");
    }
}
```
DispatcherServlet收到请求[/product/test]后，找到了Handler处理，处理结束后，返回1，InternalResourceViewResolver处理1为 [/WEB-INF/1.jsp]资源，然后，DispatcherServlet接着Forward该资源。因为/*可以拦截一切，该请求又被DispatcherServlet再次拦截，从而找不到Handler，从而报404.

#### 解决方式一：配置jsp servlet
web.xml
```xml
<servlet-mapping>
    <servlet-name>jsp</servlet-name>
    <url-pattern>*.jsp</url-pattern>
</servlet-mapping>
```
但是我们有Spring，为何还要麻烦servlet

#### 解决方式二：配置default-servlet-handler
dispatcher-servlet.xml配置中增加一行
```xml
<mvc:default-servlet-handler />
```
这样可以返回1.jsp了，但是是以纯文本的方式返回的，也即你在浏览器中看到的是1.jsp的源码，而不是渲染后的html

#### 解决方法三：更改url-pattern
可以换成1那种形式，比如*.do，但是这样的话，对WEB-INF中的资源却无能为力，所以最好的就是换成3，**/**。拦截除了jsp的请求，资源用resources过滤。
