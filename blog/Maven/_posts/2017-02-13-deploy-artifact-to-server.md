---
layout: post
title: 如何上传自己的Jar包到Maven私服
tags: Maven
source: virgin
---

    我们经常会写一些通用的jar包或者工具类，因项目全用maven管理，为了让小伙伴能更方便的用到自己的jar包，需要将jar包上传到maven私服。总共有两种方式，一种是命令行方式，一种是页面上传。

## 一、命令行方式
### 1.1. 命令

```shell
# 打jar包
mvn clean install -Dmaven.test.skip=true
# 打sources包
mvn source:jar

# 上传
mvn deploy:deploy-file -DgroupId=com.arlen -DartifactId=mybatis-generator-plugin -Dversion=1.0.0 -Dpackaging=jar -Dfile=target/mybatis-generator-plugin.jar -Durl=http://192.168.226.68:8081/nexus/content/repositories/thirdparty/ -DrepositoryId=thirdparty
```

### 1.2. 配置权限
如果没有在本地maven配置文件中配置server信息，上边命令会报401错。可配置如下：

```xml
<server>
  <id>hk_reposity</id>
  <username>admin</username>
  <password>admin123</password>
</server>
<server>
  <id>releases</id>
  <username>admin</username>
  <password>admin123</password>
</server>
<server>
  <id>snapshots</id>
  <username>admin</username>
  <password>admin123</password>
</server>
<server>
  <id>thirdparty</id>
  <username>admin</username>
  <password>admin123</password>
</server>
```

### 1.3. 确保私服已开启上传权限
如果都配置了，还报Bad Request 400错，请确保私服上thirdparty库的发布权限是否开启，如果没有请开启。

![查看私服上传权限]({{site.url}}/assets/img-blog/Maven/3rd-party-auth.png)

### 1.4 更正确的做法
在Pom文件中配置`distributionManagement`，直接执行`mvn deploy`即可，他会把jar包和sources包自动都上传到远程服务器。没必要输那么多复杂的命令了，因为Pom文件中都已经定义好了

最简单的配置如下，如果要更详细的配置，上网搜即可
```xml
<distributionManagement>
    <repository>
        <uniqueVersion>false</uniqueVersion>
        <id>thirdparty</id>
        <name>Third party</name>
        <url>http://192.168.226.68:8081/nexus/content/repositories/thirdparty/</url>
        <layout>default</layout>
    </repository>
</distributionManagement>
```

命令：

```shell
mvn deploy -Dmaven.test.skip=true
```

## 二、页面上上传
### 2.1 登录私服页面
http://192.168.226.68:8081/nexus/,右上角登录

![登陆链接]({{site.url}}/assets/img-blog/Maven/nexus-login.png)

### 2.2 进入上传页面
右侧--》Repositories--》3rd party，点击。可以在下方的Tab页 Browse Storage中看到我们上传的第三方jar包

![登陆链接]({{site.url}}/assets/img-blog/Maven/3rd-party-explorer.png)
 
### 2.3 上传自己的jar包
1. Tab页Artifact Upload中，完善jar信息。可以通过两种方式定义jar包。
    * 一种直接通过pom文件，nexus自己读取对应的groupId、artifactId。
    * 另一种是自己填，因为懒所以直接读pom
2. 选择Jar包，点Add Artifact，然后上传

![上传jar包]({{site.url}}/assets/img-blog/Maven/3rd-party-upload.png)
 
上传成功后就可以看到了

![上传jar包]({{site.url}}/assets/img-blog/Maven/3rd-party-upload-success.png)