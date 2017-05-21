---
layout: post
title: HandlerMethodArgumentResolver详解
tags: Spring AOP
source: virgin
---

    HandlerMethodArgumentResolver，用来将HttpRequest转换为Controller的形参。是Spring4.x中的新概念，其摈弃了Spring3.x中的WebArgumentResolver。平时web请求的参数转换，全都是经过他转的。

## 1. 他在Spring-MVC中充当什么角色呢？
之前也讲过了，这里简单再描述下，所有Http请求，经DispatcherServlet处理，然后给到RequestMappingHandlerAdapter处理，最终让InvocableHandlerMethod来调用实际的Controller方法去处理Http请求。在InvocableHandlerMethod触发Controller之前，会对HttpRequest中的输入流做解析和转换，转换为Controller对应的方法形参，这里就是用了HandlerMethodArgumentResolver来做的解析和转换。

## 2. 他是怎么转换的呢？
HandlerMethodArgumentResolver有很多不同的实现，其support方法决定了可以对什么形参做转换，不同的实现有不同的转换方式，大的分可以分为五种。
* 调用HttpMessageConverter转换
    对注解@RequestBody、@ResponseBody、类型HttpEntity，会调用Spring装配的HttpMessageConverter来转换；
* 调用ConversionService转换
    对基本的参数注解比如@MatrixVariable、@RequestParam且形参为Map时注解有value属性的，会调用ConversionService来转换
* 对形参为Map且注解未注明value的转换
    对形参为Map类型，且有基本的注解，但是注解没有value属性的，直接转换为Map返回
* 对特殊类型的转换
    比如形参为ServletRequest、ServletResponse的类型，直接将上下文中的信息赋值。
* 兼容3.x中的WebArgumentResolver

## 3.具体实现类
![Criteria结构图]({{site.url}}/assets/img-blog/Spring/argument-resolver-generetic.png)

基本上分为五大类
1. AbstractMessageConverterMethodArgumentResolver阵营
    处理@ResponseBody、@RequestBody和HttpEntity类型的形参和返回值的Controller方法，特殊之处在于，有这三个的，都会调用注册的**HttpMessageConverter**来做实际的参数转换。不同的HttpMessageConverter根据不同的MediaType和自己的特殊逻辑，来调用合适的HttpMessageConverter做转换。

    基本逻辑是由AbstractMessageConverterMethodArgumentResolver将HttpServletRequest转换为HttpInputMessage（即将HttpServletRequest中的inputStream封装为HttpInputMessage），然后交给HttpMessageConverter去做转换。

2. AbstractNamedValueMethodArgumentResolver阵营
    处理其实现类MethodArgumentResolver对应的注解（即子类名称MethodArgumentResolver之前的部分），并且，如果形参类型为Map，那么该注解必须有value属性，也即不光有注解，还在注解中指定了具体的参数名，该参数名即HttpRequest中的参数名！

    基本逻辑是子类实现support方法，确定是否可以被使用，然后由`AbstractNamedValueMethodArgumentResolver->resolveArgument()`做解析，其调用了子类的`resolveName()`方法，该方法是每个子类都会实现的，是解析的核心。该方法也是从HttpServletRequest中获取对应名字的参数，参数解析到之后，便调用入参WebDataBinderFactory，创建新的WebDataBinder，然后WebDataBinder用其封装的ConversionService，调用Spring core中的各种**Converter**做实际的参数转换。

3. AbstractWebArgumentResolverAdapter阵营
    为了兼容Spring3.x中的WebArgumentResolver而生，已不建议使用

4. *MapMethodArgumentResolver阵营
    处理对应**Map**MethodArgumentResolver对应的注解的（即类名MapMethodArgumentResolver前边的部分），形参为Map类型，并且其都是为了处理只有注解，没有指定具体参数名的情况。直接从WebRequest中取参数（getParameterMap）转换为Map，也即取HttpServletRequest中的ParameterMap。

5. 特殊类型阵营
    处理对应的MethodArgumentResolver对应的特殊类型，比如ServletRequest、SessionStatus等。和一些特殊的注解比如ModelAttribute等。直接将上下文中的信息赋值给形参。

注：看到有些别出一阁的Processor，这是啥，其实他就是多实现了一个接口`HandlerMethodReturnValueHandler`，也即，他不光处理入参转换，还处理返回值转换。
 
**重点**：可以看到，除了使用HttpMessageConverter的ArgumentResolver，其他的ArgumentResolver都是取的HttpServletRequest中的Parameter，而不管是Parameter还是InputStream，我们都是可以在Filter中封装自己的HttpServletRequest的（HttpServletRequestWrapper），在HttpServletRequestWrapper中，对Parameter和InputStream做一些处理。


