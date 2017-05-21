---
layout: post
title: Maven 包冲突管理
tags: Maven
source: virgin
---

    使用Maven依赖包，经常会碰到烦人的 ClassNotFoundException 或 NoSuchMethodError，很是烦人，所以搞清楚Maven对依赖冲突的处理机制很有必要。先简单记录下，后面再补充。

Maven采用“最近获胜策略（nearest wins strategy）”的方式处理依赖冲突，即如果一个项目最终依赖于相同artifact的多个版本，在依赖树中离项目最近的那个版本将被使用。让我们来看看一个实际的例子。

![包冲突示例]({{site.url}}/assets/img-blog/Maven/jar-conflict.png)

上边截图的意思是，下面的全部忽略，用最近的mybatis:3.1.1
