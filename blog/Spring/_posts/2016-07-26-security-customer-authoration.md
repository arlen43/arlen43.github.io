---
layout: post
title: Spring Security —— 定义自己的认证方式
tags: Spring Security
source: virgin
---

    因为对spring-security了解甚少，也是初步使用，仅仅实现一个简单的需求。当认证时，不要使用Spring默认的用户名密码认证方式，而是使用系统现有的用户体系认证。

做法有几种，有复杂的，也有简单的。这里只说最简单的两种实现。

Spring 认证方式的配置
```xml
<!-- 拦截 -->
<http>
  <intercept-url pattern="/**" access="ROLE_USER" />
  <http-basic />
</http>
<!-- 认证 -->
<authentication-manager>
  <authentication-provider>
    <user-service>
      <user name="jimi" password="jimispassword" authorities="ROLE_USER, ROLE_ADMIN" />
      <user name="bob" password="bobspassword" authorities="ROLE_USER" />
    </user-service>
  </authentication-provider>
</authentication-manager>
```
如上，authentication-manager用来认证，其配置了用户名、密码以及角色。如果要修改，有两种方式。一是直接指定自己的authentication-provider；二是指定自己的user-service。

### 指定自己的authentication-provider
```xml
<authentication-manager>
    <authentication-provider ref="exAuthenticationProvider" />
</authentication-manager>
```
```java
@Component
public class ExAuthenticationProvider implements AuthenticationProvider {
 
    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {
        List<SimpleGrantedAuthority> list = new ArrayList<SimpleGrantedAuthority>();
        list.add(new SimpleGrantedAuthority("ROLE_USER"));
        // 此处可以写自己的认证逻辑，即从数据库查出自己的用户名密码
        User details = new User("admin", "admin", list);
 
        UserDetails userDetails = (UserDetails)details;
 
        UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
                userDetails, authentication.getCredentials(),userDetails.getAuthorities());
        return result;
    }
 
    @Override
    public boolean supports(Class<?> arg0) {
        return true;
    }
}
```
### 指定自己的user-service
```xml
<authentication-manager>
    <authentication-provider user-service-ref="exUserDetailsService">
        <password-encoder hash="sha" />
    </authentication-provider>
</authentication-manager>
```
```java
@Component
public class ExUserDetailsService implements UserDetailsService {
 
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        List<SimpleGrantedAuthority> list = new ArrayList<SimpleGrantedAuthority>();
        list.add(new SimpleGrantedAuthority("ROLE_USER"));
        User details = new User("admin", "d033e22ae348aeb5660fc2140aec35850c4da997", list);
 
        UserDetails userDetails = (UserDetails)details;
        return userDetails;
    }
 
    public static void main(String[] args) {
        System.out.println(SHA1Util.getSHA1("admin").toLowerCase());
    }
}
```
这里看到认证的配置当中使用了SHA-1加密，所以在ExUserDetailsService中，得设置密文。也就是数据库中放的是密文。

一个实际的例子
```java
@Component
public class ExUserDetailsService implements UserDetailsService {
 
    private final static Logger logger = LoggerFactory.getLogger(ExUserDetailsService.class);
 
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock r = rwLock.readLock();
    private final Lock w = rwLock.writeLock();
 
    private final static Map<String, String> userPassMap = new HashMap<String, String>();
 
    @Resource(name = "userDaoImpl")
    private IUserDao userDao;
 
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        logger.info("用户"+username+"认证开始");
        List<SimpleGrantedAuthority> authoritylist = new ArrayList<SimpleGrantedAuthority>();
        authoritylist.add(new SimpleGrantedAuthority("ROLE_USER"));
 
        String password = "";
        try {
            r.lock();
            password = userPassMap.get(username);
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        } finally {
            r.unlock();
        }
 
        if (StringUtils.isEmpty(password)) {
            EntrustCompany entrustCompany = this.userDao.queryEntrustCompanyInfoByPhoneNum(username);
            // 用户不存在，返回默认。可以用admin/greenpass登录
            if (null == entrustCompany) {
                logger.warn("用户"+username+"不存在");
                return new User("admin", "0e750e74a21c5c47bff6640e35bf1deae0af3e74", authoritylist);
            }
            password = entrustCompany.getPassword().toLowerCase();
        }
 
        try {
            w.lock();
            userPassMap.put(username, password);
        } catch (Exception e) {
            logger.error(e.getMessage(), e);
        } finally {
            w.unlock();
        }
        return new User(username, password, authoritylist);
    }
 
    public static void main(String[] args) {
        System.out.println(SHA1Util.getSHA1("greenpass").toLowerCase());
    }
}
```

使用Spring Security其实也就是一个个过滤器，关键日志如下
```java
/chb-api/importOrder.html at position 1 of 9 in additional filter chain; firing Filter: 'SecurityContextPersistenceFilter'
/chb-api/importOrder.html at position 2 of 9 in additional filter chain; firing Filter: 'WebAsyncManagerIntegrationFilter'
/chb-api/importOrder.html at position 3 of 9 in additional filter chain; firing Filter: 'BasicAuthenticationFilter'
/chb-api/importOrder.html at position 4 of 9 in additional filter chain; firing Filter: 'RequestCacheAwareFilter'
/chb-api/importOrder.html at position 5 of 9 in additional filter chain; firing Filter: 'SecurityContextHolderAwareRequestFilter'
/chb-api/importOrder.html at position 6 of 9 in additional filter chain; firing Filter: 'AnonymousAuthenticationFilter'
/chb-api/importOrder.html at position 7 of 9 in additional filter chain; firing Filter: 'SessionManagementFilter'
/chb-api/importOrder.html at position 8 of 9 in additional filter chain; firing Filter: 'ExceptionTranslationFilter'
/chb-api/importOrder.html at position 9 of 9 in additional filter chain; firing Filter: 'FilterSecurityInterceptor'
```

官方文档翻译不错的一个帖子：http://haohaoxuexi.iteye.com/blog/2157769

