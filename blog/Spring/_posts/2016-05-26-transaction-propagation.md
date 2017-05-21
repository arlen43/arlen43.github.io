---
layout: post
title: Spring 事务 —— 传播机制
tags: Spring Transaction
source: virgin
---

**引语**：Spring对事务的嵌套调用有专门的传播机制，有时只从字面上理解往往会误入歧途，下面对传播机制做了详细讲解。
* REQUIRED
  如果当前没事务，就新建一个事务，如果已经存在一个事务，则加入到这个事务中，这个是Spring默认的传播机制；
* SUPPORTS
  支持当前事务，如果没有事务就以非事务方式执行；
* MANADATORY
  使用当前事务，如果当前没有事务，就抛出异常；
* REQUIRES_NEW
  新建事务，如果当前存在事务，则**将当前事务挂起**；
* NOT_SUPPORTED
  以非事务方式执行，如果当前存在事务，则**将当前事务挂起**；
* NEVER
  以非事务方式执行，如果当前存在事务，则抛出异常；
* NESTED
  如果当前存在事务，则在嵌套事务中执行，如果当前没有事务，则执行跟REQUIRED相似的操作。

**坑1** —— 新建子事务，导致当前事务挂起
1. 如果当前事务读取了A表的数据，然后子事务更改了A表数据，如果当前事务还去使用之前读的数据，那就属于脏读！！（因为你假如不知道这个注解会让当前事务挂起，你以为）
2. 如果当前事务修改了A表数据，然后子事务再去读取A表数据，因为当前事务被挂起，那么子事务读到的还是老的数据！！比如实际开发中遇到的一个问题，主事务中改了订单状态，子事务去查询该订单状态是否已改，并且做其他操作，但是查到的还是没有改的结果，导致出错！
  当然，当前事务修改A表数据，是用了非新建子事务A，然后再进入当前事务新建子事务B，子事务B读取数据还会是老数据！
  如果，当前事务修改A表数据，是用了新建子事务A，并且A成功提交，那么再进入当前事务新建子事务B，子事务B读到的是已经提交的新数据！

**坑2** —— 事务嵌套，新建子事务引起死锁
1. 如果当前事务修改了A表数据（加行写锁），然后子事务读取A表数据（加行读锁）没有问题，但是当子事务要去更新A表数据时，因为当前事务已经锁定了该行，那么子事务就会一直等下去，造成死锁！
    等待超时报错：`Lock wait timeout exceeded; try restarting transaction`

**坑3** —— 事务嵌套，想回滚事务至某个savepoint报错

`Transaction rolled back because it has been marked as rollback-only`
有时想对一段业务逻辑分段加事务，比如修改订单信息`orderService.updateOrder()`（A）之后想记录日志`logService.log()`（B），但是不想保存日志异常影响到正常的逻辑回滚，往往我们的第一想法就是将记录日志的代码`try catch`，想法是对的，但往往做法是错的，往往我们会在`orderService.updateOrder()`里调用`logService.log()`时加`try catch`，那么问题就来了。

**坑4** —— 配置文件跟注解同时使用

`Transaction rolled back because it has been marked as rollback-only`
同样的错误，还以为是Spring传播机制不起作用的问题，故此深挖*坑3*，将事务仔细研究了一下，最后发现Spring还是很靠谱的，不会坑蒙拐骗，那么就是项目中的问题了。项目中spring-tx如下配置：

```xml
<tx:advice id="ossTxAdvice" transaction-manager="transactionManager">
	<tx:attributes>
		<tx:method name="get*"  read-only="true" />
		<!-- ... -->
		<tx:method name="save*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.Exception" />
		<tx:method name="add*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.Exception" />
		<tx:method name="update*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.Exception" />
		<tx:method name="delete*" propagation="REQUIRED" read-only="false" rollback-for="java.lang.Exception" />
	</tx:attributes>
</tx:advice>
```
a. 以为是rollback-for配置的问题，看了代码，代码中是先判断有无rollbackFor，如果有，那么只判断rollbackFor是否是`NoRollbackRuleAttribute`的实例，如果不是，则会去回滚，也就是问题不是出在这。

b. 详细看日志，发现加了注解的方法，其实事务传播机制是`REQUIRED`，并不是注解中的`REQUIRES_NEW`，那么问题就出在这，xml文件中的将注解的配置覆盖了！

**Transaction rolled back because it has been marked as rollback-only** —— 解决方案

1. 配置嵌套子事务的传播机制为`REQUIRES_NEW`

    优：解决了抛异常的情况

    缺：会造成当前事务挂起，会有脏读情况，无法避免

2. 配置嵌套子事务的传播机制为`NESTED`

    优：完美解决抛异常情况，并且不会挂起当前事务，异常抛出时仅仅是将事务回滚到子事务入口处的savepoint；**经过测试真的就完美解决了，好鸡冻**

    缺：数据库驱动得支持JDBC3.0，不过目前的驱动只要不是太旧，都是支持的！

# 坑3解析

## 逻辑分析

Spring会在进入`orderService.updateOrder()`方法时，为其加AOP代理`CglibAopProxy`，并设置拦截器`TransactionAspectSupport->TransactionInterceptor`，执行方法A前获取连接，建立新事务，做如下操作
1. 新建主事务，初始化一堆东西，最主要的TransactionInfo，见下面主、子事务TransactionInfo信息比较
2. 从事务中注册资源
3. 操作数据库
4. 释放资源
5. 将子事务加入到当前主事务中
6. 从事务中注册资源
7. 操作数据库
8. 释放资源
9. 子事务抛出异常
10. 设置子事务的jdbc连接持有者（ResourceHolderSupport）属性rollbackOnly = true，不做实际的回滚
11. 主事务继续执行，然后事务提交
12. 判断globalRollbackOnParticipationFailure和rollbackOnly，如果为true，回滚主事务，抛出异常Participating transaction failed - marking existing transaction as rollback-only

## TransactionInfo运行值分析
主事务信息TransactionInfo txInfo

代理：TransactionInterceptor@b890db3

joinpointIdentification：com.hongkun.oss.test.trans.impl.TransParentImpl.updateOrder

oldTransaction：null

transactionStatus: DefaultTransactionStatus@5457558

\|-transaction: DataSourceTransactionObject@11739298

\|----connectionHold: **ConnectionHolder@19c53683**

transactionManager: **DataSourceTransactionManager@51dbc859**，事务管理器，包含属性globalRollbackOnParticipationFailure，表示子事务失败，主事务如何处理，很早之前的做法，目前改为PROPAGATION_NESTED来处理

子事务信息TransactionInfo txInfo

代理：TransactionInterceptor@b890db3

joinpointIdentification：com.hongkun.oss.test.trans.impl.TransChildImpl.updateOrderStatus

oldTransaction：父事务TransactionInfo txInfo

transactionStatus: DefaultTransactionStatus@144444

\|-transaction: DataSourceTransactionObject@111c19a3

\|----connectionHold: **ConnectionHolder@19c53683**

transactionManager: **DataSourceTransactionManager@51dbc859**

## 关键类图
![Transaction关键类图]({{site.url}}/assets/img-blog/Spring/transaction-class.png)

## 关键java代码分析
```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
	// ...
	// 主方法进入，调用invokeWithinTransaction方法
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
		});
	}
	/**
	 * General delegate for around-advice-based subclasses, delegating to several other template
	 * methods on this class. Able to handle {@link CallbackPreferringPlatformTransactionManager}
	 * as well as regular {@link PlatformTransactionManager} implementations.
	 * @param method the Method being invoked
	 * @param targetClass the target class that we're invoking the method on
	 * @param invocation the callback to use for proceeding with the target invocation
	 * @return the return value of the method, if any
	 * @throws Throwable propagated from the target invocation
	 */
	// 总共步 1. createTransactionIfNecessary；2. completeTransactionAfterThrowing；3. cleanupTransactionInfo；4. commitTransactionAfterReturning
	// 代码中的日志记录了抛出异常Participating transaction failed - marking existing transaction as rollback-only的过程
	protected Object invokeWithinTransaction(Method method, Class targetClass, final InvocationCallback invocation)
			throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
		// 全局的事务处理类，子事务、主事务共用一个，里边有globalRollbackOnParticipationFailure标识
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			// 如果需要的话，就创建事务，事务放入TransactionStatus
			// 如果有事务，从当前事务中初始化子事务TransactionStatus，跟当前事务用一个ConnectionHolder，里边有rollbackOnly标识
			// ------------- create new transaction start ----------------
			// Creating new transaction with name[*impl#**]: PROPAGATION_REQUIRED,ISOLATION_DEFAULT; ''
			// Acquired Connection [A] for JDBC transaction
			// Switching JDBC Connection [A] to manual commit
			// --- 实际业务方法中用到数据库连接，并操作数据库
			// --- Creating a new SqlSession
			// --- Registering transaction synchronization for SqlSession [27375710]
			// --- JDBC Connection [A] will be managed by Spring
			// --- Using Connection [A]
			// --- Preparing: some sql
			// --- Releasing transactional SqlSession [27375710]
			// ------------- create child transaction start （主事务嵌套调用其他服务的子事务，子事务非新建）---------------- 
			// Participating in existing transaction
			// --- 实际业务方法中用到数据库连接，并操作数据库
			// --- Fetched SqlSession [ 27375710] from current transaction
			// --- Using Connection [A]
			// --- Preparing: some sql
			// --- Releasing transactional SqlSession [27375710]
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				// 判断是否设置了rollbackExcepiton，如果没有设置，则回滚 ‘ex instanceof RuntimeException || ex instanceof Error’ 异常
				// 如果回滚异常，调用AbstractPlatformTransactionManager.processRollback
				// 有异常，根据子
				// Participating transaction failed - marking existing transaction as rollback-only
				// Setting JDBC transaction [A] rollback-only
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				// 如果子事务嵌入到主事务，还原事务上下文到主事务
				cleanupTransactionInfo(txInfo);
			}
			
			// Global transaction is marked as rollback-only but transactional code requested commit
			// Initiating transaction rollback
			// Rolling back JDBC transaction on Connection [A]
			// Transaction synchronization rolling back SqlSession [27375710]
			// Transaction synchronization closing SqlSession [27375710]
			// Releasing JDBC Connection [A] after transaction
			// Returning JDBC Connection to DataSource
			// ** throw UnexpectedRollbackException("Transaction rolled back because it has been marked as rollback-only")
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
		else {
			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
						new TransactionCallback<Object>() {
							public Object doInTransaction(TransactionStatus status) {
								TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
								try {
									return invocation.proceedWithInvocation();
								}
								catch (Throwable ex) {
									if (txAttr.rollbackOn(ex)) {
										// A RuntimeException: will lead to a rollback.
										if (ex instanceof RuntimeException) {
											throw (RuntimeException) ex;
										}
										else {
											throw new ThrowableHolderException(ex);
										}
									}
									else {
										// A normal return value: will lead to a commit.
										return new ThrowableHolder(ex);
									}
								}
								finally {
									cleanupTransactionInfo(txInfo);
								}
							}
						});

				// Check result: It might indicate a Throwable to rethrow.
				if (result instanceof ThrowableHolder) {
					throw ((ThrowableHolder) result).getThrowable();
				}
				else {
					return result;
				}
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
		}
	}
	// ...
}

public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {  
	 // ...
	 // 事务拦截器主入口代码，调用父类的invokeWithinTransaction，
    public Object invoke(final MethodInvocation invocation) throws Throwable {
        // Work out the target class: may be {@code null}.
        // The TransactionAttributeSource should be passed the target class
        // as well as the method, which may be from an interface.
        Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
        // Adapt to TransactionAspectSupport's invokeWithinTransaction...
        return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
            public Object proceedWithInvocation() throws Throwable {
                return invocation.proceed();
            }
        });
    } 
	// ...
} 

@SuppressWarnings("serial")
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {  

	/**
	 * Set whether to globally mark an existing transaction as rollback-only
	 * after a participating transaction failed.
	 * <p>Default is "true": If a participating transaction (e.g. with
	 * PROPAGATION_REQUIRES or PROPAGATION_SUPPORTS encountering an existing
	 * transaction) fails, the transaction will be globally marked as rollback-only.
	 * The only possible outcome of such a transaction is a rollback: The
	 * transaction originator <i>cannot</i> make the transaction commit anymore.
	 * <p>Switch this to "false" to let the transaction originator make the rollback
	 * decision. If a participating transaction fails with an exception, the caller
	 * can still decide to continue with a different path within the transaction.
	 * However, note that this will only work as long as all participating resources
	 * are capable of continuing towards a transaction commit even after a data access
	 * failure: This is generally not the case for a Hibernate Session, for example;
	 * neither is it for a sequence of JDBC insert/update/delete operations.
	 * <p><b>Note:</b>This flag only applies to an explicit rollback attempt for a
	 * subtransaction, typically caused by an exception thrown by a data access operation
	 * (where TransactionInterceptor will trigger a {@code PlatformTransactionManager.rollback()}
	 * call according to a rollback rule). If the flag is off, the caller can handle the exception
	 * and decide on a rollback, independent of the rollback rules of the subtransaction.
	 * This flag does, however, <i>not</i> apply to explicit {@code setRollbackOnly}
	 * calls on a {@code TransactionStatus}, which will always cause an eventual
	 * global rollback (as it might not throw an exception after the rollback-only call).
	 * <p>The recommended solution for handling failure of a subtransaction
	 * is a "nested transaction", where the global transaction can be rolled
	 * back to a savepoint taken at the beginning of the subtransaction.
	 * PROPAGATION_NESTED provides exactly those semantics; however, it will
	 * only work when nested transaction support is available. This is the case
	 * with DataSourceTransactionManager, but not with JtaTransactionManager.
	 * @see #setNestedTransactionAllowed
	 * @see org.springframework.transaction.jta.JtaTransactionManager
	 */
	// 注释已经说的很明确了，要改变该值，请用PROPAGATION_NESTED，不过必须得支持JDBC3.0
	public final void setGlobalRollbackOnParticipationFailure(boolean globalRollbackOnParticipationFailure) {
		this.globalRollbackOnParticipationFailure = globalRollbackOnParticipationFailure;
	}

	/**
	 * This implementation of commit handles participating in existing
	 * transactions and programmatic rollback requests.
	 * Delegates to {@code isRollbackOnly}, {@code doCommit}
	 * and {@code rollback}.
	 * @see org.springframework.transaction.TransactionStatus#isRollbackOnly()
	 * @see #doCommit
	 * @see #rollback
	 */
	public final void commit(TransactionStatus status) throws TransactionException {
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		// 如果子事务异常，全局回滚标识会被设置为rollbackOnly，回滚主事务
		if (defStatus.isLocalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Transactional code has requested rollback");
			}
			processRollback(defStatus);
			return;
		}
		// 如果子事务异常，TransactionManager的globalRollbackOnParticipationFailure为true，且事务jdbc连接的connectionHold为rollbackOnly，等主事务回滚成功后再抛出异常！
		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
			}
			processRollback(defStatus);
			// Throw UnexpectedRollbackException only at outermost transaction boundary
			// or if explicitly asked to.
			if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
			return;
		}

		processCommit(defStatus);
	}
	
    /**
     * Process an actual rollback.
     * The completed flag has already been checked.
     * @param status object representing the transaction
     * @throws TransactionException in case of rollback failure
     */
    private void processRollback(DefaultTransactionStatus status) {
        try {
            try {
                triggerBeforeCompletion(status);
                if (status.hasSavepoint()) {
                    if (status.isDebug()) {
                        logger.debug("Rolling back transaction to savepoint");
                    }
                    status.rollbackToHeldSavepoint();
                }
                else if (status.isNewTransaction()) {
                    if (status.isDebug()) {
                        logger.debug("Initiating transaction rollback");
                    }
                    doRollback(status);
                }
                else if (status.hasTransaction()) {
                    if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
                        if (status.isDebug()) {
                            logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
                        }
                        // Setting JDBC transaction [connetion name] rollback-only，标记连接拥有者rollbackOnly=true(ResourceHolderSupport.rollbackOnly)
                        doSetRollbackOnly(status);
                    }
                    else {
                        if (status.isDebug()) {
                            logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
                        }
                    }
                }
                else {
                    logger.debug("Should roll back transaction but cannot - no transaction available");
                }
            }
            catch (RuntimeException ex) {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
                throw ex;
            }
            catch (Error err) {
                triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
                throw err;
            }
            triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
        }
        finally {
            cleanupAfterCompletion(status);
        }
    }
} 
```
以上解释在网上和书本中一大堆，但是这么简单的解释，得知道里边的内涵，不然就会有无限的大坑等着你！

其中 REQUIRES_NEW 和 NESTED 的使用，REQUIRED 和 NESTED 的使用，最易混淆，以后会就这两种气情况做测试，并撰文说明。1. 主事务调用子事务，子事务抛异常？ 2. 主事务调用子事务，子事务结束后主事务回滚？
