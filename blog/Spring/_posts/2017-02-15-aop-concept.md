---
layout: post
title: Spring-AOP概念理解及注解方式的配置和利弊
tags: Spring AOP
source: virgin
---

## 基本概念
 
1. Aspect，中译为切面，即要让代理实现的功能的统称。如果没有AOP，而是用自己写的动态代理，则可以将自己写的动态代理的.java文件看成Aspect。比如，CacheProxy，可以叫做缓存切面。为什么叫切面，是因为，这么一个代理可以切入到任何方法中。
2. Joinpoint，中译为连接点，即在切面的整个流程中（假如一个代理（切面）中有三个方法被切入执行），目标对象的方法被调用的点（也即PointCut所指定的点）
3. Pointcut，中译为切点，即告诉切面，哪些点是要切入执行（被代理代理执行）的。就代码中的AspectJ execution表达式，看成正则表达式，也即一个规则，被这个规则所匹配到的点叫做Joinpoint
4. Advice，中译为通知，分为好几种，around, before和after等。即切面在切点执行连接点的前后、前 和 后，要通知的执行事件。相当于自己的动态代理中，方法执行前、执行后所要执行的方法。
5. Target Object，中译为目标对象，即被切入的对象。就是动态代理被代理的对象。
6. AOP Proxy，AOP创建的代理对象，即运行时为切面中Advice的执行所创建的代理对象。通俗点讲就是运行时的.class对象
7. Weaving，中译为织入。即将切面织入到目标对象中去执行
 
### Aspect
声明切面
```java
@Aspect
public class DictionaryAspect {
}
```
**注意** Aspect除了加@Aspect注解外，必须得确保Spring可以找的到该Component。也就是要么加@Component注解，要么在Srping的配置文件中配置bean。

### Pointcut
#### Pointcut指示符
Spring支持的切入点表达式即AspectJ的切入点表达式。
最常用的切入点指示符（PCD）就是execution和within
* execution - 匹配特定**方法**执行的连接点，这是你将会用到的Spring的最主要的切入点指示符。
* within - 限定匹配特定**类型**的连接点（在使用Spring AOP的时候，在匹配的类型中定义的方法上执行）。 
* this - 限定匹配特定**Bean reference（Spring AOP的代理对象）**的连接点。
* target - 限定匹配特定**目标对象**的连接点。
* args - 限定匹配特定**参数类型**的连接点。这里的参数类型是指方法调用时传进来的实参类型，而不是方法声明时的形参类型。因为动态校验，性能较低，非必要，最好不用。
* @target - 限定匹配**目标对象的类型有指定类型注解**的连接点。
* @args - 限定匹配**运行时参数类型持有指定类型注解**的连接点。
* @within - 限定匹配**连接点所在的类有指定类型注解**的连接点。
* @annotation - 限定匹配**连接点本身方法有指定类型注解**的连接点。
* bean - Spring特定的指示符。限定匹配特定**Spring bean名称**一个或者集合的连接点。

Execution表达式格式如下：
`execution（modifiers-pattern? ret-type-pattern declaring-type-pattern? name-pattern(param-pattern)  throws-pattern?）`
注意，除了返回类型、方法名、参数是必选外，其余均是可选
* 返回类型：最常用的是\*，代表返回值为一切类型
* 方法名：\*可以用来通配所有方法名或部分方法名。
* 参数：一般有下面三类
    * (\*)：匹配一个只有**一个**任何类型参数的方法
    * (..)：匹配一个有任何个数（零或者多个）任何类型参数的方法
    * ()：匹配一个不接受任何参数的方法

eg. `execution(* com.xyz.service..*.*(..))`
匹配com.xyz.service包或子包下，返回值为任意类型，参数为任意个任意类型的所有方法

#### Pointcut每个指示符的例子
* 任意公共方法的执行： `execution(public * *(..))`
* 任何一个名字以“set”开始的方法的执行： `execution(* set*(..))`
* AccountService接口定义的任意方法的执行： `execution(* com.xyz.service.AccountService.*(..))`
* 在service包中定义的任意方法的执行： `execution(* com.xyz.service.*.*(..))`
* 在service包或其子包中定义的任意方法的执行： `execution(* com.xyz.service..*.*(..))`
* 在service包中的任意连接点(在Spring AOP中只是方法执行)：`within(com.xyz.service.*)`
* 在service包或其子包中的任意连接点(在Spring AOP中只是方法执行)：`within(com.xyz.service..*)`
* 实现了AccountService接口的**代理对象**的任意连接点 (在Spring AOP中只是方法执行)：`this(com.xyz.service.AccountService)`
* 实现AccountService接口的**目标对象**的任意连接点 (在Spring AOP中只是方法执行)：`target(com.xyz.service.AccountService)`
* 任何一个只接受一个参数，并且运行时所传入的参数是Serializable 接口的连接点(在Spring AOP中只是方法执行)：`args(java.io.Serializable)`
  请注意在例子中给出的切入点不同于 execution(* *(java.io.Serializable))： args版本只有在动态**运行时**候传入参数是Serializable时才匹配，而execution版本在方法签名中**声明**只有一个 Serializable类型的参数时候匹配。
* 目标对象中有一个 @Transactional 注解的任意连接点 (在Spring AOP中只是方法执行)：`@target(org.springframework.transaction.annotation.Transactional)`
* 任何一个**目标对象**声明的类型有一个 @Transactional 注解的连接点 (在Spring AOP中只是方法执行)：`@within(org.springframework.transaction.annotation.Transactional)`
* 任何一个执行的**方法**有一个 @Transactional 注解的连接点 (在Spring AOP中只是方法执行)：`@annotation(org.springframework.transaction.annotation.Transactional)`
* 任何一个只接受一个参数，并且运行时所传入的参数类型具有@Classified 注解的连接点(在Spring AOP中只是方法执行)：`@args(com.xyz.security.Classified)`
  具体例子参照帖子：http://www.javarticles.com/2015/09/spring-aop-annotation-pointcut-designator-example.html
* 任何一个在名为'tradeService'的Spring bean之上的连接点 (在Spring AOP中只是方法执行)：`bean(tradeService)`
* 任何一个在名字匹配通配符表达式'*Service'的Spring bean之上的连接点 (在Spring AOP中只是方法执行)：`bean(*Service)`

另外，表达式可以组合使用，使用&&和\|\|来组合；已经声明的Pointcut可以在其他地方引用。可以在其他PointCut中引用，也可以在Advice中引用，见最后边的例子

### Advice
#### Advice类型
* Before advice：连接点执行前，不可直接返回来中断调用连接点。只能抛出异常来中断
* After returning advice：连接点正常执行之后
* After throwing advice：连接点抛出异常时执行
* After (finally)advice：连接点最后退出时，相当于finally块中，不论正常与否，抛出异常与否，均执行
* Around advice：包围连接点。可选择继续执行或者直接返回，或者抛出异常返回。在某种线程安全的环境下，需要在连接点执行前后共享某个状态时可放心使用。

#### Advice配置
需要注意的地方：
* Before advice：正常配置，指定Pointcut即可。例如 `@Before（"execution（* com.xyz.myapp.dao.*.*（..））"）`
* After returning advice：除了正常配置，如果要获取连接点的实际返回值，可如下配置
  ```
  @AfterReturning（
    pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation（）",
    returning="retVal"）
  public void doAccessCheck（Object retVal） {
    // ...
  }
  ```
  注意，returning指定的参数值必须跟通知方法的入参一致。
* After throwing advice：除了正常配置，如果要获取抛出的异常，可如下配置
  ```
  @AfterThrowing（
    pointcut="com.xyz.myapp.SystemArchitecture.dataAccessOperation（）",
    throwing="ex"）
  public void doRecoveryActions（DataAccessException ex） {
    // ...
  }
  ```
* After (finally)advice：正常配置
* Around advice：第一个参数必须为ProceedingJoinPoint，如下
  ```
  @Around（"com.xyz.myapp.SystemArchitecture.businessService（）"）
  public Object doBasicProfiling（ProceedingJoinPoint pjp） throws Throwable {
    // start stopwatch
    // pjp.proceed，即调用连接点方法。可以传入参数，见下文
    Object retVal = pjp.proceed（）;
    // stop stopwatch
    return retVal;
  }
  ```

#### Advice参数
很多情况下，我们在通知中想获取连接点的参数，而Spring提供了相应的机制

1. 在通知方法中访问当前的连接点
任何通知方法可以将第一个参数定义为org.aspectj.lang.JoinPoint类型（环绕通知的第一个参数ProceedingJoinPoint，它是JoinPoint的一个子类）。JoinPoint接口提供了一些列方法，比如：getArgs();（返回方法参数）、getThis();（返回代理对象）、getTarget();（返回目标对象）、getSignature();（返回正在被通知的方法的相关信息）、toString();（打印正在被通知的方法的有用信息）

2. 传递参数给通知方法
前边看到了如何获取返回值和获取抛出的异常，为了在通知方法内获取参数，我们使用指示符`args`表达式来绑定。将`args`中用来指定参数类型的地方写成一个参数名，那么当前通知方法执行时对应的参数值就会用该参数传进来。
```java
@Before("com.xyz.myapp.SystemArchitecture.dataAccessOperation() &&" +
        "args(account,..)")
public void validateAccount(Account account) {
  // ...
}
```
args(account,..)，有两个含义。一是，限定一个最少一个参数的方法，且第一个参数类型必须是Account的方法；二是，其使得在通知方法体内，可以通过account获取到连接点的Account对象。

另一种定义方法，定义一个切入点，在切入点指定参数，在Advice时引用
```java
@Pointcut("com.xyz.myapp.SystemArchitecture.dataAccessOperation() &&" +
          "args(account,..)")
private void accountDataAccessOperation(Account account) {}
 
@Before("accountDataAccessOperation(account)")
public void validateAccount(Account account) {
  // ...
}
```
代理对象（this）、目标对象（target） 和注解（@within, @target, @annotation, @args）都可以用一种类似的格式来绑定。详见官网。

3. 指定参数名

在通知和切入点的注解中，有一个"argNames"属性用来指定参数名称。最好不用吧，具体详见官网。

4. 处理参数
```java
@Around("execution(List<Account> find*(..) &&" +
        "com.xyz.myapp.SystemArchitecture.inDataAccessLayer() && " +
        "args(accountHolderNamePattern)")       
public Object preProcessQueryPattern(ProceedingJoinPoint pjp, String accountHolderNamePattern)
throws Throwable {
  String newPattern = preProcess(accountHolderNamePattern);
  return pjp.proceed(new Object[] {newPattern});
} 
```
大部分情况下，我们都会这样去给proceed绑定参数

#### 通知顺序
如果有多个通知想要在同一个连接点执行，那么通知执行的次序是未知的。可以使用下面两种方式解决：
1. 切面类实现org.springframework.core.Ordered，实现getValue方法
2. 使用Order注解，属性value

在多个切面中，Order.getValue()返回值越小的优先级越高

eg：
```java
package com.xyz.someapp;
 
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
 
@Aspect
public class SystemArchitecture {
 
  /**
   * web层的所有类
   */
  @Pointcut（"within（com.xyz.someapp.web..*）"）
  public void inWebLayer（） {}
 
  /**
   * service层的所有类
   */
  @Pointcut（"within（com.xyz.someapp.service..*）"）
  public void inServiceLayer（） {}
 
  /**
   *  dao层的所有类
   */
  @Pointcut（"within（com.xyz.someapp.dao..*）"）
  public void inDataAccessLayer（） {}
 
  /**
   * 同上边的inServiceLayer
   */
  @Pointcut（"execution（* com.xyz.someapp.service.*.*（..））"）
  public void businessService（） {}
 
  /**
   * 同上边的inDataAccessLayer
   */
  @Pointcut（"execution（* com.xyz.someapp.dao.*.*（..））"）
  public void dataAccessOperation（） {}

  /**
   * 一个组合的例子，也引用了自身类中已定义的businessService和dataAccessOperation
   */ 
  @Pointcut（"businessService() && dataAccessOperation()"）
   private void test（） {}
}
```
```java
  import org.aspectj.lang.annotation.Aspect;
  import org.aspectj.lang.annotation.Before;
 
  @Aspect
  public class BeforeExample {
 
  /**
   * 引用了上边定义的dataAccessOperation
   */
  @Before（"com.xyz.myapp.SystemArchitecture.dataAccessOperation（）"）
  public void doAccessCheck（） {
    // ...
  }
  /**
   *或者自己定义
   */
  @Before（"execution（* com.xyz.myapp.dao.*.*（..））"）
  public void doAccessCheck（） {
    // ...
  }
}
```

## 写在最后
用惯了注解方式，当然觉得用注解方式配置AOP比较清爽简单。但是他也有自己的缺点。
1. 使用注解方式，在Advice方法体访问连接点，只能是方法的第一个参数为JoinPoint，然而，该接口只有几个简单的方法，操作很不便利。没有XML中用Interceptor来的方便，其可以反射得到你想要的任何信息。当然，注解方式也可以，比如注解也可以传入，但是，对于Mybatis接口方式的Mapper，其不光不支持CGlib代理，还无法匹配接口中的注解，很是蛋疼。
2. 使用注解方式，无法用切点匹配到接口方法上的注解！

