---
layout: post
title: No tests found matching
tags: Exception
source: virgin
---

    JUnit用的4.10，然后把项目中SpringFramework升级到4.3.3，所有的JUnit单元测试运行都报这个错，最后单独将spring-test包回退到3.2.4之后便又好了。

## 异常堆栈
```java
Test.test
initializationError(org.junit.runner.manipulation.Filter)
java.lang.Exception: No tests found matching [{ExactMatcher:fDisplayName=test], {ExactMatcher:fDisplayName=test(com.hongkun.product.test.Test)], {LeadingIdentifierMatcher:fClassName=com.hongkun.product.test.Test,fLeadingIdentifier=test]] from org.junit.internal.requests.ClassRequest@4722292a
    at org.junit.internal.requests.FilterRequest.getRunner(FilterRequest.java:37)
    at org.eclipse.jdt.internal.junit4.runner.JUnit4TestLoader.createFilteredTest(JUnit4TestLoader.java:77)
    at org.eclipse.jdt.internal.junit4.runner.JUnit4TestLoader.createTest(JUnit4TestLoader.java:68)
    at org.eclipse.jdt.internal.junit4.runner.JUnit4TestLoader.loadTests(JUnit4TestLoader.java:43)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:444)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.runTests(RemoteTestRunner.java:678)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.run(RemoteTestRunner.java:382)
    at org.eclipse.jdt.internal.junit.runner.RemoteTestRunner.main(RemoteTestRunner.java:192)
 ```

## 异常原因
异常原因有一大堆，可以看 [StackOverFlow](http://stackoverflow.com/questions/2332832/no-tests-found-with-test-runner-junit-4)中的帖子。我碰到的原因当然是因为升级Spring所导致的，故此将spring-test回退到3.2.4就好了