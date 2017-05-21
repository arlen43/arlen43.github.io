---
layout: post
title: Spring Interceptor —— URI并发拦截器
tags: Spring Interceptor
source: virgin
---

    最近做供应商API对接时，因为对方数据量比较大，采用的方案是分批次导入。那么对方分批次就意味着我们这边的并发压力，为了不让弱不禁风的tomcat挂掉，只能采用拦截器的办法，为每个URI，同一个用户同时的并发数强制限制为5。

拦截器java代码：
```java
public class ConcurrentNumberInterceptor extends BaseController implements HandlerInterceptor {
 
    private final static int MAX_THREAD_SIZE = 2;
    private final static String DIV_STR = "_";
    private final static String LOCKER = "locker";
    private final static Logger logger = LoggerFactory.getLogger(ConcurrentNumberInterceptor.class);
 
    private final static ConcurrentHashMap<String, List<String>> uriMap = new ConcurrentHashMap<String, List<String>>();
 
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
 
        String requestPath = request.getRequestURI();
        EntrustCompany loginUser = getLoginUserInfo(request);
        if (loginUser == null) {
            logger.info("并发拦截开始，当前无登录用户信息，返回");
            return true;
        }
        String orgCode = getLoginUserInfo(request).getOrgCode();
        logger.info("企业["+orgCode+"]请求["+requestPath+"]并发拦截开始");
        String key = orgCode + DIV_STR + requestPath;
        synchronized (LOCKER) {
            if (!uriMap.containsKey(key)) {
                logger.info("["+requestPath+"]当前无线程运行");
                uriMap.put(key, new ArrayList<String>());
            } else {
                logger.info("["+requestPath+"]当前运行线程"+JSON.toJSONString(uriMap.get(key))+"");
                if (uriMap.get(key).size() >= MAX_THREAD_SIZE) {
                    String resStr = JSON.toJSONString(new ResponseVo(false, "当前并发数超过限定值["+MAX_THREAD_SIZE+"]，请稍后重试", ErrorCode.API_ERROR, null));
                    TRestUtil.write(response, resStr);
                    logger.info("["+requestPath+"]当前运行线程数超过限制"+MAX_THREAD_SIZE+"个");
                    return false;
                }
            }
            uriMap.get(key).add(Thread.currentThread().getName());
        }
        return true;
    }
 
    // after handler invoked
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler,
            ModelAndView modelAndView) throws Exception {
        String requestPath = request.getRequestURI();
        EntrustCompany loginUser = getLoginUserInfo(request);
        if (loginUser == null) {
            logger.info("并发拦截结束，当前无登录用户信息，返回");
        }
        String orgCode = getLoginUserInfo(request).getOrgCode();
        logger.info("企业["+orgCode+"]请求["+requestPath+"]并发拦截结束");
        String key = orgCode + DIV_STR + requestPath;
        synchronized (LOCKER) {
            if (uriMap.containsKey(key)) {
                String threadName = Thread.currentThread().getName();
                logger.info("并发拦截结束，删除当前线程标记["+threadName+"]");
                if (!CollectionUtils.isEmpty(uriMap.get(key))) {
                    uriMap.get(key).remove(threadName);
                }
            }
        }
    }
 
    // after view rendered
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex)
            throws Exception {
    }
 
}
```

dispacher-servlet.xml中的配置
```xml
<mvc:interceptors>
        <mvc:interceptor>
            <mvc:mapping path="/chb-api/**"/>
            <bean id="concurrentInterceptor" class="com.hongkun.greenpass.exchange.basic.web.ConcurrentNumberInterceptor"></bean>
        </mvc:interceptor>
</mvc:interceptors>
```