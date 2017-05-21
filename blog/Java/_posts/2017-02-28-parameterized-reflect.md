---
layout: post
title: 通过反射获取泛型类以及原理解析
tags: Java
source: virgin
---

    编码过程中，经常会遇到泛型，然后通用的一些方法，想用反射获取到泛型的实际类型，之前一直以为没办法，都是用注解在变量前边指定的，今天查了下，还是有方法的。

获取泛型的实际类型，得先明白其原理。反射是如何做的，其实是取的编译好的.class文件中的信息来解析的（表述可能不准确），而泛型是什么，是在运行时才会确定的东西。那么也就知道关键点了。要获取泛型的类型，只能是获取那些已经在代码中指定了实际泛型类型的代码。有点绕，也即类中声明的 泛型类型的属性、返回值为泛型类型的方法、参数为泛型类型的方法 和 泛型类型的子类。而声明的局部变量和泛型类本身是无法获取实际的运行时泛型类型的。下面详细介绍。

**强调**： 以下所有均是我根据自己的使用场景总结的，肯定有遗漏之处，大佬们见谅。待我有空恶补java基础时再补充。另外这是一篇长文，费了我两天时间，如果你不吝看完，想必收获会很大。

## 1 先来个栗子
### 1.1 栗子
下面就各种情况都举例说明，尽量结合实际研发中可能会碰到的情况举例。通过篮子、苹果篮子、苹果、学生几个类来说明问题。想必看了代码就都明白了，如果不懂，下面会有解析。
```java
package com.arlen.common.paramtype;
 
public class Apple {
 
}
```
```java
package com.arlen.common.paramtype;
 
import java.util.ArrayList;
import java.util.List;
 
public class Basket<T> {
 
    private List<T> goodsList = new ArrayList<T>();
 
    public void put(T goods) {
        goodsList.add(goods);
    }
 
    public List<T> getGoodsList() {
        return goodsList;
    }
}
```
```java
package com.arlen.common.paramtype;
 
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
 
public class AppleBasket extends Basket<Apple> {
 
    // 1. 获取子类的泛型类型
    public void getSubClassActualType() {
        Type superType = AppleBasket.class.getGenericSuperclass();
        if (ParameterizedType.class.isAssignableFrom(superType.getClass())) {
            ParameterizedType superParamType = (ParameterizedType)superType;
            System.out.println("Actual subclass argument: " + superParamType.getActualTypeArguments()[0]);
        }
    }
}
```
```java
package com.arlen.common.paramtype;
 
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
 
import org.springframework.util.ReflectionUtils;
 
public class Student {
 
    private Basket<Apple> appleBasket;
 
    public Basket<Apple> getAppleBasket() {
        return appleBasket;
    }
 
    public void setAppleBasket(Basket<Apple> appleBasket) {
        this.appleBasket = appleBasket;
    }
 
    // 2. 获取属性的泛型类型
    public void getFieldActualType() {
        Field field = ReflectionUtils.findField(Student.class, "appleBasket");
        Type fieldType = field.getGenericType();
        if (ParameterizedType.class.isAssignableFrom(fieldType.getClass())) {
            ParameterizedType fieldParamType = (ParameterizedType)fieldType;
            System.out.println("Actual type arguments: "+fieldParamType.getActualTypeArguments()[0]);
        }
    }
 
    // 3. 获取方法的返回值泛型类型
    public void getMethodReturnActualType() {
        Method method = ReflectionUtils.findMethod(Student.class, "getAppleBasket");
        Type returnType = method.getGenericReturnType();
        if (ParameterizedType.class.isAssignableFrom(returnType.getClass())) {
            ParameterizedType returnParamType = (ParameterizedType)returnType;
            System.out.println("Actual return arguments: "+returnParamType.getActualTypeArguments()[0]);
        }
    }
 
    // 4. 获取方法的参数泛型类型
    public void getMethodParamActualType() {
        Method method = ReflectionUtils.findMethod(Student.class, "setAppleBasket", Basket.class);
        Type[] paramTypes = method.getGenericParameterTypes();
        for (Type paramType : paramTypes) {
            if (ParameterizedType.class.isAssignableFrom(paramType.getClass())) {
                ParameterizedType paramParamType = (ParameterizedType)paramType;
                System.out.println("Actual param arguments: " + paramParamType.getActualTypeArguments()[0]);
            }
        }
    }
 
    // 5. 获取局部变量的泛型类型——行不通
    public void getVariableActualType() {
        Basket<Apple> appleBasket = new Basket<Apple>();
        // 想要获取该局部变量 appleBasket 的泛型类型是没有任何办法的，原因看class文件的截图
    }
 
    // 5. 获取局部变量的泛型类型——行得通
    public void getVariableActualType1() {
        // Basket<Apple> appleBasket = new Basket<Apple>();
        AppleBasket appleBasket = new AppleBasket();
        Type type = appleBasket.getClass().getGenericSuperclass();
        if (ParameterizedType.class.isAssignableFrom(type.getClass())) {
            System.out.println("Actual variable arguments: " + ((ParameterizedType)type).getActualTypeArguments()[0]);
        }
    }
 
    public static void main(String[] args) {
        new AppleBasket().getSubClassActualType();
        new Student().getFieldActualType();
        new Student().getMethodReturnActualType();
        new Student().getMethodParamActualType();
        new Student().getVariableActualType1();
    }
}
```
### 1.2 写代码时的一些思考
![写代码时的一些思考1]({{site.url}}/assets/img-blog/Java/reflect-example-1.png)
![写代码时的一些思考2]({{site.url}}/assets/img-blog/Java/reflect-example-2.png)
![写代码时的一些思考3]({{site.url}}/assets/img-blog/Java/reflect-example-3.png)
![写代码时的一些思考4]({{site.url}}/assets/img-blog/Java/reflect-example-4.png)
![写代码时的一些思考5]({{site.url}}/assets/img-blog/Java/reflect-example-5.png)


## 2 原理解析

### 2.1 class文件解析
1. 获取子类的泛型类型

    ![class文件解析1]({{site.url}}/assets/img-blog/Java/reflect-class-1.png)

2. 获取属性的泛型类型

    ![class文件解析2]({{site.url}}/assets/img-blog/Java/reflect-class-2.png)

3. 获取方法返回值的泛型类型

    ![class文件解析3]({{site.url}}/assets/img-blog/Java/reflect-class-3.png)

4. 获取方法参数的泛型类型

    ![class文件解析4]({{site.url}}/assets/img-blog/Java/reflect-class-4.png)

5. 获取局部变量的泛型类型——行不通

    ![class文件解析5]({{site.url}}/assets/img-blog/Java/reflect-class-5.png)

经过查看编译后的class文件，可以看到class文件中只要有`// Signature`标识的，都能获取到具体的泛型类型。也就是说，只有你在代码里指定具体的泛型类型，反射才可以拿到。也可以看到，为什么局部变量想拿到其反射类型拿不到，因为编译好的class文件中并没有`// Signature`标识来指定具体的泛型类型。

### 2.2 Class 和 Type
从最开始的例子中的代码可以看出，获取泛型的实际类型，实际上都是通过`ParameterizedType`接口来获取的，而`ParameterizedType`接口的父接口是`Type`。那么会有个疑问，Class类在运行时持有类的所有反射信息，为什么会有Type。翻看源代码，发现Type是Class的父类，也是java所有类型的父类。不管是原生类型、泛型类型、数组还是一般类型。

#### 2.2.1 Type
Type是java的所有类型的父接口，包括接口类型： 原生类型（raw types）、参数化类型（parameterized types）、数组类型（array types）、类型变量(泛型)（type variables）、原始类型（primitive types）。为什么会有Type这个东西，是因为加入了泛型而生的，具体请看文章最后。

以下是Type的简单类图，图片来自网络，自己没有画

![Type简单类图]({{site.url}}/assets/img-blog/Java/reflect-type-class.png)

Type 直接子类
* Class: 见下面详解

#### 2.2.2 Class
Java运行时为所有对象维护一个运行时类型标识，即Class对象。其保存了对象所属类的所有信息。
以下是Class、Method、Field的相关类图，图片来自网络，自己没有画
![Class相关类图]({{site.url}}/assets/img-blog/Java/reflect-class-method-field-class.png)

### 2.3 使用实例
首先得知道，Type以及其直接子接口都是为了泛型而生。下面的讲解得一直参考上边的第一张类图

### 2.3.1 Type直接子接口都是啥，怎么用
* ParameterizedType: 表示一种参数化的类型，比如List<String>、List<T>，总之带尖括号的都是
    * 方法：getActualTypeArguments，获取`<>`中间的类型
* GenericArrayType: 表示一种元素类型是数组类型，比如String[]、List<T>[]、T[]，总之带中括号的都是
    * 方法：getGenericComponentType，获取`[]`左边的类型
* TypeVariable: 表示泛型，比如 T，总之尖括号里边，没有？的都是
    * 方法：getBounds，获取运行时T的类型
* WildcardType: 表示泛型，比如?、? extends Number、? super Integer，总之尖括号里边有？的都是
    * 方法：getUpperBounds，获取通配符的上边界。比如 ? extends Number，上边界为Number。? super Integer 上边界为？，即Object
    * 方法：getLowerBounds，获取通配符的下边界。比如 ? extends Number ，下边界无法确定，即null。? super Integer 下边界为Integer

### 2.3.2 Class转Type
如上边的例子所示，总共有4种方法将Class转为Type，也即上边2.1，class文件解析时，有`// Signature`标识的四个地方。还少了一种，方法抛出的异常为泛型的也可获取到。

根据Class直接获取Type的六种方法（getGeneric*）：
* 父类 —— getGenericSuperclass()，例`AppleBasket.class.getGenericSuperclass();` 或 `appleBasket.getClass().getGenericSuperclass();`
* 类 —— getTypeParameters()，例`AppleBasket.class.getTypeParameters();`，比较特殊，直接返回子接口 TypeVariable
* 属性 —— getGenericType()，例`field.getGenericType();`
* 方法返回值 —— getGenericReturnType()，例`method.getGenericReturnType();`
* 方法参数 —— getGenericParameterTypes()，例`method.getGenericParameterTypes();`
* 方法异常 —— getGenericExceptionTypes()，例`method.getGenericExceptionTypes();`

### 2.3.3 获取到自己想要的Type子接口
下面这个例子诠释了一切，通过对Type的直接子接口间的转换，和Type的直接子类Class中有关Type的两个函数的解析，来说明问题。
```java
package com.arlen.common.paramtype;
 
import java.lang.reflect.Field;
import java.lang.reflect.GenericArrayType;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.lang.reflect.TypeVariable;
import java.lang.reflect.WildcardType;
import java.util.ArrayList;
import java.util.List;
 
import org.springframework.util.ReflectionUtils;
 
public class TypeTest<T> {
 
    private List objList = new ArrayList<>();
    private List<String> strList = new ArrayList<>();
    private List<T> paraList = new ArrayList<>();
    private List<? extends Number> numList = new ArrayList<>();
    private List<T[]> arrList = new ArrayList<>();
    private List<? super Number>[] numArr = new ArrayList[1];
 
    /**
     * 测试Type子接口间的互相转换及含义
     */
    public void testTypeSonInterface() {
        Field objList = ReflectionUtils.findField(TypeTest.class, "objList");
        // 返回值为Class类型，即List
        Type objType = objList.getGenericType();
 
        Field strList = ReflectionUtils.findField(TypeTest.class, "strList");
        // 返回值为ParameterizedType类型，即 List<String>
        Type strType = strList.getGenericType();
        // 返回值为Class类型，即List
        Type strRawTypes = ((ParameterizedType)strType).getRawType();
        // 返回值为Type数组，数组元素为Class类型，即String
        Type[] strArguArr = ((ParameterizedType)strType).getActualTypeArguments();
 
        Field paraList = ReflectionUtils.findField(TypeTest.class, "paraList");
        // 返回值为ParameterizedType类型，即 List<T>
        Type paraType = paraList.getGenericType();
        // 返回值为Class类型，即List
        Type paraRawTypes = ((ParameterizedType)paraType).getRawType();
        // 返回值为Type数组，数组元素为TypeVariable类型，即T
        Type[] paraArguArr = ((ParameterizedType)paraType).getActualTypeArguments();
        // 返回值为Type数组，数组元素为Class类型，即Object（因泛型擦除，T即Object）
        Type[] paraVarArr = ((TypeVariable)paraArguArr[0]).getBounds();
 
        Field numList = ReflectionUtils.findField(TypeTest.class, "numList");
        // 返回值为ParameterizedType类型，即List<? extends Number>
        Type numType = numList.getGenericType();
        // 返回值为Class类型，即List
        Type numRawTypes = ((ParameterizedType)numType).getRawType();
        // 返回值为Type数组，数组元素为WildCardType类型，即 ? extends Number
        Type[] numArguArr = ((ParameterizedType)numType).getActualTypeArguments();
        // 返回值为Type数组，数组元素为Class类型，即Number（获取通配类型的上边界）
        Type[] numUBoundArr = ((WildcardType)numArguArr[0]).getUpperBounds();
        // 返回值为Type数组，数组元素为Class类型，为null，因为下边界不可知
        Type[] numLBoundArr = ((WildcardType)numArguArr[0]).getLowerBounds();
 
        Field arrList = ReflectionUtils.findField(TypeTest.class, "arrList");
        // 返回值为ParameterizedType类型，即List<T[]>
        Type arrType = arrList.getGenericType();
        // 返回值为Class类型，即List
        Type arrRawTypes = ((ParameterizedType)arrType).getRawType();
        // 返回值为Type数组，数组元素为GenericArrayType类型，即T[]
        Type[] arrArguArr = ((ParameterizedType) arrType).getActualTypeArguments();
        // 返回值为TypeVariable类型，即T
        Type arrComType = ((GenericArrayType)arrArguArr[0]).getGenericComponentType();
        // 返回值为Type数组，数组元素为Class类型，即Object（因泛型擦除，T即Object）
        Type[] arrComBound = ((TypeVariable)arrComType).getBounds();
 
        Field numArr = ReflectionUtils.findField(TypeTest.class, "numArr");
        // 返回值为GenericArrayType类型，即List<? super Number>[]
        Type numArrType = numArr.getGenericType();
        // 返回值为ParameterizedType类型，即List<? super Number>
        Type numArrComType = ((GenericArrayType) numArrType).getGenericComponentType();
        // 返回值为Class类型，即List
        Type numArrRawTypes = ((ParameterizedType)numArrComType).getRawType();
        // 返回值为WildCardType类型，即<? super Number>
        Type[] numArrArguTypes = ((ParameterizedType)numArrComType).getActualTypeArguments();
        // 返回值为Type数组，数组元素为Class类型，即Object（也即?）
        Type[] numArrUBoundArr = ((WildcardType)numArrArguTypes[0]).getUpperBounds();
        // 返回值为Type数组，数组元素为Class类型，即Number
        Type[] numArrLBoundArr = ((WildcardType)numArrArguTypes[0]).getLowerBounds();
 
        Class ttt1 = new Type2Class().getRawTypeOfType(numArrComType);
        //Class ttt2 = new Type2Class().typeTrackToClass(numArrComType);
        System.out.println(ttt1);
    }
 
    /**
     * 测试Type子类的两个方法
     */
    public void testTypeSonClass() {
        // 1. getComponentType
        List<? extends Number>[] nums = new ArrayList[2];
        // 返回值为Class类型，即ArrayList[]
        Class<?> numsClass = nums.getClass();
        // 返回值为Class类型，即ArrayList
        Class<?> numsComType = nums.getClass().getComponentType();
 
        // 返回值为null，因TypeTest没有数组类型，也即getComponentType是针对变量才能使用的
        Class<?> clazz = TypeTest.class.getComponentType();
 
        // 2. getTypeParameters
        // 局部变量
        TypeTest<Student> test = new TypeTest<>();
        // 返回值为TypeVarialble数组，数组元素为TypeVarialble类型，即T
        TypeVariable[] tv = test.getClass().getTypeParameters();
        // 返回值为Type数组，数组元素为Class类型，即Object（因泛型擦除，T即Object）
        // 因为test.getClass，返回的是TypeTest的声明，局部定义的不会去拿，想拿也拿不到，并且
        Type[] boud = tv[0].getBounds();
 
        // 类属性
        TypeVariable[] tv1 = strList.getClass().getTypeParameters();
        // 返回值为Type数组，数组元素为Class类型，即Object（因泛型擦除，T即Object）
        // 因为strList.getClass，返回的是TypeTest的声明，属性定义的不会拿
        Type[] boud1 = tv1[0].getBounds();
 
        // 类
        // 返回值为TypeVarialble数组，数组元素为TypeVarialble类型，即T
        TypeVariable[] tv2 = TypeTest.class.getTypeParameters();
        // 返回值为Type数组，数组元素为Class类型，即Object（因泛型擦除，T即Object）
        Type[] boud2 = tv2[0].getBounds();
    }
 
    public static void main(String[] args) {
        new TypeTest<Student>().testTypeSonInterface();;
        new TypeTest<Student>().testTypeSonClass();
    }
 
}
```
* Type直接子接口间转换

  开始也讲了，获取Type的5种方法，例子中仅仅用了属性，即Field.getGenericType，其他都类似。

  从例子中可以看到，只要理解了4个直接子接口代表什么含义，就会用其接口中的方法在4个接口之间转换，拿到自己想要的泛型或数组类型

* Type直接子类Class相关

  首先，他是Class相关的方法，也就是从Class中去获取Type信息，看完写在最后，你就知道Class实际是没有任何泛型信息的，只有部分数组信息，故此，想通过Class的相关方法获取到实际的泛型，都是白搭。

  getTypeParameters方法，获取变量、属性、类的类泛型。也即获取class后边的<>中的内容，无法获取实际的泛型类型

  getComponentType方法，获取变量、属性的数组类型。也即获取[]左边的内容，返回值为Class，无法获取实际的泛型类型

### 2.3.4 Type转Class
很多情况下，通过反射获取泛型的实际类型，但是你想要拿到的是Class类型的，下面的例子讲了如何从Type转为Class。上边已经看了Type各个子接口之间的转换，代码也就是一层层的去剥离泛型，然后找到想要的东西。
```java
package com.arlen.common.paramtype;
 
import java.lang.reflect.Field;
import java.lang.reflect.GenericArrayType;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.lang.reflect.TypeVariable;
import java.lang.reflect.WildcardType;
import java.util.ArrayList;
import java.util.List;
 
import org.springframework.util.ReflectionUtils;
 
public class Type2Class {
 
    /**
     * 根据type获取最终的泛型类型，因为是示例性代码，不够完善<br>
     * TypeVariable只支持一种泛型，eg: class Test &lt;T&gt;<br>
     * WildCardType只支持上边界<br>
     * ParameterizedType只支持一种类型参数，eg: List&ltString&gt<br>
     */
    public static Class<?> getArgumentOfType(Type type) {
 
        if (type instanceof TypeVariable) {
            Type bounds = ((TypeVariable<?>)type).getBounds()[0];
            return getArgumentOfType(bounds);
        } else if (type instanceof WildcardType) {
            Type uBoundArr[] = ((WildcardType)type).getUpperBounds();
            if (uBoundArr.length > 0) {
                return getArgumentOfType(uBoundArr[0]);
            }
            return Object.class;
        } else if (type instanceof ParameterizedType) {
            Type argu = ((ParameterizedType)type).getActualTypeArguments()[0];
            return getArgumentOfType(argu);
        } else if (type instanceof GenericArrayType) {
            Type comp = ((GenericArrayType)type).getGenericComponentType();
            return getArgumentOfType(comp);
        } else {
            // (type instanceof Class)
            return (Class<?>)type;
        }
    }
 
    /**
     * 根据Type获取其对应的Class<br>
     * Type到了ParameterizedType之后，就可以直接取出其原生类了
     *
     */
    public static Class<?> getRawTypeOfType(Type type) {
        if (type instanceof TypeVariable) {
            // 已到最里层
            Type bounds = ((TypeVariable<?>)type).getBounds()[0];
            return (Class<?>)bounds;
        } else if (type instanceof WildcardType) {
            // 已到最里层
            Type uBoundArr[] = ((WildcardType)type).getUpperBounds();
            if (uBoundArr.length > 0) {
                return (Class<?>)uBoundArr[0];
            }
            return Object.class;
        } else if (type instanceof ParameterizedType) {
            // 获取原生类型
            Type argu = ((ParameterizedType)type).getRawType();
            return (Class<?>)argu;
        } else if (type instanceof GenericArrayType) {
            // 递归，直到找到ParameterizedType
            Type comp = ((GenericArrayType)type).getGenericComponentType();
            return getRawTypeOfType(comp);
        } else {
            // (type instanceof Class)
            return (Class<?>)type;
        }
    }
 
    private List<? super Number>[] numArr = new ArrayList[1];
 
    public static void main(String[] args) {
        Field numArr = ReflectionUtils.findField(Type2Class.class, "numArr");
        Type numArrType = numArr.getGenericType();
        // 输出：class java.lang.Object
        System.out.println(Type2Class.getArgumentOfType(numArrType));
        // 输出：interface java.util.List
        System.out.println(Type2Class.getRawTypeOfType(numArrType));
    }
}
```
 
### 2.4 利用Spring4.x的新特性——便捷的反射API
利用Spring的反射API，ResolvableType，一个工具类，底层实现原理还是上边讲的，只是加了缓存等，并且操作更方便。

## 写在最后：Type、Class、泛型的孽缘
1 泛型出现之前的类型

   没有泛型的时候，只有所谓的原始类型。此时，所有的原始类型都通过字节码文件类Class类进行抽象。Class类的一个具体对象就代表一个指定的原始类型。
 
2 泛型出现之后的类型

   泛型出现之后，扩充了数据类型。从只有原始类型扩充了参数化类型、类型变量类型、泛型限定的的参数化类型 (含通配符+通配符限定表达式)、泛型数组类型。
 
3 与泛型有关的类型不能和原始类型统一到Class的原因

3.1【产生泛型擦除的原因】

   本来新产生的类型+原始类型都应该统一成各自的字节码文件类型对象。但是由于泛型不是最初Java中的成分。如果真的加入了泛型，涉及到JVM指令集的修改，这是非常致命的。
 
3.2【Java中如何引入泛型】

   为了使用泛型的优势又不真正引入泛型，Java采用泛型擦除的机制来引入泛型。Java中的泛型仅仅是给编译器javac使用的，确保数据的安全性和免去强制类型转换的麻烦。但是，一旦编译完成，所有的和泛型有关的类型全部擦除。
 
3.3【Class不能表达与泛型有关的类型】

   因此，与泛型有关的参数化类型、类型变量类型、泛型限定的的参数化类型 (含通配符+通配符限定表达式)、泛型数组类型这些类型全部被打回原形，在字节码文件中全部都是泛型被擦除后的原始类型，并不存在和自身类型一致的字节码文件。所以和泛型相关的新扩充进来的类型不能被统一到Class类中。
 
4 与泛型有关的类型在Java中的表示

   为了通过反射操作这些类型以迎合实际开发的需要，Java就新增了ParameterizedType，GenericArrayType，TypeVariable 和WildcardType几种类型来代表不能被归一到Class类中的类型但是又和原始类型齐名的类型。
 
5 Type的引入：统一与泛型有关的类型和原始类型Class

5.1【引入Type的原因】

   为了程序的扩展性，最终引入了Type接口作为Class，ParameterizedType，GenericArrayType，TypeVariable和WildcardType这几种类型的总的父接口。这样实现了Type类型参数接受以上五种子类的实参或者返回值类型就是Type类型的参数。

5.2【Type接口中没有方法的原因】

   从上面看到，Type的出现仅仅起到了通过多态来达到程序扩展性提高的作用，没有其他的作用。因此Type接口的源码中没有任何方法。
