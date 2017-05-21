---
layout: post
title: Maven打包filter配置
tags: Maven
source: virgin
---

    比如本系统调用系统A 10个服务，然后properties文件中配置了10个url，每个url有相同的ip:port，每次修改都会很麻烦。将相同的ip:port抽出来单独配置，其他用诸如${hostname}代替，然后让maven的filter自动替换即可。更有甚者，可以指定不同的文件，让maven编译时用properties中的值取替换。

### 一个完整的例子
```xml
<!-- 配置不同的profile，用于不同环境打包，此处只放了dev -->
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <ENV>dev</ENV>
        </properties>
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>
    </profile>
</profiles>

<!-- ... -->

<build>
    <finalName>sample</finalName>
    <!-- 指定maven filter所使用的源，即用哪个文件中的变量去替换别的文件中的${}通配符 -->
    <filters>
        <filter>src/main/resources/config/resource/${ENV}/config.properties</filter>
    </filters>
    <resources>
        <!-- filtering为true指定，该文件夹下的文件需要过滤，看到与上边的“源”有相同文件，都是没问题的，同一文件中也可以过滤替换 -->
        <resource>
            <directory>src/main/resources/config/resource/${ENV}</directory>
            <filtering>true</filtering>
        </resource>
        <!-- 过滤一个外部指定的文件 -->
        <resource>
            <directory>src/main/resources/PriceRule</directory>
            <filtering>true</filtering>
            <!-- 指定要过滤哪些文件，可指定通配符 -->
            <includes>
                <include>vendorPriceRules.xml</include>
            </includes>
            <!-- 指定过滤后的输出目录，没有指定，默认输出到class目录，相对ROOT而言，也就是class的根目录。这里还生成到相同的目录 -->
            <targetPath>PriceRule/addPriceRule</targetPath>
        </resource>
        <resource>
            <!-- 除了ENV目录下的其他文件也是resources -->
            <directory>src/main/resources</directory>
            <excludes>
                <exclude>**/test/*.*</exclude>
                <exclude>**/sit/*.*</exclude>
                <exclude>**/staging/*.*</exclude>
                <exclude>**/prd/*.*</exclude>
            </excludes>
        </resource>
    </resources>
</build>
```

