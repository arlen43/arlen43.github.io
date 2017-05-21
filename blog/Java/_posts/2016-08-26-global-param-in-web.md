---
layout: post
title: WebApplicationContext下的全局变量配置
tags: Java
source: virgin
---

    在java web开发中，经常会碰到在JSP中访问一些变量的情况。小伙伴们肯定都知道，在java代码中，我们可以通过maven管理不同环境的配置文件，实现配置的环境化，那么也就可以想办法，使用maven管理的配置文件，来让前端页面也环境化。具体用到的方案就是，在Web.xml中，将maven编译后的全局配置加载至WebApplicationContext中。

web.xml配置
```xml
<!-- 全局变量 -->
<listener>
    <description>ServletContextListener</description>
    <listener-class>com.arlen.common.listener.ConfigListener</listener-class>
</listener>
```

ConfigListener.java
```java
public class ConfigListener implements ServletContextListener {
 
    private final static Logger logger = LoggerFactory.getLogger(ConfigListener.class);
 
    public void contextInitialized(ServletContextEvent sce) {
 
        InputStream inputStream = null;
        try {
            Resource ss = new ClassPathResource("baseconfig.properties");
            inputStream = ss.getInputStream();
 
            Properties props = new Properties();
            props.load(inputStream);
            Iterator<Entry<Object, Object>> it = props.entrySet().iterator();
            while (it.hasNext()) {
                Entry<Object, Object> entry = it.next();
                sce.getServletContext().setAttribute(entry.getKey().toString(), entry.getValue());
                logger.info("加载WebApplicationContext全局配置：");
                logger.info(entry.getKey() + ": " + entry.getValue());
            }
        } catch (IOException ex) {
            logger.error("加载WebApplicationContext全局配置失败", ex);
        } finally {
            if (null != inputStream) {
                try {
                    inputStream.close();
                } catch (IOException e) {
                    logger.warn("加载WebApplicationContext全局配置，关闭文件流失败", e);
                }
            }
        }
    }
 
    public void contextDestroyed(ServletContextEvent sce) {
 
    }
}
```
