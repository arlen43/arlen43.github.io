---
layout: post
title: Mybatis generator 开发自己的插件
tags: Mybatis Plugin
source: virgin
---
    Mybatis generator着实减少了平时开发中的很多工作量，但是每次生成的东西必须的修修改改才可以用，何不让其生成的东西一次性就可以使用呢？

参照官网链接：http://generator.sturgeon.mopaas.com/reference/extending.html
参照小码各博客：http://www.jianshu.com/u/231b43e2c05f

我这里用的是mybatis-generator-maven-plugin，用eclipse或者直接跑mybatis-generator-core源码的，可以借鉴插件的写法和编译等。

### 一、创建项目 mybatis-generator-plugin

创建一个maven骨架工程，没有合适的，就先创建webapp，然后手动改为jar项目
```shell
mvn archetype:generate -DgroupId=com.arlen -DartifactId=mybatis-generator-plugin  -DarchetypeArtifactId=maven-archetype-webapp
```

手动修改pom文件packaging为jar方式，添加mybatis-generator-core依赖，添加maven-compiler-plugin编译插件。
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.arlen</groupId>
    <artifactId>mybatis-generator-plugin</artifactId>
    <packaging>jar</packaging>
    <version>1.0.0</version>
    <name>mybatis-generator-plugin</name>
 
    <dependencies>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>3.8.1</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-core</artifactId>
            <version>1.3.2</version>
        </dependency>
    </dependencies>
 
    <build>
        <finalName>mybatis-generator-plugin</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.3</version>
                <configuration>
                    <source>1.7</source>
                    <target>1.7</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </build>
 
</project>
```

### 二、编写自己的插件
编写自己的插件，可以通过对PluginAdapter的扩展来实现，基本你想要的功能都能实现。我总共开发了五个插件，分别为 生成自定义注释插件、分页插件、批量操作插件、字典项插件、重命名java client类名

下面以最简单的自定义注释插件为例
```java
public class DBColumnCommentPlugin extends PluginAdapter {
 
    /**
     * 无任何参数，无需校验
     */
    public boolean validate(List<String> warnings) {
        return true;
    }
 
    /**
     * 生成字段时调用方法
     */
    @Override
    public boolean modelFieldGenerated(Field field, TopLevelClass topLevelClass, IntrospectedColumn introspectedColumn,
            IntrospectedTable introspectedTable, ModelClassType modelClassType) {
        String comment = introspectedColumn.getRemarks();
        if (comment != null && comment.trim().length() > 0) {
            field.addJavaDocLine("/**");
            field.addJavaDocLine(" * " + introspectedColumn.getRemarks());
            field.addJavaDocLine(" */");
        }
        return super.modelFieldGenerated(field, topLevelClass, introspectedColumn, introspectedTable, modelClassType);
    }
 
    @Override
    public boolean modelGetterMethodGenerated(Method method, TopLevelClass topLevelClass,
            IntrospectedColumn introspectedColumn, IntrospectedTable introspectedTable, ModelClassType modelClassType) {
        String comment = introspectedColumn.getRemarks();
        if (comment != null && comment.trim().length() > 0) {
            method.addJavaDocLine("/**");
            method.addJavaDocLine(" * @return " + introspectedColumn.getActualColumnName() + " " + introspectedColumn.getRemarks());
            method.addJavaDocLine(" */");
        }
        return super.modelGetterMethodGenerated(method, topLevelClass, introspectedColumn, introspectedTable, modelClassType);
    }
 
    @Override
    public boolean modelSetterMethodGenerated(Method method, TopLevelClass topLevelClass,
            IntrospectedColumn introspectedColumn, IntrospectedTable introspectedTable, ModelClassType modelClassType) {
        String comment = introspectedColumn.getRemarks();
        if (comment != null && comment.trim().length() > 0) {
            Parameter param = method.getParameters().get(0);
            method.addJavaDocLine("/**");
            method.addJavaDocLine(" * @param " + param + " " + introspectedColumn.getRemarks());
            method.addJavaDocLine(" */");
        }
        return super.modelSetterMethodGenerated(method, topLevelClass, introspectedColumn, introspectedTable, modelClassType);
    }
}
```

### 三、打包自己的插件
写好的插件要打包，让mybatis-generator插件来依赖，在pom目录打包
```shell
mvn clean install -Dmaven.test.skip=true
```

### 四、将自己的插件（jar）安装到本地仓库
为了让maven很方便的依赖，而不是把jar包拷到mybatis-generator源码里再打包一次，我们讲自己的插件安装到本地仓库
```shell
mvn install:install-file -Dfile=target/mybatis-generator-plugin.jar -DgroupId=com.arlen -DartifactId=mybatis-generator-plugin -Dversion=1.0.0 -Dpackaging=jar
```
 
### 五、让mybatis-generator插件可以找到自己写的插件
为了让mybatis-generator的maven插件可以使用自己写的插件，需要在自己项目的pom文件中添加
```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/maven-v4_0_0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>sample</groupId>
    <artifactId>sample</artifactId>
    <packaging>war</packaging>
    <version>1.0</version>
    <name>sample Maven Webapp</name>
    <url>http://maven.apache.org</url>
    <!-- ...你自己项目的一堆依赖和配置... -->
    <build>
        <finalName>sample</finalName>
        <plugins>
            <plugin>
                <groupId>org.mybatis.generator</groupId>
                <artifactId>mybatis-generator-maven-plugin</artifactId>
                <version>1.3.2</version>
                <configuration>
                    <configurationFile>src/main/resources/config/generator/generator-sample.xml</configurationFile>
                    <verbose>true</verbose>
                    <overwrite>true</overwrite>
                </configuration>

                <dependencies>
                    <dependency>
                        <groupId>com.arlen</groupId>
                        <artifactId>mybatis-generator-plugin</artifactId>
                        <version>1.0.0</version>
                    </dependency>
                </dependencies>

            </plugin>
        </plugins>
    </build>
</project>
```

### 六、在项目中配置自己的插件
下面的配置中总共配了6个插件。分别是：
* 在Domain类上生成数据库中对应的注释——DBColumnCommentPlugin
* 在Domain类的字段上生成对应的字典项注解和中英文字段——DictionaryAnnotationPlugin
* 在Example类和SqlMap文件中生成分页相关参数——PagePlugin
* 在JavaClient中和SqlMap文件中生成批量增删改内容——BatchPlugin
* 重命名JavaClient类为 I***Dao——RenameClientClassPlugin
* 重命名Example类为***Query——RenameExampleClassPlugin
对应的类见我的代码库。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE generatorConfiguration PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN" "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd" >
<generatorConfiguration>
    <classPathEntry
        location="D:\maven\repository\mysql\mysql-connector-java\5.1.19\mysql-connector-java-5.1.19.jar" />
    <context id="DB2Tables" targetRuntime="MyBatis3" defaultModelType="flat">
 
        <property name="javaFileEncoding" value="UTF-8" />
        <property name="beginningDelimiter" value="`" />
        <property name="endingDelimiter" value="`" />
 
        <plugin type="com.arlen.generator.plugin.DBColumnCommentPlugin" />
        <plugin type="com.arlen.generator.plugin.DictionaryAnnotationPlugin" />
        <plugin type="com.arlen.generator.plugin.PagePlugin" />
        <plugin type="com.arlen.generator.plugin.BatchPlugin" />
        <plugin type="com.arlen.generator.plugin.RenameClientClassPlugin">
            <property name="searchRegex" value="Mapper" />
            <property name="replaceString" value="Dao" />
            <property name="prefix" value="I" />
        </plugin>
        <plugin type="org.mybatis.generator.plugins.RenameExampleClassPlugin">
            <property name="searchString" value="Example" />
            <property name="replaceString" value="Query" />
        </plugin>
 
        <commentGenerator>
            <property name="suppressDate" value="true"/>
            <property name="suppressAllComments" value="true" />
        </commentGenerator>
 
        <jdbcConnection driverClass="com.mysql.jdbc.Driver"
            connectionURL="jdbc:mysql://127.0.0.1:3306/greenpass"
            userId="root"
            password="123456"
            />
 
        <javaTypeResolver>
            <property name="forceBigDecimals" value="true" />
            <!-- 默认false，把JDBC DECIMAL 和 NUMERIC 类型解析为 Integer true，把JDBC DECIMAL 和 NUMERIC 类型解析为java.math.BigDecimal -->
        </javaTypeResolver>
 
        <javaModelGenerator targetPackage="com.hongkun.greenpass.member.order.domain"
            targetProject="./src/main/java">
            <property name="enableSubPackages" value="true" />
            <!-- set的时候是否去除尾部空格 -->
            <property name="trimStrings" value="true" />
        </javaModelGenerator>
 
        <sqlMapGenerator targetPackage="reportorder-gen" targetProject="./src/main/resources/mybatis">
          <property name="enableSubPackages" value="true" />
        </sqlMapGenerator>
 
        <javaClientGenerator type="XMLMAPPER"
          targetPackage="com.hongkun.greenpass.member.order.dao" targetProject="./src/main/java">
          <property name="enableSubPackages" value="true" />
        </javaClientGenerator>
 
        <table tableName="order_extra_fee" alias="oef" domainObjectName="OrderExtraFee"
            enableDeleteByExample="false" enableUpdateByExample="false">
            <generatedKey column="id" sqlStatement="JDBC" identity="true" />
        </table>
 
    </context>
</generatorConfiguration>
```

### 七、运行mybatis-generator-maven-plugin生成代码
右击自己的项目-->Run as-->Run configurations...-->选择自己的项目-->Goals填mybatis-generator:generate-->运行

![Example相关类图]({{site.url}}/assets/img-blog/Mybatis/eclipse-run-generator.png)

然后代码就生成了。

或者直接用maven直接打包自己的maven项目。

### 八、让别的小伙伴一起来用你的插件吧——上传jar包到maven私服
1. 命令行方式
```shell
mvn deploy:deploy-file -DgroupId=com.arlen -DartifactId=mybatis-generator-plugin -Dversion=1.0.0 -Dpackaging=jar -Dfile=target/mybatis-generator-plugin.jar -Durl=http://192.168.226.68:8081/nexus/content/repositories/thirdparty/ -DrepositoryId=thirdparty
```
首先得确保私服开了上传jar包的权限，其次，本地maven配置文件中得配置对应的用户名密码，祥见 Maven--》如何上传jar包到Maven私服
2. 页面上传
http://192.168.226.68:8081/nexus/
登录私服页面，然后上传第三方jar包，见 Maven--》如何上传jar包到Maven私服 文章，或者直接百度，很多详细的。
 
