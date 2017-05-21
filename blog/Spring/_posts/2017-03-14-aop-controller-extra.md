---
layout: post
title: Spring-AOP拦截Controller引发的血案
tags: Spring AOP
source: virgin
---

    之前的文章也讲了，不要强制使用CGlib代理，只需要将其包加入到项目中，如果需要代理非接口实现的类，Spring会自动调用CGlib来代理。那么Controller中的方法应该也能拦截吧，然而又会遇到一些奇葩的问题。

## 1 我的栗子
我要在select方法前，将页面的查询参数，自动转换为Mybatis的Criteria，这样你们就再也不用各种复杂的查询写起来麻烦啦。转换的代码有点多，就忽略了哈。另外代码中有些工具类或方法，在我之前的文章中都有讲到过，不了解的可以翻我之前的文章，或者直接留言问我。

以下代码及配置，仅仅是我写一些示例性功能时写的，特别糙，大神喷前留情。

### Controller代码
```java
@Controller
@RequestMapping("/order")
public class OrderController {
 
    @Resource
    private IOrderExtraFeeDao dao;
 
    @RequestMapping(value="/select", method = RequestMethod.GET)
    // @ResponseJson、@RequestJson 大伙看成 @ResponseBody和@RequestBody就行啦，不要在意这些细节
    @ResponseJson
    public List<OrderExtraFee> select(@RequestJson PageSearchDto<OrderExtraFeeQuery> searchDto) {
        OrderExtraFeeQuery query = searchDto.getQuery();
        List<OrderExtraFee> list = dao.selectByExample(query);
        return list;
    }
}
```

### Aspect配置
```java
@Component
@Aspect
public class PageSerachAspect {
 
    private final static Logger logger = LoggerFactory.getLogger(DictionaryAspect.class);
 
    @Around(value = "execution(* com.hongkun..controller..*.*(..)) && args(com.hongkun.common.base.PageSearchDto)")
    public Object generateCriteriaQuery(ProceedingJoinPoint pjp) throws Throwable {
        Signature signature = pjp.getSignature();
        logger.info("Generate CriteriaQuery for join point {} -> {} start...", signature.getDeclaringTypeName(), signature.getName());
 
        boolean canGenerator = true;
        Type[] paramTypes = ((MethodSignature)signature).getMethod().getGenericParameterTypes();
        if (null == paramTypes || paramTypes.length <= 0) {
            logger.error("Generate CriteriaQuery for join point {} -> {}, but arguments type is empty", signature.getDeclaringTypeName(), signature.getName());
            canGenerator = false;
        }
        if (!ParameterizedType.class.isAssignableFrom(paramTypes[0].getClass())) {
            logger.error("Generate CriteriaQuery for join point {} -> {}, but arguments type is not parameterized", signature.getDeclaringTypeName(), signature.getName());
            canGenerator = false;
        }
        Type typeArgument = ((ParameterizedType)paramTypes[0]).getActualTypeArguments()[0];
 
        Object[] args = pjp.getArgs();
        if (null == args || args.length <= 0) {
            logger.warn("Generate CriteriaQuery for join point {} -> {}, but arguments is empty", signature.getDeclaringTypeName(), signature.getName());
            canGenerator = false;
        }
        if (!(args[0] instanceof PageSearchDto)) {
            logger.warn("Generate CriteriaQuery for join point {} -> {}, but arguments is not supported", signature.getDeclaringTypeName(), signature.getName());
            canGenerator = false;
        }
        if (canGenerator) {
            logger.info("Generate CriteriaQuery for join point {} -> {}", signature.getDeclaringTypeName(), signature.getName());
            PageSearchDto<?> pageSearch = (PageSearchDto<?>)args[0];
            pageSearch.toQuery(Type2Class.getArgumentOfType(typeArgument));
        }
 
        Object ret = pjp.proceed(args);
        logger.info("Generate CriteriaQuery for join point {} -> {} end...", signature.getDeclaringTypeName(), signature.getName());
        return ret;
    }
}
```
### 1.3 Spring配置
![spring示例配置]({{site.url}}/assets/img-blog/Spring/spring-config-example.png)

spring-web，其他配置略
```xml
<context:annotation-config />
<context:component-scan base-package="com.hongkun" />
```

spring-aop，其他配置略
```xml
<!-- 请勿强制使用CGlib代理，只需加入包即可 -->
<aop:aspectj-autoproxy proxy-target-class="false" />
```

dispatc-servlet，其他配置略
```xml
<!-- 开启注解 -->
<context:annotation-config />
<!-- 注解扫描基础包 -->
<context:component-scan base-package="com.hongkun" />
  
<mvc:annotation-driven>
    <mvc:argument-resolvers>
        <beans:bean class="com.arlen.common.web.argument.RequestJsonArgumentResolver" />
    </mvc:argument-resolvers>
    <mvc:return-value-handlers>
        <beans:bean class="com.arlen.common.web.returnvalue.Base64HandlerMehodReturnValueHandler" />
    </mvc:return-value-handlers>
</mvc:annotation-driven>
```

## 2 然后我们看下怎么用

### 2.1 用JUnit测试，没有任何问题
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = { "classpath*:config/spring/spring-*.xml" })
public class BaseTest {
}
```

```java
public class OrderExtraFeeTest extends BaseTest {
 
    @Resource
    private OrderController controller;
 
    @Test
    public void controller() {
        PageSearchDto<OrderExtraFeeQuery> search = new PageSearchDto<>();
        search.setSidx("id");
        search.setSort("desc");
        search.setSearchField("orderNumber");
        search.setSearchOper("ni");
        search.setSearchString("22,33,44,55");
        controller.select(search);
    }
}
```

### 2.2 用tomcat，页面请求，没有拦截到
PS：我写文章就是有个毛病，想以我解决问题的思维来引导大家，如何解决问题，而不是盲目的去百度。

首先，我们一般的Spring的配置会同时配置ContextLoaderListener和DispatcherServlet，对应会有两个不同ApplicationContext，Spring命名如下：Root-WebApplicationContext和http-servlet-WebApplicationContext

#### 2.2.1 猜想原因
**针对上边的配置**猜想原因：
1. 代理没加上？

    我们知道，AOP在为Bean添加代理是在ApplicationContext初始化的时候添加的，你通过BeanFactory获取的Bean就是已经代理过的Bean，用JUnit测试没问题就证明了Controller已经被代理了。结论是：在ApplicationContext中，Controller已经被代理了，并且生效了。

2. 代理加上了，但是对页面请求不生效？也即对DispatcherServlet不生效？

    我们也知道，在DispatcherServlet中，通过WebApplicationContext中加载的RequestMappingHandlerMapping中的HandlerMethod来反射执行Controller，而HandlerMethod也是从WebApplicationContext中已经被代理的Bean中拿到的，即HandlerMethod持有被代理的Bean和Bean的方法，当反射调用时就是调的已经代理过的Bean和方法。结论是：如果在WebApplicationContext中，Controller真被代理了，就肯定会生效。（解析见文末）

#### 2.2.2 问题答案
问题分析：经过分析上边的原因，Controller既然已经被代理了，对DispatcherServlet也生效，那是到底什么鬼？

问题的症结：我们代理的Controller在Root-WebApplicationContext中，而要在http-servlet-WebApplicationContext中生效。

症结诊断1： 常理是没问题的啊，Root-WebApplicationContext是根Context，按说http-servlet-WebApplicationContext中可以拿到任何Root-WebApplicationContext中的Bean才对。

症结诊断2： 

    * 看了之前的文章，我们知道，Spring在解析`annotation-driven`标签时，初始化了RequestMappingHandlerMapping Bean，初始化时会将所有的HandlerMethod扫描并赋给RequestMappingHandlerMapping Bean，翻看源码，发现，扫描方法只会扫描**当前**ApplicationContext中的Bean，而不会去扫描Ancestor ApplicationContext。

    * 我们发现，spring-web.xml和dispatch-servlet.xml中均定义了`<context:component-scan base-package="com.hongkun" />`，也就是说，在Root-WebApplicationContext中初始的Bean，在http-servlet-WebApplicationContext中又会初始化一遍，而RequestMappingHandlerMapping Bean是在http-servlet-WebApplicationContext中初始化的，扫描Bean时也就是扫描的http-servlet-WebApplicationContext中的，而被代理的Controller Bean在Root-WebApplicationContext中，所以代理没有生效，也即AOP没有生效。

## 3 问题解决

### 3.1 简单解决
只需要将AOP配置加入dispatch-servlet.xml中即：
```xml
<!-- 加入这一段即可-->
<aop:aspectj-autoproxy proxy-target-class="false" />
<!-- 开启注解 -->
<context:annotation-config />
<!-- 注解扫描基础包 -->
<context:component-scan base-package="com.hongkun" />
<!-- ... ... -->
```
细心的读者已经发现了，这样配置，`<aop:aspectj-autoproxy />`、`<context:component-scan />`都配置了两次，那么在Root-WebApplicationContext中和http-servlet-WebApplicationContext中会有相同的两套Bean，暂时性会没问题，但是当两个`<context:component-scan />`中指定不同的base-package，或少配了一个`<aop:aspectj-autoproxy />`，各种潜在的问题就会爆发出来，比如，@Transaction注解，实则为AOP，你在spring-web.xml中扫描到，而在dispatch-servlet.xml中没有扫描到，一般@Resource、@Autowired注解都会优先取当前Context中的Bean，也就是你的@Transaction一直不生效。
 
### 3.2 标准方案
先将之前文章中的Spring官方对于Root-WebApplicationContext和http-servlet-WebApplicationContext的解释贴一下：

    Spring lets you define multiple contexts in a parent-child hierarchy.
    The applicationContext.xml defines the beans for the "root webapp context", i.e. the context associated with the webapp.
    The spring-servlet.xml (or whatever else you call it) defines the beans for one servlet's app context. There can be many of these in a webapp, one per Spring servlet (e.g. spring1-servlet.xml for servlet spring1, spring2-servlet.xml for servlet spring2).
    Beans in spring-servlet.xml can reference beans in applicationContext.xml, but not vice versa.
    All Spring MVC controllers must go in the spring-servlet.xml context.
    In most simple cases, the applicationContext.xml context is unnecessary. It is generally used to contain beans that are shared between all servlets in a webapp. If you only have one servlet, then there's not really much point, unless you have a specific use for it.

如果你的ApplicationContext只在Web环境中用，那么完全没必要定义ContextLoadListener，只需要定义DispatcherServlet，加载一个WebApplicationContext即可。但是我们一般的标准方案是：
dispatch-servlet.xml配置一些关于Web相关的，比如下面这样
```xml
<!-- 请勿强制使用CGlib代理，只需加入包即可 -->
<aop:aspectj-autoproxy proxy-target-class="false" />
 
<context:annotation-config />
<!-- 只配置到controller，aop扫描也只扫描controller中的，不会跟spring-web中的重复 -->
<context:component-scan base-package="com.hongkun..controller" />

<mvc:annotation-driven>
    <mvc:argument-resolvers>
        <beans:bean class="com.arlen.common.web.argument.RequestJsonArgumentResolver" />
    </mvc:argument-resolvers>
    <mvc:return-value-handlers>
        <beans:bean class="com.arlen.common.web.returnvalue.Base64HandlerMehodReturnValueHandler" />
    </mvc:return-value-handlers>
</mvc:annotation-driven>

```
spring-*.xml中配置Application相关的：
```xml
<!-- 请勿强制使用CGlib代理，只需加入包即可 -->
<aop:aspectj-autoproxy proxy-target-class="false" />

<context:annotation-config />
<!-- 只配置到service，aop扫描也只扫描service中的，不会跟spring-web中的重复 -->
<context:component-scan base-package="com.hongkun..service..impl" />
```

### 3.3 如果只有一个WebApplicationContext，推荐方案
一般，我们只会有一个WebApplicationContext，那么干脆，dispatch-servlet.xml中不要配置bean，只是配简单的资源性的东西，其余全都在spring-*.xml中配置

dispatch-servlet.xml示例配置
```xml
<!-- 静态资源配置 浏览器缓存时间1小时-->
<mvc:resources mapping="/res/**" location="/WEB-INF/res/" cache-period="3600" />
<mvc:resources mapping="/js/**" location="/WEB-INF/js/" cache-period="3600" />
<mvc:resources mapping="/css/**" location="/WEB-INF/css/" cache-period="3600" />
<mvc:resources mapping="/images/**" location="/WEB-INF/images/" cache-period="3600" />
<mvc:resources mapping="/bootstrap/**" location="/WEB-INF/bootstrap/" cache-period="3600" />
<!-- view resolvers 等配置 。。。 -->
```
spring-*.xml示例配置
```xml
<!-- 请勿强制使用CGlib代理，只需加入包即可 -->
<aop:aspectj-autoproxy proxy-target-class="false" />
 
<context:annotation-config />
<context:component-scan base-package="com.hongkun" />

<mvc:annotation-driven>
    <mvc:argument-resolvers>
        <beans:bean class="com.arlen.common.web.argument.RequestJsonArgumentResolver" />
    </mvc:argument-resolvers>
    <mvc:return-value-handlers>
        <beans:bean class="com.arlen.common.web.returnvalue.Base64HandlerMehodReturnValueHandler" />
    </mvc:return-value-handlers>
</mvc:annotation-driven>
```

### 3.4 注意Junit测试
不管你怎么配置，在Junit测试时加载到你所要用的spring配置

## 4 写在最后

### 4.1 代理的Controller Bean，Spring如何获取到TargetObject的方法的

在`AbstractHandlerMethodMapping.detectHandlerMethods`方法中：
```java
public abstract class HandlerMethodSelector {
 
    protected void detectHandlerMethods(final Object handler) {
        Class<?> handlerType =
                (handler instanceof String ? getApplicationContext().getType((String) handler) : handler.getClass());
 
        // Avoid repeated calls to getMappingForMethod which would rebuild RequestMappingInfo instances
        final Map<Method, T> mappings = new IdentityHashMap<Method, T>();
        // 获取用户类，也即获取代理的TargetObject，如果不是代理，则返回自身
        final Class<?> userType = ClassUtils.getUserClass(handlerType);
 
        Set<Method> methods = HandlerMethodSelector.selectMethods(userType, new MethodFilter() {
            @Override
            public boolean matches(Method method) {
                T mapping = getMappingForMethod(method, userType);
                if (mapping != null) {
                    mappings.put(method, mapping);
                    return true;
                }
                else {
                    return false;
                }
            }
        });
        for (Method method : methods) {
            registerHandlerMethod(handler, method, mappings.get(method));
        }
    }
}
```
### 4.2 我之前碰到的问题

问题一：如果dispatch-servlet.xml中没有配置component-scan，但是配置了annotation-driven，无法访问controller方法，无法找到HandlerMethod

解释：因为Spring是在处理`annotation-driven`标签时才去初始化RequestMappingHandlerMapping的，初始化时只会从**当前**ApplicationContext中获取Controller Bean中的方法，不像DispatcherServlet中初始化`handlerMappings`，会从当前及父级ApplicationContext中获取HandlerMapping的Bean。

问题二：如果dispatch-servlet.xml中没有配置`<aop:aspectj-autoproxy />`，无法对controller进行拦截

解释：跟上边一个原因，因为RequestMappingHandlerMapping初始化时，只会从**当前**ApplicationContext中获取Controller Bean中的方法，而不会取到父级ApplicationContext中已经被代理的Controller Bean中的方法。在取Bean方法时，Spring会判断，如果是代理，则取其target Object的方法，见4.1的方法。

PS：建议不明白的读者看看我之前的文章Spring在Web中启动的过程解析。

