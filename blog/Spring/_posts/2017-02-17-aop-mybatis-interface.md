---
layout: post
title: Spring AOP拦截Mybatis接口方式的Mapper问题
tags: Spring AOP
source: virgin
---

    因为之前写了一个字典项填充插件，需要对select出来的对象填充字典项的中英文字段，考虑到这种东西应该对数据访问层就透明，必然不能放到service里去做。那就想到了动态代理来做。Spring框架中AOP是绝佳选择。但是问题来了，同样的Pointcut表达式，service能拦截，但是dao的接口就不能拦截，纠结了一天。如果要看怎么解决，直接看问题解决即可。

## 问题描述
之前改写的Mybatis-generator插件，可以在生成的Domain中添加字典项的中英文名字段，然后在省城的Dao接口的select*方法上生成@DictionaryFill注解。有了这些，我们就可以在Dao接口上增加AOP拦截，拦截有DictionaryFill注解的方法，并对其返回值做字典项填充。

### 生成的Domain文件

省略了其余无用字段
```java
@DictionaryFill(OrderExtraFee.class)
public class OrderExtraFee {
    /**
     * 自动生成字段，费用编码Dict[currency,cn,en]
     */
    private String feeNameCN;
    /**
     * 自动生成字段，费用编码Dict[currency,cn,en]
     */
    private String feeNameEN;
    /**
     * 费用编码Dict[currency,cn,en]
     */
    @Dictionary(dictIndex="currency", cnFlag=true, enFlag=true)
    private String feeCode;
    /**
     * 自动生成get方法，feeNameCN
     * @return feeNameCN 费用编码Dict[currency,cn,en]
     */
    public String getFeeNameCN() {
        return feeNameCN;
    }
    /**
     * 自动生成set方法，feeNameCN
     * @param String feeNameCN 费用编码Dict[currency,cn,en]
     */
    public void setFeeNameCN(String feeNameCN) {
        this.feeNameCN = feeNameCN == null ? null : feeNameCN.trim();
    }
    /**
     * 自动生成get方法，feeNameEN
     * @return feeNameEN 费用编码Dict[currency,cn,en]
     */
    public String getFeeNameEN() {
        return feeNameEN;
    }
    /**
     * 自动生成set方法，feeNameEN
     * @param String feeNameEN 费用编码Dict[currency,cn,en]
     */
    public void setFeeNameEN(String feeNameEN) {
        this.feeNameEN = feeNameEN == null ? null : feeNameEN.trim();
    }
    /**
     * @return fee_code 费用编码Dict[currency,cn,en]
     */
    public String getFeeCode() {
        return feeCode;
    }
    /**
     * @param String feeCode 费用编码Dict[currency,cn,en]
     */
    public void setFeeCode(String feeCode) {
        this.feeCode = feeCode == null ? null : feeCode.trim();
    }
}
```

### 生成的Dao文件
```java
public interface IOrderExtraFeeDao {
    int countByExample(OrderExtraFeeQuery example);
 
    int deleteByPrimaryKey(Integer id);
 
    int insert(OrderExtraFee record);
 
    int insertSelective(OrderExtraFee record);
 
    @DictionaryFill(OrderExtraFee.class)
    List<OrderExtraFee> selectByExample(OrderExtraFeeQuery example);
 
    @DictionaryFill(OrderExtraFee.class)
    OrderExtraFee selectByPrimaryKey(Integer id);
 
    int updateByPrimaryKeySelective(OrderExtraFee record);
 
    int updateByPrimaryKey(OrderExtraFee record);
 
    int batchInsertSelective(List<OrderExtraFee> recordList);
 
    int batchUpdateByPrimaryKeySelective(List<OrderExtraFee> recordList);
}
```

### 自定义注解
```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface DictionaryFill {
    Class<?> value();
}
```

### 填充工具类
示例性方法，没有做严格的逻辑控制，不看也罢
```java
public class DictionaryTool {
 
    private class DictConfig {
        private String dictIndex;
        private Map<String, ? extends BaseDictionary> dictItemMap;
        private String codeField;
        private boolean cnFlag;
        private boolean enFlag;
        private String codeGetMethod;
        private String nameCNSetMethod;
        private String nameENSetMethod;
 
        public void setFieldInfo(String codeField) {
            this.codeField = codeField;
            this.nameCNSetMethod = getNameCNMethod();
            this.nameENSetMethod = getNameENMethod();
            this.codeGetMethod = getCodeMethod();
        }
        public String getCodeMethod() {
            String fieldName = this.codeField;
            String upper = fieldName.substring(0,1).toUpperCase();
            String methodName = "get" + upper + fieldName.substring(1);
            return methodName;
        }
 
        public String getNameCNMethod() {
            return getNameMethod("set", "NameCN");
        }
        public String getNameENMethod() {
            return getNameMethod("set", "NameEN");
        }
 
        // 拼接方法名
        private String getNameMethod(String prefix, String suffix) {
            String fieldName = codeField;
            if (fieldName.endsWith("code") || fieldName.endsWith("Code")) {
                fieldName = fieldName.substring(0, fieldName.length() - 4);
            } else if (fieldName.endsWith("co") || fieldName.endsWith("Co")) {
                fieldName = fieldName.substring(0, fieldName.length() - 2);
            }
            fieldName = fieldName + suffix;
            String upper = fieldName.substring(0,1).toUpperCase();
            String methodName = "set" + upper + fieldName.substring(1);
            return methodName;
        }
 
        // 填充信息
        public void fillInfo(Object obj, Class<?> clazz) {
            Method codeGetMethod = ReflectionUtils.findMethod(clazz, this.codeGetMethod);
            Object code = ReflectionUtils.invokeMethod(codeGetMethod, obj);
            if (dictItemMap == null || dictItemMap.size() == 0) {
                dictItemMap = DictinaryCache.getDictionaryByIndex(this.dictIndex);
            }
            BaseDictionary dictionary = dictItemMap.get(code);
            if (dictionary == null) {
                System.out.println("找不到该字典项");
                return;
            }
 
            if (this.cnFlag) {
                Method method = ReflectionUtils.findMethod(clazz, this.nameCNSetMethod, String.class);
                ReflectionUtils.invokeMethod(method, obj, dictionary.getNameCNByCode((String)code));
            }
 
            if (this.enFlag) {
                Method method = ReflectionUtils.findMethod(clazz, this.nameENSetMethod, String.class);
                ReflectionUtils.invokeMethod(method, obj, dictionary.getNameENByCode((String)code));
            }
        }
    }
 
    public void fillName(Class<?> clazz, Object... dtoList) {
        if (dtoList == null || dtoList.length == 0 || clazz == null) {
            return;
        }
 
        DictionaryFill fill = clazz.getAnnotation(DictionaryFill.class);
        if (fill == null) {
            return;
        }
 
        // 同一个类中有几个dictIndex
        List<DictConfig> dictConfigList = new ArrayList<DictConfig>();
        List<Field> fieldList = Arrays.asList(clazz.getDeclaredFields());
        for (Field field : fieldList) {
            Dictionary dictionary = field.getAnnotation(Dictionary.class);
            if (dictionary == null) {
                continue;
            }
            DictConfig dictConfig = new DictConfig();
            dictConfig.dictIndex = dictionary.dictIndex();
            dictConfig.cnFlag = dictionary.cnFlag();
            dictConfig.enFlag = dictionary.enFlag();
            dictConfig.setFieldInfo(field.getName());
            dictConfigList.add(dictConfig);
        }
 
        // 遍历多个index，为每个对象填充值
        for (Object object : dtoList) {
            if (null == object) continue;
            for (DictConfig dictConfig : dictConfigList) {
                dictConfig.fillInfo(object, clazz);
            }
        }
 
    }
 
}
```

### 最开始的想法
针对Dao中的方法设置切点。Dao没有实现类。比如像下面这样：
```java
@Component
@Aspect
public class DictionaryAspect {
 
    // 1. @Pointcut("execution(* com.hongkun..service..*.*(..))")
    // 2. @Pointcut("execution(* com.hongkun..service..*.*(..)) && @annotation(com.hongkun.common.annotation.DictionaryFill)")
    // 3. @Pointcut("execution(* com.hongkun..dao..*.select*(..))")
    @Pointcut("execution(* com.hongkun..dao.*.*(..)) && @annotation(com.hongkun.common.annotation.DictionaryFill) && args(dictionaryFill,..)")
    public void dictionaryFill() {}
 
    // @AfterReturning(pointcut = "dictionaryFill()")
    @AfterReturning(pointcut = "dictionaryFill()", returning = "returnVal")
    public void doDictionaryFill(DictionaryFill dictionaryFill, Object returnVal) {
 
        if (dictionaryFill == null || dictionaryFill.value() == null) {
            return;
        }
 
        if (returnVal.getClass().isPrimitive()) {
            return;
        }
 
        DictionaryTool tool = new DictionaryTool();
        if (returnVal instanceof Collection<?>) {
            tool.fillName(dictionaryFill.value(), ((Collection<?>) returnVal).toArray());
        } else {
            tool.fillName(dictionaryFill.value(), returnVal);
        }
    }
} 
```
### 碰到的问题
1. 如上代码中的Pointcut表达式1所示，简单的execution匹配，如果是service会被切面织入。如果换成dao，则不行

   查度娘查了好久，最后有人说是因为项目中强制使用了CGlib代理所致。因为CGlib代理都要接口有实现类，最终是在实现类代理，也即类要有构造方法。而Dao只是接口，并没有实现类，所以切面无法织入。顾此将项目设置成非强制性用CGlib即可。

2. 解决了问题1后，如上Pointcut表达式2所示用@annotation匹配，如果是service会织入，如果是dao则不行

   又是个头疼的问题，service用@annotation匹配没有任何问题，dao就是不行。上网查了无果，翻AspectJ、AOP源码，没找到症结所在。于是，用参数传入Annotation的做法不可行。顾此，采用了表达式3的折中方法：用execution匹配select*方法，用反射获取注解信息。

3. 问题2解决之后，也就伴随着遗留问题，无法获取方法的注解，从而无法确定selectList返回的泛型具体是什么类型

   于是想到了，在Advice方法体通过反射获取方法上的注解。AspectJ注解方式和XML配置的方式均可实现。

注：虽说我的实际问题不能用AspectJ的注解@annotation方式解决，但是，我们可以在service层来做这样的事情，见下解决方案。

## 问题解决

看了上边的问题，AspectJ注解的方式和XML配置的方式，均不能使用@annotation来切入并将参数传入Advice方法体，只能用反射去获取注解。要用@annotation，必须有实现类，即包一层service即可。

### 首先设置项目非强制CGlib

```xml
<aop:aspectj-autoproxy proxy-target-class="false" />
<!-- 如果设置了事务注解方式，一样也不能强制CGlib -->
<tx:annotation-driven transaction-manager="transactionManager" proxy-target-class="false" />
```

### 使用XML配置方式——未使用@annotation

因Pointcut无法拦截到接口方法定义的注解，故此，在方法拦截器中通过反射获取
spring-aop文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:aop="http://www.springframework.org/schema/aop" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
    http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
    http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.1.xsd
 
    <!-- 请勿强制使用CGlib代理，只需加入包即可 -->
    <aop:aspectj-autoproxy proxy-target-class="false" />
 
    <aop:config>
        <aop:pointcut id="dictPointcut" expression="(execution(* com.hongkun..dao..*.select*(..)))" />
        <aop:advisor advice-ref="dictAfterAdvisor" pointcut-ref="dictPointcut" order="100"/>
    </aop:config>
 
    <bean id="dictAfterAdvisor" class="com.hongkun.order.aspect.DictioanryInterceptor" />
</beans>
```

方法拦截器DictioanryInterceptor
```java
public class DictioanryInterceptor implements MethodInterceptor {
 
    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        Object returnVal = invocation.proceed();
 
        DictionaryFill dictionaryFill = invocation.getMethod().getAnnotation(DictionaryFill.class);
 
        if (dictionaryFill == null || dictionaryFill.value() == null) {
            return returnVal;
        }
 
        if (returnVal.getClass().isPrimitive()) {
            return returnVal;
        }
 
        DictionaryTool tool = new DictionaryTool();
        if (returnVal instanceof Collection<?>) {
            tool.fillName(dictionaryFill.value(), ((Collection<?>) returnVal).toArray());
        } else {
            tool.fillName(dictionaryFill.value(), returnVal);
        }
        return returnVal;
    }
}
```

### 使用AspectJ注解方式——未使用@annotation
Aspect配置，因无法使用@annotation，用连接点的signature强转MethodSignature，来反射获取注解信息
```java
@Component
@Aspect
public class DictionaryAspect {
 
    @AfterReturning(value = "execution(* com.hongkun..dao..*.select*(..))", returning = "returnVal")
    public void doDictionaryFill1(JoinPoint joinPoint, Object returnVal) {
        Signature signature = joinPoint.getSignature();
        DictionaryFill dictionaryFill = (MethodSignature)methodSignature.getMethod().getAnnotation(DictionaryFill.class);
 
        if (dictionaryFill == null || dictionaryFill.value() == null) {
            return;
        }
 
        if (returnVal.getClass().isPrimitive()) {
            return;
        }
 
        DictionaryTool tool = new DictionaryTool();
        if (returnVal instanceof Collection<?>) {
            tool.fillName(dictionaryFill.value(), ((Collection<?>) returnVal).toArray());
        } else {
            tool.fillName(dictionaryFill.value(), returnVal);
        }
    }
} 
```

### 使用AspectJ注解方式——使用@annotation

1 再封装一层service

```java
@Service
public class OrderExtraFeeServiceImpl implements IOrderExtraFeeService {
 
    @Resource
    private IOrderExtraFeeDao orderExtraFeeDao;
 
    @Override
    @DictionaryFill(OrderExtraFee.class)
    public List<OrderExtraFee> selectOrderFeeList(OrderExtraFee feeQuery) {
        OrderExtraFeeQuery query = (OrderExtraFeeQuery)GeneratorUtils.transDomainToQuery(feeQuery);
        List<OrderExtraFee> feeList = orderExtraFeeDao.selectByExample(query);
        return feeList;
    }
 
}
```

2 Aspect配置

```java
@Component
@Aspect
public class DictionaryAspect {
 
    @AfterReturning(value = "execution(* com.hongkun..service..*.*(..)) && @annotation(dictionaryFill)", returning="returnVal")
    public void doDictionaryFill(DictionaryFill dictionaryFill, Object returnVal) {
        if (dictionaryFill == null || dictionaryFill.value() == null) {
            return;
        }
 
        if (returnVal.getClass().isPrimitive()) {
            return;
        }
 
        DictionaryTool tool = new DictionaryTool();
        if (returnVal instanceof Collection<?>) {
            tool.fillName(dictionaryFill.value(), ((Collection<?>) returnVal).toArray());
        } else {
            tool.fillName(dictionaryFill.value(), returnVal);
        }
 
    }
} 
```

## 写到最后不得不说的坑——@annotation

Pointcut表达式用@annotation，并且将**自定义**注解传入Advice方法体。Spring reference中讲的很清楚了，如下：
```java
@AfterReturning(value = "execution(* com.hongkun..service..*.*(..)) && @annotation(dictionaryFill)")
public void doDictionaryFill(DictionaryFill dictionaryFill) {}
```
注意@annotation中，本该放类型，但是放了小写的变量名，然后变量名跟Advice通知方法的参数名一样。可是，你直接运行，会报错！！报错！！！
```shell
error Type referred to is not an annotation type: dictionaryFill
```
错误是，找不到注解类型dictionaryFill。纠结了整整一个下午，最后在Stack Overflow中看到，需要Maven-》Update Project一下，当时觉得太不可思议，结果真管用！

Stack Overflow中的另一个坑：http://stackoverflow.com/questions/8574348/error-type-referred-to-is-not-an-annotation-type
