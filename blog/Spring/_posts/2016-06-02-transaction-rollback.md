---
layout: post
title: Spring 事务 —— rollback for
tags: Spring Transaction
source: virgin
---

**引语**：事务回滚，可以配置哪些异常回滚，哪些异常不回滚，如果不细究其内部实现，则会一直担心不知道该怎么配，比如以为配了`rollbackFor = Exception.class`则误以为，只有抛出Exception异常时才会回滚，实则不然！

**规则**

名词定义：异常深度，即如果当前捕的获异常(A)是rollbackFor配置的异常(B)的父类，那B是A的几层继承就是其异常深度。比如 `A <-- C <-- B`，那么A对于B的异常深度就是2，C对于B的异常深度就是1；

1. `rollback for`和`no rollback for`都没有配置，则遵循如下规则，`(ex instanceof RuntimeException || ex instanceof Error)`，也就是说异常是`RuntimeException`或者`Error`的实例，就会回滚；

2. 只配了`rollback for`
    如果异常是当前配置的异常或其父类，则直接回滚

    如果异常不是当前配置的异常或其父类，则判断`(ex instanceof RuntimeException || ex instanceof Error)`，决定是否回滚

3. 只配了`no rollback for`

    如果异常是当前配置的异常或其父类，则不回滚

    如果异常不是当前配置的异常或其父类，则判断`(ex instanceof RuntimeException || ex instanceof Error)`，决定是否回滚

4. `rollback for`和`no rollback for`都有配置

    如果异常不是rollbackFor和noRollbackFor或他们的父类，则判断`(ex instanceof RuntimeException || ex instanceof Error)`，决定是否回滚

    如果异常是当前配置的rollbackFor或其父类，且不是noRollbackFor或其父类，则直接回滚

    如果异常是当前配置的noRollbackFor或其父类，且不是rollbackFor或其父类，则不回滚

    如果异常同时是rollbackFor和noRollbackFor或他们的父类，那么判断各自的异常深度，哪个深度小则为胜出者，胜出者再继续2和3的流程；如果异常深度相同，则优先使用rollbackFor的规则，也即2；

**代码解析**

```java
@SuppressWarnings("serial")
public class RuleBasedTransactionAttribute extends DefaultTransactionAttribute implements Serializable {
	
	/**
	 * Winning rule is the shallowest rule (that is, the closest in the
	 * inheritance hierarchy to the exception). If no rule applies (-1),
	 * return false.
	 * @see TransactionAttribute#rollbackOn(java.lang.Throwable)
	 */
	@Override
	public boolean rollbackOn(Throwable ex) {
		if (logger.isTraceEnabled()) {
			logger.trace("Applying rules to determine whether transaction should rollback on " + ex);
		}

		RollbackRuleAttribute winner = null;
		int deepest = Integer.MAX_VALUE;

		// rollbackRules链表中放了rollbackFor异常和noRollbackFor异常，其中rollbackFor在前边，noRollbackFor排在后边
		if (this.rollbackRules != null) {
			for (RollbackRuleAttribute rule : this.rollbackRules) {
				// 获取捕获的异常对于配置的异常的异常深度
				int depth = rule.getDepth(ex);
				// 如果哪个异常深度小，则肯定会是胜出者
				if (depth >= 0 && depth < deepest) {
					deepest = depth;
					winner = rule;
				}
			}
		}

		if (logger.isTraceEnabled()) {
			logger.trace("Winning rollback rule is: " + winner);
		}

		// User superclass behavior (rollback on unchecked) if no rule matches.
		// 如果配置的异常都没有找到，则调父类默认规则：return (ex instanceof RuntimeException || ex instanceof Error);
		if (winner == null) {
			logger.trace("No relevant rollback rule found: applying default rules");
			return super.rollbackOn(ex);
		}

		// 如果胜出者是不回滚的异常，则不回滚，否则反之
		return !(winner instanceof NoRollbackRuleAttribute);
	}
	
}

/**
 * Rule determining whether or not a given exception (and any subclasses)
 * should cause a rollback.
 *
 * <p>Multiple such rules can be applied to determine whether a transaction
 * should commit or rollback after an exception has been thrown.
 *
 * @author Rod Johnson
 * @since 09.04.2003
 * @see NoRollbackRuleAttribute
 */
@SuppressWarnings("serial")
public class RollbackRuleAttribute implements Serializable{
	
	/**
	 * Return the depth of the superclass matching.
	 * <p>{@code 0} means {@code ex} matches exactly. Returns
	 * {@code -1} if there is no match. Otherwise, returns depth with the
	 * lowest depth winning.
	 */
	public int getDepth(Throwable ex) {
		return getDepth(ex.getClass(), 0);
	}


	private int getDepth(Class exceptionClass, int depth) {
		if (exceptionClass.getName().indexOf(this.exceptionName) != -1) {
			// Found it!
			return depth;
		}
		// If we've gone as far as we can go and haven't found it...
		if (exceptionClass.equals(Throwable.class)) {
			return -1;
		}
		// 递归，找到父类中有没有该异常
		return getDepth(exceptionClass.getSuperclass(), depth + 1);
	}
	
}
```
