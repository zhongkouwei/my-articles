# @Transactional 事务分析



在Spring开发中，经常会用到这个注解，但它是怎么起到开启事务的作用的，还一知半解，只知道应该是基于AOP的，今天就来分析一波它的用法和具体实现。

## 编程式事务和声明式事务

​	Spring 支持两种事务管理方式：编程式和声明式，编程式是在程序中显式地使用TransactionTemplate来控制事务的开启和提交。

​	声明式事务是使用@Transactional注解，在类或方法上使用这个注解，就可以起到开启事务的作用，声明式事务是基于AOP的方式，在方法前开启一个事务，在方法执行后进行commit，中间进行一些异常的判断和处理。

​	相比来说，声明式事务使用起来更加优雅，AOP的方式对代码没有侵入性，比较推荐在日常开发中使用。

## 声明式事务的用法

 	@Transactional注解可以加在类或方法上，加在类上时是对该类的所有public方法开启事务。加在方法上时也是只对public方法起作用。另外@Transactional注解也可以加在接口上，但只有在设置了基于接口的代理时才会生效，因为注解不能继承。所以该注解最好是加在类的实现上。

​	下面看一下@Transactional注解的各项参数。

```java
public @interface Transactional {
    @AliasFor("transactionManager")
    String value() default "";

    @AliasFor("value")
    String transactionManager() default "";

    Propagation propagation() default Propagation.REQUIRED;

    Isolation isolation() default Isolation.DEFAULT;

    int timeout() default -1;

    boolean readOnly() default false;

    Class<? extends Throwable>[] rollbackFor() default {};

    String[] rollbackForClassName() default {};

    Class<? extends Throwable>[] noRollbackFor() default {};

    String[] noRollbackForClassName() default {};
}
```

### timtout

timtout是用来设置事务的超时时间，可以看到默认为-1，不会超时。

### isolation

isolation属性是用来设置事务的隔离级别，数据库有四种隔离级别：读未提交、读已提交、可重复读、可串行化。MySQL的默认隔离级别是可重复读。

```java
public enum Isolation {
    DEFAULT(-1),
    READ_UNCOMMITTED(1),  // 读未提交
    READ_COMMITTED(2), 		// 读已提交
    REPEATABLE_READ(4), 	// 可重复读
    SERIALIZABLE(8);			// 可串行化
}
```

### readOnly

​	readOnly属性用来设置该属性是否是只读事务，只读事务要从两方面来理解：它的功能是设置了只读事务后在整个事务的过程中，其他事务提交的内容对当前事务是不可见的。

​	那为什么要设置只读事务呢？它的好处是什么？可以看出它的作用是保持整个事务的数据一致性，如果事务中有多次查询，不会出现数据不一致的情况。所以在一个事务中如果有多次查询，可以启用只读事务，如果只有一次查询就无需只读事务了。

​	另外，使用了只读事务，数据库会提供一些优化。

​	但要注意的是，只读事务中只能有读操作，不能含有写操作，否则会报错。

### propagation

propagation属性用来设置事务的传播行为，对传播行为的理解，可以参考如下场景，一个开启了事务的方法A，调用了另一个开启了事务的方法B，此时会出现什么情况？这就要看传播行为的设置了。

```java
public enum Propagation {
    REQUIRED(0), 				// 如果有事务则加入，没有则新建
    SUPPORTS(1),				// 如果已有事务就用，如果没有就不开启(继承关系)
    MANDATORY(2),				// 必须在已有事务中
    REQUIRES_NEW(3),		// 不管是否已有事务，都要开启新事务，老事务挂起
    NOT_SUPPORTED(4),   // 不开启事务
    NEVER(5),						// 必须在没有事务的方法中调用，否则抛出异常
    NESTED(6);					// 如果已有事务，则嵌套执行，如果没有，就新建(和REQUIRED类似，和REQUIRES_NEW容易混淆)
}
```

REQUIRES_NEW 和 NESTED非常容易混淆，因为它们都是开启了一个新的事务。我去查询了一下它们之间的区别，大概是这样：

REQUIRES_NEW是开启一个完全的全新事务，和当前事务没有任何关系，可以单独地失败、回滚、提交。并不依赖外部事务。在新事务执行过程中，老事务是挂起的。

NESTED也是开启新事务，但它开启的是基于当前事务的**子事务**，如果失败的话单独回滚，但如果成功的话，并不会立即commit，而是等待外部事务的执行结果，外部事务commit时，子事务才会commit。

### rollbackFor

   在@Transactional注解中，有一个重要属性是roolbackFor，这是用来判断在什么异常下会进行回滚的，当方法内抛出指定的异常时，进行事务回滚。**rollbackForClassName**也是类似的。

​	rollbackFor有个问题是默认情况会做什么，以前认为默认会对所有异常进行回滚，但其实默认情况下只对RuntimeException回滚。	

### noRollbackFor

这个和上面正好相反，用来设置出现指定的异常时，不进行回滚。

## 声明式事务的实现机制



Spring在调用事务增强器的代理类时会首先执行TransactionInterceptor进行增强，在TransactionInterceptor的invoke方法中完成事务逻辑。首先看下TransactionInterceptor的类图结构。

![](https://s2.ax1x.com/2019/06/13/VfLIiD.png)

### TransactionInterceptor

```java
	public Object invoke(MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
    // 获取目标类
		Class<?> targetClass = (invocation.getThis() != null ?AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass,invocation::proceed);
	}
```

在invoke方法里，调用了父类的模板方法invokeWithinTransaction，下面我们看下TransactionAspectSupport类。

```java
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
			final InvocationCallback invocation) throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
  	// 获取事务属性
		TransactionAttributeSource tas = getTransactionAttributeSource();
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);
  	// 获取PlatformTransactionManager的实现类，底层事务的处理实现都是由PlatformTransactionManager的实现类实现的，比如 DataSourceTransactionManager 管理 JDBC 的 Connection。
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
  	// 切点标识
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
      // 这里是根据事务的传播行为属性去判断是否创建一个事务
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
        // around增强，这里是执行的回调的目标方法
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
        // 异常回滚
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
        // 清理信息
				cleanupTransactionInfo(txInfo);
			}
      // 提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
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
							throwableHolder.throwable = ex;
							return null;
						}
					}
					finally {
						cleanupTransactionInfo(txInfo);
					}
				});

				// Check result state: It might indicate a Throwable to rethrow.
				if (throwableHolder.throwable != null) {
					throw throwableHolder.throwable;
				}
				return result;
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
			catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}
		}
	}
```

### createTransactionIfNecessary

可以看到，这个invokeWithinTransaction方法已经包含了事务执行的整个流程，这里是使用了模板模式，具体的实现交给子类去实现。下面我们分析一下其中的重要方法。

```java
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
        // 获取事务状态，判断各种属性
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
  	// 返回一个TransactionInfo
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```

### getTransaction

下面们看下用来用来获取TransactionStatus的getTransaction方法，这个方法是在AbstractPlatformTransactionManager抽象类中，

```java
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
  	// 这里是开启一个事务，不同的事务类型(JPA  kafka jta等有不同的实现)
		Object transaction = doGetTransaction();

		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			// Use defaults if no transaction definition given.
			definition = new DefaultTransactionDefinition();
		}
		// 判断当前是否存在事务，如果存在则进行一些判断和操作
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}
		// 设置超时
		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

  	// 不存在事务，但上面讲了PROPAGATION_MANDATORY要求必须已有事务，则抛出异常
		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
  	//后续其他属性都需要去新建一个事务
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error ex) {
				resume(null, suspendedResources);
				throw ex;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
```

可以看到getTransaction是用来做了一些事务的初始化工作，包括一些判断，新建事务等等。

其中一些对已有事务的处理、嵌入式事务的处理的细节，暂时就略过了~

### completeTransactionAfterThrowing

下面我们回到TransactionInterceptor，看一下下一个流程：回滚

```
/**
	 * Handle a throwable, completing the transaction.
	 * We may commit or roll back, depending on the configuration.
	 * @param txInfo information about the current transaction
	 * @param ex throwable encountered
	 */
	protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			if (logger.isTraceEnabled()) {
				logger.trace("Completing transaction for [" + txInfo.getJoinpointIdentification() +
						"] after exception: " + ex);
			}
			// 这里是判断异常类型：默认会判断RuntimeException和Error，后面会分析
			if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					// 回滚处理，这里是由不同的子类实现回滚的
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by rollback exception", ex);
					throw ex2;
				}
			}
			// 下面是不满足回滚条件的，会照样提交
			else {
				// We don't roll back on this exception.
				// Will still roll back if TransactionStatus.isRollbackOnly() is true.
				try {
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				catch (TransactionSystemException ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					ex2.initApplicationException(ex);
					throw ex2;
				}
				catch (RuntimeException | Error ex2) {
					logger.error("Application exception overridden by commit exception", ex);
					throw ex2;
				}
			}
		}
	}
```



上面有个rollbackOn(ex)方法，是用来判断回滚类型的，我们看下它的实现，它有不同的实现类，看下默认的

![](https://s2.ax1x.com/2019/06/13/Vh98it.png)

```java
	public boolean rollbackOn(Throwable ex) {
		return (ex instanceof RuntimeException || ex instanceof Error);
	}

```

确实是RuntimeException和Error。

我们接着看一下回滚的实现。

```java
public final void rollback(TransactionStatus status) throws TransactionException {
  // 事务已经完成时，回滚会出现异常
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		processRollback(defStatus, false);
	}
```



### processRollback

```
 * Process an actual rollback.
	 * The completed flag has already been checked.
	 * @param status object representing the transaction
	 * @throws TransactionException in case of rollback failure
	 */
	private void processRollback(DefaultTransactionStatus status, boolean unexpected) {
		try {
			boolean unexpectedRollback = unexpected;

			try {
				triggerBeforeCompletion(status);
				// 有保存点时候回退到保存点(这里是指子事务？应该是指嵌套事务吧)
				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Rolling back transaction to savepoint");
					}
					status.rollbackToHeldSavepoint();
				}
				// 如果是新事务，则直接回退
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction rollback");
					}
					doRollback(status);
				}
				else {
					// Participating in larger transaction
					if (status.hasTransaction()) {
						if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
							if (status.isDebug()) {
								logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
							}
							// 如果不是独立的事务，就只记录下来，等外部事务执行完毕一起执行
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
					// Unexpected rollback only matters here if we're asked to fail early
					if (!isFailEarlyOnGlobalRollbackOnly()) {
						unexpectedRollback = false;
					}
				}
			}
			catch (RuntimeException | Error ex) {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				throw ex;
			}

			triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);

			// Raise UnexpectedRollbackException if we had a global rollback-only marker
			if (unexpectedRollback) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
		}
		finally {
			cleanupAfterCompletion(status);
		}
	}
```

上面的代码中，有保存点的是指的嵌套事务，因为嵌套事务并不是真正的两个事务，所以会有保存点的信息进行回滚。

事务的提交和回滚的流程类似，同样进行了不同事务类型的判断，在此不进行额外的分析。

## 结语

​	通过idea的反编译工具，分析了一波代码，同时参考了《Spring源码解析》，练习了一下分析源码的能力，能大体地看出事务的实现过程，大概看过之后感觉Spring这样写是非常合理的，代码非常清晰，每个功能点都拆分地特别细。博客没有分析事务的加载，因为那块和事务本身的处理并没有太大的关系，而是Spring对注解等属性的加载并生成TransactionInterceptor代理对象。

​	而且Spring里面使用了非常多的模板方法，使用模板方法对我们的好处是，能从大到小、从整体到局部地了解到整个过程，而且大大降低代码的耦合性。以后在编码时也会注意多使用模板方法。