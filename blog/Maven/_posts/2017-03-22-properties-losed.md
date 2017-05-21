---
layout: post
title: Maven打包main/java包下的properties文件缺失
tags: Maven
source: virgin
---

    写了一个工具包，在Java类的当前目录里写了一个properties文件，然后本地代码跑没有任何问题，Maven打包上传到私服后，找不到该properties文件，打开Jar包中，也没有，奇了怪了。

不用奇怪，将properties文件加到pom的resources中即可。比如像下面这样

```xml
<build>
<!-- ... -->
<resources>
    <resource>
        <directory>src/main/java</directory>
        <includes>
            <include>**/*.properties</include>
        </includes>
    </resource>
</resources>
<!-- ... -->
</build>
```
