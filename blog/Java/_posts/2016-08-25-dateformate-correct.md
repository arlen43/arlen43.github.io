---
layout: post
title: DateFormat应该这样写
tags: Java
source: virgin
---

因DateFormat类并不是线程安全的，所以在可能并发的情况下不能直接使用。

### 可以借助ThreadLocal来实现多线程安全性。
```java
private final static ThreadLocal<DateFormat> df = new ThreadLocal<DateFormat>() {
    @Override
    protected DateFormat initialValue() {
        return new SimpleDateFormat("yyyy-MM-dd");
    }
};
```

具体使用如下：
```java
System.out.println(df.get().format(new Date()));
```

### 或者使用Joda-Time代替，它是一个很棒的开源JDK的日期和日历API的替代品。使用如下

```java
private final DateTimeFormatter fmt = DateTimeFormat.forPattern("yyyyMMdd");
public void test() {
    DateTime d = fmt.parseDateTime("20160808");
}
```