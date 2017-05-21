---
layout: post
title: 深入浅出Spring MVC，玩转参数转换
tags: Spring AOP
source: virgin
---

    SpringMVC用了几年，一直停留在会用层面，annotation-driver的配置一直都是上网搜或者抄老项目的，前台传JSON进来后台如何接收，后台传Date类型，如何格式化给前台？为什么要配置Filter，为什么要配置拦截器，ArgumentResolver和HttpMessageConverter啥区别？如何便捷的校验前台参数？这些东西，全部是SpringMVC做的，不知道的话很难施展拳脚。

## 1 先从顶层看一下SpringMVC处理HttpRequest的过程
下图是我初步整理的一个流程图，里边的关键词都是平时能看到的。

![Criteria结构图]({{site.url}}/assets/img-blog/Spring/springmvc-process.png)

 
### 1.1 在这之前，先回顾下SpringMVC如何在项目中启动的
1. Web容器启动-》JVM启动-》Servlet容器启动-》解析war包-》解析web.xml-》ServletContext初始化
2. Spring ContextLoaderListener监听，当ServletContext初始化时，加载`contextConfigLocation`配置文件，初始化Spring IOC容器，初始化ApplicationContext。初始化后，其作为父级公共ApplicationContext供WebApplicationContext使用。
3. 创建Filter。
4. 创建DispatcherServlet，执行其父类FrameworkServlet的init方法。
5. init方法基于父Context——ApplicationContext，读取Servlet参数`contextConfigLocation`->`dispatcher-servlet.xml`，初始化WebApplicationContext，包括`annotation-driver`所声明的`RequestMappingHandlerAdapter`（装载默认的ArgumentResolver、MessageConverter、ConversionService等），还有`handlerMappings`。
6. init方法初始化DispatcherServlet，DispatcherServlet的`initStrategies`方法，从WebApplicationContext中初始化其自身属性，包含`RequestMappingHandlerAdapter`，并将ApplicationContext、WebApplicationContext放入HttpRequest属性中，供后面用。
7. 至此，DispatcherServlet加载完成，SpringMVC也加载完成。

### 1.2 SpringMVC处理HttpRequest的过程
依照上述流程图的文字描述，以HttpServletRequest做为引子，来描述处理过程。
以下描述不尽准确，因为涉及到的继承类太多，我只是将我觉得比较合适的名称放出来，用它来代表一个继承体系。另外，我的关注点在过滤器、拦截器、转换器等对参数的处理上，所以，其他地方都没有深究，描述有误的地方见谅
1. 浏览器发起Http请求，到达Web容器也即Servlet容器，Servlet容器为请求创建一个具体的HttpServletRequest（不同容器有不同实现）。
2. HttpServletRequest在到达具体的Servlet前，有Filter对其做了过滤。过滤过程中可以**扩展自己的HttpServletRequest**，也即HttpServletRequestWrapper，其中可能做了一些处理，比如对参数解码，比如做日志记录。或者扩展自己的HttpServletResponse，也即HttpServletResponseWrapper
3. HttpServletRequest请求到达具体的Servlet——DispatcherServlet，其对HttpServletRequest做实际的处理。
4. DispatcherServlet先调用拦截器HandlerInterceptor，做一些业务上的事情，比如登录拦截（注意，**拦截器不能改变HttpServletRequest**）。
5. DispatcherServlet从`handlerMappings`中获取合适的`HandlerExecutionChain`，拿到HandlerMethod，然后调用RequestMappingHandlerAdapter。
6. RequestMappingHandlerAdapter将HttpServletRequest包装为ServletWebRequest(NativeWebRequest)。
7. RequestMappingHandlerAdapter用属性webBindingInitializer(包含ConversionService)封装WebDataBinderFactory
8. RequestMappingHandlerAdapter将HandlerMethod、WebDataBinderFactory封装为ModelFactory，并对特殊的注解或Model类型的参数做处理。（此处没深究）
9. RequestMappingHandlerAdapter将HandlerMethod、WebDataBinderFactory、默认装配的ArgumentResolver和ReturnValueHandler封装为InvocableHandlerMethod
10. InvocableHandlerMethod使用ArgumentResolver解析参数，ArgumentResolver使用HttpMessageConverter或者ConversionService(含Spring-core中的一堆默认Converter)对解析后的参数做转换
11. InvocableHandlerMethod去触发HandlerMethod，也即反射的方式调用了Controller中的方法
12. InvocableHandlerMethod使用ReturnValueHandler处理返回值，如果有ResponseBody注解，会利用HttpMessageConverter做返回值转换，如果没有则对MVCContainer做一定的处理（将返回值放入mvcContainer），返回到RequestMappingHandlerAdapter
13. RequestMappingHandlerAdapter将mavContainer、modelFactory、webRequest封装为ModelAndView，返回到DispatcherServlet。
13. DispatcherServlet利用RequestToViewNameTranslator，根据HttpServletRequest，查找具体的ViewPath，给到ModelAndView。
14. DispatcherServlet调用拦截器HandlerInterceptor，做一些业务处理，比如记录日志
15. DispatcherServlet处理返回结果，如果是正常的ModelAndView，调用配置的`AbstractView`子类渲染View，然后获取`RequestDispatcher`，将view include至Response中，或forward到新的url。
16. 在HttpServletResponse返回前，Filter拦截，可以对其进行包装，返回HttpServletRequestWrapper。
17. 返回到Web容器。


## 2 关键点解析
上边过程中提到的类，下边基本都会一一解析。
### 2.1 Web容器部分
#### 2.1.1 ServletRequest、ServletResponse接口
ServletRequest，持有客户端的请求信息，Servlet容器会创建一个`ServletRequest`对象，并将其作为参数传给`Servlet.service`方法。`ServletRequest`持有的信息有` parameter name and values, attributes, and an input stream`，实现该接口的类可以提供协议相关的其他信息，来丰富扩充`ServletRequest`。
ServletResponse，用来帮助Servlet发送返回（Response）给客户端。当请求到达时，Servlet容器会创建一个`ServletResponse`对象，并将其作为参数传给`Servlet.service`方法。
#### 2.1.2 HttpServletRequest、HttpServletResponse接口
HttpServletRequest，继承了`ServletRequest`接口，提供HTTP请求的信息给Servlet。Servlet容器会创建一个`HttpServletRequest`对象，并将其作为参数传给`Servlet.service`方法。不同的Servlet容器，有不同的实现，比如Tomcat的实现为`org.apache.catalina.connector.RequestFacade`。另外，SpringMVC为了方便使用，定义了WebRequest接口，其实现了HttpServletRequest接口。
HttpServletResponse，继承了`ServletResponse`接口，专门处理HTTP请求的返回，比如它有获取HTTP header 和 cookie values的方法。跟上边一样，它也是Servlet容器创建的，然后传递给`Servlet.service`方法。
#### 2.1.3 ServletRequestWrapper、ServletResponseWrapper接口
ServletRequestWrapper继承了`ServletRequest`接口，即`ServletRequest`的包装或装饰器模式，供开发者去增加或修改合适请求信息给Servlet。
ServletResponseWrapper继承了`ServletResponse`接口，即`ServletResponse`的包装或装饰器模式，供开发者去增加或修改合适的返回信息给客户端。
#### 2.1.4 HttpServletRequestWrapper、HttpServletResponseWrapper
分别继承了`HttpServletRequest`和`HttpServletResponse`接口，即`HttpServletRequest`和`HttpServletResponse`的包装或装饰器模式，供开发者去增加或修改合适的请求和返回信息给Servlet。这里经常会写自己的HttpServletRequestWrapper，处理跨域请求，拦截sql注入，或者，将前台请求的JSON InputStream，转为HttpServletRequest的ParameterMap。
### 2.1.5 Filter
HttpServletRequest到达具体的Servlet前，Filter可以对其进行拦截，并且可以改变HttpServletRequest。比如上边自定义的HttpServletRequestWrapper，然后将其传入下一个Filter，最终到达Servlet，比如下面这样
```java
Map<String, String> paramMap = ServletUtil.getHttpJsonParamMap(request, base64);
JsonHttpServletRequestWrapper decodingRequest = new JsonHttpServletRequestWrapper(request, paramMap);
chain.doFilter(decodingRequest, response);
```

### 2.2 SpringMVC部分
因DispatcherServlet是SpringMVC的入口，也放在这边介绍，此章是重点。

#### 2.2.1 DispatcherServlet
**配置**：
DispatcherServlet是拦截所有http请求的Servlet，在web.xml中配置
```xml
<servlet>
    <servlet-name>http</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/config/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```
继承关系如下：
```java
DispatcherServlet extends FrameworkServlet
FrameworkServlet extends HttpServletBean
```
**工作原理**：
0. 其类的静态代码块中，读取了默认的Strategy的配置文件`DispatcherServlet.properties`，其中配置了DispatcherServlet中所依赖的所有默认实现类；
1. 其父类`HttpServletBean`上一个标准的Servlet，其init方法中，调用了`FrameworkServlet .initWebApplicationContext()`方法；
2. 它读取ServletContex中的`contextConfigLocation`且基于根ApplicationContext来初始化WebApplicationContext；
3. 然后调用了`DispatcherServlet.onRefresh()`方法，其调用了`DispatcherServlet.initStrategies()`方法，`initStrategies`方法建议细看，里边所有的初始化都在这里做，包括HandlerMethodMapping；
4. 该方法对`DispatcherServlet`中的各种Spring的Bean（DispatcherServlet中的属性字段）做了初始化。可以看到`DispatcherServlet`中有一堆BeanName常量，供从ApplicationContext中获取bean，可以查看源码，如果指定的名字我们自己定义了bean，则返回，如果没有，则返回spring自己初始化的默认Bean。如果想扩展这些Bean，只要我们在Spring配置文件中定义了这些Bean，就会被加载进去。
5. 当Http请求到达MVC，Servlet的doGet、doPost等方法，全部调用了`DispatcherServlet.doService()`方法，doService方法将Spring的上下文信息塞入了HttpRequest对象中，然后doService方法调用了`DispatcherServlet.doDispatch()`，这样`doDispatch`和`doService`方法中便可以使用ApplicationContext中的Bean了。然后，在`DispatcherServlet.doDispatch()`方法中，调用了`HandlerAdapter.handle()`方法。目前HandlerAdapter的实现类有五个，处理SpringMVC请求的就`AnnotationMethodHandlerAdapter`（已过期）和`RequestMappingHandlerAdapter`。

**自定义扩展**

DispatcherServlet有如下Bean属性，针对需求可以定义自己的对应的Bean，Spring会将其加载进去。
```java
    private MultipartResolver multipartResolver;
    private LocaleResolver localeResolver;
    private ThemeResolver themeResolver;
    /** List of HandlerMappings used by this servlet，HandlerMapping中放了HandlerExecutionChain，总共4种Handler，HttpRequestHandler、Servlet、Controller、HandlerMethod
    */
    private List<HandlerMapping> handlerMappings;
     /** List of HandlerAdapters used by this servlet，总共有5种，SpringMVC用的RequestMappingHandlerAdapter  */
    private List<HandlerAdapter> handlerAdapters;
    private List<HandlerExceptionResolver> handlerExceptionResolvers;
    private RequestToViewNameTranslator viewNameTranslator;
    private FlashMapManager flashMapManager;
    private List<ViewResolver> viewResolvers;
```

#### 2.2.2 RequestMappingHandlerAdapter
**配置**：

可以使用annotation-driver配置，也可以直接配置RequestMappingHandlerAdapter Bean，配置之后DispatcherServlet便可以从WebApplicationContext中拿到该Bean。

通过查看XSD，发现在4.1版本之前，`annotation-driven`元素是关联到了`AnnotationMethodHandlerAdapter`类，而4.1之后，该类被`@Deprecated`标记位过期，而是用`RequestMappingHandlerAdapter `取而代之，所以为了使用后者，我们需要使用4.1之后的Schema文件`http://www.springframework.org/schema/mvc/spring-mvc-4.2.xsd`;

下面是简单的一个annotation-driven配置。
```xml
<!-- http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc-4.2.xsd -->
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager">
    <mvc:message-converters>
      <bean class="org.springframework.http.converter.ByteArrayHttpMessageConverter"/>
      <bean class="org.springframework.http.converter.json.MappingJacksonHttpMessageConverter"></bean>   
    </mvc:message-converters>  
</mvc:annotation-driven>
```
**继承关系**

```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
        implements BeanFactoryAware, InitializingBean
```
**调用过程**：

`RequestMappingHandlerAdapter.handleInternal()`->`RequestMappingHandlerAdapter.invokeHandleMethod()`
**工作原理**：

通过代码能看到DispatcherServlet，通过HandlerAdapter去执行http请求，而这个HandlerAdapter包括MessageConverter、ArgumentResolver等，都支持扩展，我们一般都会在Spring-mvc.xml文件中配置，比如：
1. 在spring中配置后，spring加载该bean时做了对应的初始化，在其构造方法中装载默认的HttpMessageConverter，属性中初始化ParameterNameDiscoverer。
2. 因`RequestMappingHandlerAdapter`实现了`InitializingBean`接口，在其`afterPropertiesSet()`方法中装配了Spring默认的和用户自定义的`ArgumentResolver`、`ReturnValueHandler`，并将这三者封装为其对应的组合类*Composite，赋值给类属性。比如`getDefaultArgumentResolvers()`方法用来初始化类属性`argumentResolvers`，其装配了一堆的Spring默认的ArgumentResolver，然后加载了开发者自己写的CustomArgumentResolver。
3. DispatcherServlet调用的`HandlerAdapter.handle()`方法，最终调用了`RequestMappingHandlerAdapter.invokeHandleMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod)`方法，。
4. `invokeHandleMethod()`方法，将HttpServletRequest封装为WebRequest（ServletNativeRequest）。
5. 根据属性`webBindingInitializer`的配置（含ConversionService、Validator），封装WebDataBinderFactory。（WebDataBinderFactory中组合了BindingInitializer，当创建DataBinder时，将BindingInitializer中的ConversionService、Validator等给到DataBinder）
6. 用WebDataBinderFactory和HandlerMethod封装ModelFactory。
7. 用WebDataBinderFactory、HandlerMethod、ArgumentResolverComposite、ReturnValueHandlerComposite、ParameterNameDiscoverer五个类，封装构造InvocableHandlerMethod。
8. 初始化ModelAndViewContainer，ModelFactory初始化Model。
9. 触发InvocableHandlerMethod.invokeAndHandle方法，做实际的HttpServletRequest处理
10. 封装ModelAndView，返回给DispatcherServlet。

**自定义扩展**：

RequestMappingHandlerAdapter 有如下类属性，我们均可以根据自己的需求自定义自己的扩展，在配置annotation-driver或RequestMappingHandlerAdapter  Bean时去指定。
```java
    private List<HandlerMethodArgumentResolver> customArgumentResolvers;
    private HandlerMethodArgumentResolverComposite argumentResolvers;
    private HandlerMethodArgumentResolverComposite initBinderArgumentResolvers;
    private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;
    private HandlerMethodReturnValueHandlerComposite returnValueHandlers;
    private List<ModelAndViewResolver> modelAndViewResolvers;
    private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();
    private List<HttpMessageConverter<?>> messageConverters;
    private WebBindingInitializer webBindingInitializer;
```

#### 2.2.3 ServletWebRequest
**继承关系**：

```java
ServletWebRequest extends ServletRequestAttributes implements NativeWebRequest
NativeWebRequest extends WebRequest
```
**说明**：

做了一层封装，为了Spring框架的设计考虑，做了部分扩展。注意这里的HttpServletRequest是一个接口类，如果自己定义了filter，且指定了自定义的HttpServletRequestWrapper，那么此处的HttpServletRequest实现类即是你自己的HttpServletRequestWrapper。如果没有定义Filter和Wrapper，则其实现类是Web容器中的实现类，不同的Web容器有不同的实现，比如Tomcat的为`org.apache.catalina.connector.RequestFacade`

#### 2.2.4 HttpMethodArgumentResolver

**继承关系**

HttpMethodArgumentResolver是SpringMVC中一个庞大的体系，下面介绍几个有代表性的，具体参见另一篇文章。
```java
// HandlerMethodArgumentResolver的组合器模式，对外提供统一接口，内部调用合适的Resolver解析参数
HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver
// NamedValueResolver，对基本注解参数解析，然后调用ConversionService、Converter做参数转换，调用Validator做参数校验
AbstractNamedValueMethodArgumentResolver implements HandlerMethodArgumentResolver
// MessageConverterResolver，对@ResponseBody @RequestBody HttpEntity做处理，调用MessageConverter做转换
AbstractMessageConverterMethodArgumentResolver implements HandlerMethodArgumentResolver
// ServletRequestResolver，对特殊类型的参数，比如ServletRequest参数做处理，直接从上下文中取值赋值
ServletRequestMethodArgumentResolver implements HandlerMethodArgumentResolver
// MapMethodResolver，对参数类型为Map，且未指定参数名的参数做处理，直接从HttpServletRequest的ParameterMap中取
RequestParamMapMethodArgumentResolver implements HandlerMethodArgumentResolver
```
**找个壮丁**：

在其庞大的继承体系中`RequestResponseBodyMethodProcessor`占了大头，其同时实现了 `HandlerMethodReturnValueHandler`和`HandlerMethodArgumentResolver`接口，他也是NamedValueResolver家族中的一员，他对@RequestBody、@ResponseBody做处理，首先将HttpServletRequest转换为HttpInputMessage，然后交给HttpMessageConverter转换为Object，然后用Valid校验，最后用WebDataBinder将校验结果绑定到BindingResult中，并将结果加入到`ModelAndViewContainer`属性中。因其还实现了`HandlerMethodReturnValueHandler`接口，所以他还会处理返回值的类型转换工作。其工作是最复杂的，且处理了大部分SpringMVC请求。

**工作原理**：

1. 在`RequestMappingHandlerAdapter.afterPropertiesSet()->getDefaultArgumentResolvers()`方法中，装配了一堆Spring的和开发者自定义的ArgumentResolver，然后赋值给属性argumentResolvers`，其是各ArgumentResolver的组合器——HandlerMethodArgumentResolverComposite 
2. `argumentResolvers`用于封装`ServletInvocableHandlerMethod`和`ModelFactory`
    * ServletInvocableHandlerMethod：实际的SpringMVC处理，在处理前，通过ArgumentResolver的`support()`方法获取合适的ArgumentResolver类解析参数，不同的ArgumentResolver有不同的解析过程，见继承体系中的注释
    * ModelFactory modelFactory：在最终controller方法执行前，处理有@ControllerAdvice注解的Bean中，有@ModelAttribute和@SessionAttribute注解的方法。因为不是关注点，稍微看了下源码，是先从Attributes中取，如果取不到，则触发对应的MethodHandler，触发时使用了已装配的ArgumentResolver。然后将返回值放入Attributes中。

**自定义扩展**：

开发中经常需要转换前台的参数，可以自定义自己的实现，RequestMappingHandlerAdapter会将我们自定义的ArgumentResolver塞到customArgumentResolvers属性中，然后在初始化时，和Spring默认的一起加入到argumentResolvers中
```xml
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager">
        <argument-resolvers>
            <beans:bean
                class="com.arlen.common.web.argument.JsonMapperArgumentResolver" />
        </argument-resolvers>
</mvc:annotation-driven> 
```

#### 2.2.5 ReturnValueHandler

**工作原理**：

1. 在`RequestMappingHandlerAdapter.afterPropertiesSet()->getDefaultReturnValueHandlers()`方法中，装配了一堆Spring自带的和开发者自定义的ReturnValueHandler（包含了封装MessageConverter的Handler——RequestResponseBodyMethodProcessor），然后在赋值给属性`returnValueHandlers`，其是各ReturnValueHandler的组合器——HandlerMethodReturnValueHandlerComposite
2. `returnValueHandlers`用于封装ServletInvocableHandlerMethod，在`RequestMappingHandlerAdapter.createRequestMappingMethod`中，将`returnValueHandlers`给到HandlerMethod。
3. ServletInvocableHandlerMethod触发完`InvocableHandlerMethod.invokeForRequest()`方法后，用ReturnValueHandler对返回值做处理，如果方法有@ResponseBody注解，则调用HttpMessageConverter做实际的转换。

**自定义扩展**：

同上边的ArgumentResolver

#### 2.2.6 HttpMessageConverter

**工作原理**：

实则为ArgumentResolver、ReturnValueHandler的一个消息转换工具，只是针对@ResponseBody、@RequestBody、HttpEntity参数的特殊处理。
1. 在RequestMappingHandlerAdapter的构造方法中装配默认的HttpMessageConverter，总共有如下四种：StringHttpMessageConverter、ByteArrayHttpMessageConverter、SourceHttpMessageConverter、AllEncompassingFormHttpMessageConverter。
2. 在`RequestMappingHandlerAdapter.getDefaultArgumentResolvers()`中，将上边装配的`messageConverters`，包装为`RequestResponseBodyMethodProcessor`、`RequestPartMethodArgumentResolver`、`HttpEntityMethodProcessor`
3. 在`RequestMappingHandlerAdapter.getDefaultReturnValueHandlers()`中，将`messageConverters`包装为`HttpEntityMethodProcessor`、`RequestResponseBodyMethodProcessor`，与上边不同的是，装配时，指定了`ContentNegotiationManager`和`List<Object> responseBodyAdvice`参数。，也就是说ContentNegotiationManager使用来处理返回值用的。
4. 在ArgumentResolver、ReturnValueHandler做参数转换时，调用HttpMessageConverter做实际的转换。

**自定义扩展**：

HttpMessageConverter可以自定义实现，或覆盖修改现有的。比如下面这样，将StringHttpMessageConverter覆盖，为其制定参数，以UTF-8编码：
```xml
<mvc:annotation-driven content-negotiation-manager="contentNegotiationManager">
    <mvc:message-converters>
        <bean class="org.springframework.http.converter.StringHttpMessageConverter">
               <constructor-arg>
                   <value>UTF-8</value>
               </constructor-arg>
         </bean>
    </mvc:message-converters>  
</mvc:annotation-driven> 
```
***注意***：上边的配置都是覆盖RequestMappingHandlerAdapter构造方法中加载的MessageConverter，比如上边的配置，只为RequestMappingHandlerAdapter指定了一个MessageConverter，其他的都被你覆盖没啦。

#### 2.2.7 WebDataBinderFactory、DataBinder、WebBindingInitializer
WebDataBinderFactory有两个用途，一是，ModelFactory；二是，InvocableHandlerMethod。但是都是在ArgumentResolver解析参数，需要转换参数时使用。

**继承体系**：

```java
ServletRequestDataBinderFactory extends InitBinderDataBinderFactory
InitBinderDataBinderFactory extends DefaultDataBinderFactory
DefaultDataBinderFactory implements WebDataBinderFactory
```
**工作原理**：

1. 在SpringMVC启动时，会默认为RequestMappingHandlerAdapter加载一个WebBindingInitializer Bean。加载一个ConversionServiceFactoryBean工厂bean，产生（会加载所有的SpringCore中的Converter）一个ConversionService封装到WebBindingInitializer中。还有MessageCodesResolver、BindingErrorProcessor、Validator、PropertyEditorRegistrar都会封装到WebBindingInitializer中。（此处不尽准确，因翻看了SpringMVC的源码都没找到默认的Bean是在哪加载的，只看到了WebMvcConfigurationSupport方式配置时的加载方式，具体有兴趣可以去翻看Spring-IOC的源码）
2. 在`RequestMappingHandlerAdapter.invokeHandleMethod()`方法中，构建WebDataBinderFactory，将InitBinderMethod、WebBindingInitializer封装给WebDataBinderFactory；
3. 在ArgumentResolver（AbstractNamedValueMethodArgumentResolver）解析参数时，如果需要转换参数，则会调用WebDataBinderFactory创建一个WebDataBinder，然后调用WebBindingInitializer来初始化WebDataBinder。在1中我们也看到了，WebBindingInitializer中含有ConversionService等其他组件，初始化WebDataBinder时，都会将这些组件封装进去。（ConversionService、MessageCodesResolver、BindingErrorProcessor、Validator、PropertyEditorRegistrar）
4. 拿到WebDataBinder后，调用`WebDataBinder.convertIfNecessary()`进行参数转换，方法中，会将封装给WebDataBinder的ConversionService转换为`SimpleTypeConverter extends TypeConverterSupport`，然后调用`TypeConverterSupport.convertIfNecessary()->doConvert()`，最后通过TypeConverterDelegate，判断是调用PropertyEditorRegistrar，还是调用ConversionService做具体的转换。如果是ConversionService的话，转换是调用SpringCore中的Converter做的具体转换

**自定义扩展**：

可以自定义WebBindingInitializer扩展，可以指定自己定义的ConversionService、Validator、PropertyEditorRegistrar。因为annotation-driver不支持RequestMappingHandlerAdapterde的webBindingInitializer属性配置，所以必须单独配置RequestMappingHandlerAdapterde bean。
```xml
<bean class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
    <property name="webBindingInitializer">
            <bean class="com.arlen.common.web.MyWebBindingInitializer" />
    </property>
</bean>
```

#### 2.2.8 ConversionService、Converter、Validator
ConversionService是SpringCore中的接口，用来替代Java PropertyEditor来做类型转换的，在SpringMVC中也会用到。

上边知道了，ConversionService是在`RequestMappingHandlerAdapter->WebBindingInitializer`中的，但是Spring默认的ConversionService是如何加载进来的？查Spring文档，要自己定义一个ConversionService的话，必须用`ConversionServiceFactoryBean`，那么猜想Spring初始化默认的ConversionService是用的`ConversionServiceFactoryBean`，所以从`ConversionServiceFactoryBean`入手去找。

**工作原理**：

1. `ConversionServiceFactoryBean`继承了`InitializingBean`，在其`afterPropertiesSet()`方法中调用了自身的`createConversionService()`方法，该方法返回一个新的`DefaultConversionService`。
2. 在`DefaultConversionService`构造方法中，调用了`DefaultConversionService.addDefaultConverters()`，该方法中加载了所有Spring定义的Converter。也即DefaultConversionService拥有所有的Converter，同时DefaultConversionService也组合到每个Converter中。不同的Converter有不同的转换方法，但是最终都是实现了`GenericConverter`接口的`Object convert(Object source, TypeDescriptor sourceType, TypeDescriptor targetType)`方法。（还有实现`Converter`、`ConverterFactory`接口的，此处没深究）
3. WebDataBinder会调用ConversionService的`convert`方法做转换，在ConversionService的convert方法中，最终会调用`GenericConversionService.convert()`方法，用`ConversionUtils.invokeConverter()`方法实现`GenericConverter`的`convert`方法调用。
4. 如果Converter转换的过程中有其他复杂类型，则会继续调用ConversionService的`convert`方法（2中已经讲了，Converter也持有ConversionService）。

**自定义扩展**：

如果要用自定义的和Spring自己的默认Converter，则必须在Spring配置文件中声明`ConversionServiceFactoryBean`工厂Bean，Spring会用我们定义的工厂Bean覆盖默认的，进而加载我们定义的Converter。
也可以将自己声明的Bean给到RequestMappingHandlerAdapter的WebBindingInitializer。也即在Annotation-driver中配置`WebBindingInitializer`，并且指定`conversionService`属性为自己定义的`ConversionServiceFactoryBean`工厂Bean

```xml
<bean id="conversionService"
    class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
    <property name="converters">
        <list>
            <bean class="com.arlen.common.core.converter.CustomConverter" />
        </list>
    </property>
</bean>
```

***注意***：同HttpMessageConverter，如果配置了`ConversionServiceFactoryBean`工厂Bean，会覆盖默认加载的Converters，所以最好的办法是扩展ConversionServiceFactoryBean，定义自己的工厂Bean，不光可以改变默认Converters的顺序，还能加载自己想要的Converter

***附***： Formater也是基于ConversionService的，跟ConversionService一样的用法，具体查看Spring文档。

#### 2.2.9 ServletInvocableHandlerMethod
终于到了ServletInvocableHandlerMethod，最终的controller方法被反射调用，此为核心（写得累死了，还得干活。。。）

**继承体系**：

```java
ServletInvocableHandlerMethod extends InvocableHandlerMethod
InvocableHandlerMethod extends HandlerMethod
```

**工作原理**：

1. 在RequestMappingHandlerAdapter的方法`createRequestMappingMethod`中创建，封装了HandlerMethod、WebDataBinderFactory、ArgumentResolver、ReturnValueHandler、ParameterNameDiscoverer
2. `ServletInvocableHandlerMethod `，一切都是为它服务，包括：自身被包装对象HandlerMethod，参数解析器ArgumentResolvers，返回处理器ReturnValueHandlers，数据绑定工厂DataBinderFactory，参数名查找器ParameterNameDiscoverer。
3. RequestMappingHandlerAdapter调用`ServletInvocableHandlerMethod`->`invokeAndHandle()`方法，该方法调用了`InvocableHandlerMethod.invokeForRequest()`。
4. `InvocableHandlerMethod.invokeForRequest()`->`getMethodArgumentValues()`->`this.argumentResolvers.resolveArgument()`，该方法即调用了ArgumentResolver获取参数的值。
5. 获取参数后，调用了`doInvoke()`->`getBridgedMethod().invoke(getBean(), args)`，然后底层就不看啦，就是反射的方法调用啦。
6. 等返回后，`ServletInvocableHandlerMethod`->`invokeAndHandle()`继续处理返回值，调用了`this.returnValueHandlers.handleReturnValue()`。

#### 2.2.10 ContentNegotiationManager

实则为ArgumentResolver的一个返回值处理工具，只针对@ResponseBody注解方法和HttpEntity类型返回值方法。
1. 默认`contentNegotiationManager = new ContentNegotiationManager()`，然后用户自定义的会替换默认的
2. `RequestMappingHandlerAdapter.getDefaultReturnValueHandlers`方法中，封装成
    * HttpEntityMethodProcessor，`HttpEntityMethodProcessor extends AbstractMessageConverterMethodProcessor`，在其父类的`writeWithMessageConverters()`->`getAcceptableMediaTypes()`方法中，获取ContentNegotiationManager中所配置的接受的MediaType。父类的`writeWithMessageConverters()`由子类的`handleReturnValue()`方法调用
    * RequestResponseBodyMethodProcessor，`RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor`，同上，在其父类中获取可以接受的MediaType，同上也在子类的`handleReturnValue`方法中调用

#### 2.2.11 ModelAndViewContainer
略

#### 2.2.12 ParameterNameDiscoverer
略


**写在最后**：之前没有好好学习，自己写了自己的Validator框架，写了自己的Converter框架，这两天看完SpringMVC后，得考虑摈弃自己写的那些框架了。


## 3 源码中发现的设计模式
Spring:
`HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver`：组合器模式，首先组合器继承Resolver接口，组合器有多个ArgumentResolver，每个Resolver有自己的Support方法，然后组合器对外统一接口（即Resolver对外的接口），然后内部遍历取合适的Resolver，做相应的操作。

`HttpRequestHandlerAdapter implements HandlerAdapter`：适配器模式，将Spring封装的不同的Servlet处理器Handler（有各自不同的接口，比如HttpRequestHandler——handleRequest、Servlet——service、Controller(Spring的)——handleRequest），通过适配（在适配器中转换），对外统一开放同样的接口。**重点来了**：不同于其他Handler，Spring MVC中的`HandlerMethod`（即你@RequestMapping声明的方法），没有实际的Handler，而是只有自己实际的适配器`RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter`，之前一直纠结的MessageConverter、ArgumentResolver啥的是啥关系，都从这里找到了根源。其中的`handleInternal`方法是切入点。

`HttpServletRequestWrapper extends ServletRequestWrapper implements HttpServletRequest`：装饰模式，跟被装饰者继承相同的接口，然后对外调用时，做一定的处理，然后再调用被装饰者的方法。


## 4. 其他
### 4.1 WebMvcConfigurationSupport
之前都是基于XML配置Spring的，在阅读SpringMVC源码过程中发现了WebMvcConfigurationSupport。其是用于通过类方式来配置Spring的，只要自己的配置类继承该类即可。

### 4.2 spring-mvc-4.2.xsd
annotation-driven，可以配置很多东西，截了个图放这边，方便查询。

![Criteria结构图]({{site.url}}/assets/img-blog/Spring/springmvc-xsd.png)
