# spring 事务执行源代码

spring 事务的拦截器类为 `TransactionInterceptor`，所有事务拦截的代码都会到这个类的 `#invoke` 方法，这个 invoke 方法主要是调用 `org.springframework.transaction.interceptor.TransactionAspectSupport#invokeWithinTransaction`

该方法源代码如下：
```java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
		throws Throwable {

	final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
	final PlatformTransactionManager tm = determineTransactionManager(txAttr);
	final String joinpointIdentification = methodIdentification(method, targetClass);

	if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
		TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
		Object retVal = null;
		try {
			retVal = invocation.proceedWithInvocation();
		}
		catch (Throwable ex) {
			completeTransactionAfterThrowing(txInfo, ex);
			throw ex;
		}
		finally {
			cleanupTransactionInfo(txInfo);
		}
		commitTransactionAfterReturning(txInfo);
		return retVal;
	}

	else {
        // 省略，大部分情况下走上面的逻辑
	}
}
```
方法说明
- getTransactionAttributeSource() 通常情况下返回的是 `AnnotationTransactionAttributeSource`，所以第一句代码 TransactionAttribute txAttr = AnnotationTransactionAttributeSource.getTransactionAttribute(method, targetClass); 该方法会在对应方法，类上寻找 `@Transactional` 注解，如果找到了注解，就会返回 TransactionAttribute 对象
- 找到对应的事务管理器 `PlatformTransactionManager`，这个跟具体配置的事务管理器有关，通常返回的是 `DataSourceTransactionManager`
```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource"/>
</bean>
```
- 开启事务的主要代码是 `#createTransactionIfNecessary` 这个方法的调用
- catch 中进行事务的回滚；finally 中清除相关事务的绑定，将 TransactionInfo 和当前线程解除绑定；finally 之后进行事务的提交 `commitTransactionAfterReturning`


我们看下 `#createTransactionIfNecessary` 这个方法，这个方法里面重要的调用是 `DataSourceTransactionManager#getTransaction(txAttr);` 方法，并且在最后调用 `prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);` 将此次创建的事务和当前线程绑定。


```java
public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
    // DataSourceTransactionManager#doGetTransaction
    Object transaction = doGetTransaction();

    // Cache debug flag to avoid repeated checks.
    boolean debugEnabled = logger.isDebugEnabled();

    if (definition == null) {
        // Use defaults if no transaction definition given.
        definition = new DefaultTransactionDefinition();
    }

    if (isExistingTransaction(transaction)) {
        // Existing transaction found -> check propagation behavior to find out how to behave.
        return handleExistingTransaction(definition, transaction, debugEnabled);
    }

    // Check definition settings for new transaction.
    // timeout < -1 抛出异常
    if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
    }

    // @Transactional(propagation = Propagation.MANDATORY)  MANDATORY 表示支持当前事务，如果当前不存在现存事务，直接抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(
                "No existing transaction found for transaction marked with propagation 'mandatory'");
    }
    else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        // 挂起当前线程事务，如果不存在返回 null
        SuspendedResourcesHolder suspendedResources = suspend(null);
        if (debugEnabled) {
            logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
        }
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // 子类 DataSourceTransactionManager#doBegin，根据配置的 dataSource，获取 connection，并且设置 con.setAutoCommit(false);
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException ex) {
            // 出现异常的话，将原挂起的事务还原
            resume(null, suspendedResources);
            throw ex;
        }
        catch (Error err) {
            resume(null, suspendedResources);
            throw err;
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

通过上面这个方法，开启事务的代码就已经很清楚了，我们再回到 `TransactionAspectSupport#invokeWithinTransaction` 来看下事务提交的代码，提交事务主要是调用 `#commitTransactionAfterReturning` -> `DataSourceTransactionManager#commit` -> `AbstractPlatformTransactionManager#processCommit`，这个方法里面处理了各种异常情况下的事务回滚，最后会调用到 `DataSourceTransactionManager#doCommit`，这个方法里面调用了 `con.commit();` 进行了事务提交。
