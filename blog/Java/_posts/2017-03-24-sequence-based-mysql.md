---
layout: post
title: 基于数据库（不使用数据库函数）实现sequence
tags: Java Sequence
source: virgin
---

*分布式全局sequence生成貌似一直都没有一个完美的解决方案，结合之前在‘架构师之路’看到的五种方式，自己基于数据库，用java实现了一个全局sequence生成的工具。其他的方案，数据库自增太局限，数据库分布式不适用；数据库函数的不便于迁移、数据库压力大；单机生成的又太长；uuid识别度太差；所以用了其中一种方案，我觉得还可以接受的。*

## 功能描述
1. 可以动态加载一个新的Sequence；如果数据库不存在，插入一条新纪录；如果存在，则从数据库加载；
2. 每隔一个步长，更新数据库一次。步长内，采用递增方式产生。主要用于防止应用停止又重启，保存最大的id；
3. 利用Spring和Mybatis操作数据库，但是工具类不作为Spring的Bean

## 具体实现
### 1 表结构
备份保存最后的id，用于应用重启等
```sql
DROP TABLE IF EXISTS `prd_base_sequence`;
CREATE TABLE `prd_base_sequence` (
  `seq_key` tinyint(4) NOT NULL DEFAULT '0' COMMENT '业务类型，0：product；1：line；2：user',
  `start_id` int(11) NOT NULL DEFAULT '1000000' COMMENT '起始id',
  `step_by` int(11) NOT NULL DEFAULT '10' COMMENT '步长',
  PRIMARY KEY (`seq_key`)
) ENGINE=InnoDB AUTO_INCREMENT=1000 DEFAULT CHARSET=utf8 COMMENT='全局sequence生成表';
```

### 2 对外的SequenceUtil
对外提供的Util，可以根据业务Key来获取对应的Sequence
```java
package com.arlen.common.sequence;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
 
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
 
public class SequenceUtil {
 
    private final Logger logger = LoggerFactory.getLogger(SequenceUtil.class);
 
    private final static Map<Byte, BaseSequence> sequenceMap = new ConcurrentHashMap<Byte, BaseSequence>();
 
    private SequenceUtil() {}
 
    /**
     * 根据业务key获取对应的自增序列
     * @param key
     * @return 序列号
     * @since CodingExample　Ver(编码范例查看) 1.1
     * @author arlen
     * @throws SequenceGenerateException
     */
    public synchronized static int getNextId(Byte key) {
        try {
            if (!sequenceMap.containsKey(key)) {
                sequenceMap.put(key, new BaseSequence(key));
            }
            return sequenceMap.get(key).getNextId();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

### 3 实际的Sequence实体
Sequence产生实体
```java
package com.arlen.common.sequence;
 
import org.springframework.context.ApplicationContext;
import com.arlen.common.sequence.dao.IBaseSequenceDao;
import com.arlen.common.spring.SpringContextUtil;

public class BaseSequence {
    private final static int START = 1000000;
    private final static int STEP = 10;
    /**
     * 业务类型，0：product；1：line；2：user
     */
    private Byte seqKey;
    /**
     * 起始id
     */
    private Integer startId;
    /**
     * 步长
     */
    private Integer stepBy;
    /**
     * 下个id
     */
    private int nextId;
 
    private IBaseSequenceDao dao;
 
    public BaseSequence() {
    }
 
    public BaseSequence(byte seqKey) {
        this.seqKey = seqKey;
        checkAndReload();
    }
 
    public int getNextId() {
        if (this.nextId == this.startId) {
            this.startId += this.stepBy;
            getDao().updateByPrimaryKeySelective(this);
        }
        return nextId++;
    }
 
    private void checkAndReload() {
        BaseSequence sequence = getDao().selectByPrimaryKey(this.seqKey);
        if (sequence == null) {
            this.nextId = START;
            this.startId = START + STEP;
            this.stepBy = STEP;
            getDao().insert(this);
        } else {
            this.nextId = sequence.getStartId();
            this.stepBy = sequence.getStepBy();
            this.startId = sequence.getStartId() + this.stepBy;
            getDao().updateByPrimaryKeySelective(this);
        }
    }
 
    private IBaseSequenceDao getDao() {
        if (dao == null) {
            // 如果在Web环境下，可以用ContextLoader.getCurrentWebApplicationContext();，我这里是写了一个简单的工具类，见文章最后
            ApplicationContext applicationContext = SpringContextUtil.getApplicationContext();
            if (applicationContext == null) {
                throw new RuntimeException("Init sequence faild, not in web environment. Get WebApplication failed");
            }
            dao = applicationContext.getBean(IBaseSequenceDao.class);
            if (dao == null) {
                throw new RuntimeException("Init sequence faild, IBaseSequenceDao dose not init");
            }
        }
        return dao;
    }
 
    public Byte getSeqKey() {
        return seqKey;
    }
    public void setSeqKey(Byte seqKey) {
        this.seqKey = seqKey;
    }
    public Integer getStepBy() {
        return stepBy;
    }
    public void setStepBy(Integer stepBy) {
        this.stepBy = stepBy;
    }
    public Integer getStartId() {
        return startId;
    }
    public void setStartId(Integer startId) {
        this.startId = startId;
    }
}
```

### 4 Mybatis SqlMap文件
Mybatis SqlMap文件必须得在自己的Spring配置中扫描到
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.arlen.common.sequence.dao.IBaseSequenceDao">
  <resultMap id="BaseResultMap" type="com.arlen.common.sequence.BaseSequence">
    <id column="t_seq_key" jdbcType="TINYINT" property="seqKey" />
    <result column="t_start_id" jdbcType="INTEGER" property="startId" />
    <result column="t_step_by" jdbcType="INTEGER" property="stepBy" />
  </resultMap>
  <sql id="Base_Column_List">
    t.seq_key as t_seq_key, t.start_id as t_start_id, t.step_by as t_step_by
  </sql>
  <select id="selectByPrimaryKey" parameterType="java.lang.Byte" resultMap="BaseResultMap">
    select
    <include refid="Base_Column_List" />
    from prd_base_sequence t
    where t.seq_key = #{seqKey,jdbcType=TINYINT}
  </select>
  <insert id="insert" parameterType="com.arlen.common.sequence.BaseSequence">
    insert into prd_base_sequence (seq_key, start_id, step_by
      )
    values (#{seqKey,jdbcType=TINYINT}, #{startId,jdbcType=INTEGER}, #{stepBy,jdbcType=INTEGER}
      )
  </insert>
  <update id="updateByPrimaryKeySelective" parameterType="com.arlen.common.sequence.BaseSequence">
    update prd_base_sequence
    <set>
      <if test="startId != null">
        start_id = #{startId,jdbcType=INTEGER},
      </if>
      <if test="stepBy != null">
        step_by = #{stepBy,jdbcType=INTEGER},
      </if>
    </set>
    where seq_key = #{seqKey,jdbcType=TINYINT}
  </update>
</mapper>
```

### 5 测试
使用JUnit测试，注意是基于Spring环境。100个线程并发获取。
```java
public class OrderExtraFeeTest extends BaseTest {
 
    @Test
    public void sequence() {
        System.out.println("========================================");
        //System.out.println(SequenceUtil.getNextId((byte)3));
        for (int i = 0; i < 100; i ++) {
            new Thread(new Runnable() {
 
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName() + "loop in");
                    try {
                        System.out.println(SequenceUtil.getNextId((byte)4));
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("========================================");
    }
}
```

### 6 业务系统中的使用
我的做法是定义了一个枚举类，然后各自业务系统用自己的即可，当然还有其他方式，就不一一列举了
```java
public enum SequenceEnum {
 
    PRODUCT((byte)1),LINE((byte)1),UC((byte)1);
 
    private byte value;
 
    private SequenceEnum(byte value) {
        this.value = value;
    }
 
    public int getID() {
        return SequenceUtil.getNextId(this.value);
    }
}
```

### 附：SpringContextUtil
还有很多方式，可以参照网上，如何获取ApplicationContext。注意下面的类需要被Spring扫描到
```java
@Component
public class SpringContextUtil implements ApplicationContextAware {
 
    private static ApplicationContext applicationContext;
 
    @Override
    public void setApplicationContext(ApplicationContext ac) throws BeansException {
        applicationContext = ac;
    }
 
    /**
     * 获取applicationContext
     *
     * @return applicationContext
     */
    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }
 
    /**
     * 根据id获取Bean
     *
     * @param id
     * @return Bean
     */
    public static Object getBeanById(String id) {
        return applicationContext.getBean(id);
    }
 
    /**
     * 根据class获取Bean
     *
     * @param clazz
     * @return Bean
     */
    public static Object getBeanByClass(Class<?> clazz) {
        return applicationContext.getBean(clazz);
    }
 
    /**
     * 根据class获取所有的Bean
     *
     * @param clazz
     * @return Map
     */
    public static Map<String, ?> getBeansByClass(Class<?> clazz) {
        return applicationContext.getBeansOfType(clazz);
    }
}
```
