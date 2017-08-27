# BeanPostProcessor 介绍


### 相关接口

#### BeanPostProcessor
接口定义如下：
``` java
// 实例化、依赖注入完成，初始化之前被调用
Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

// 实例化、依赖注入完成，初始化之后被调用
Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;

```

接口方法被调用的入口：  
`AbstractAutowireCapableBeanFactory#initializeBean`，部分源代码如下：
``` java
Object wrappedBean = bean;
if (mbd == null || !mbd.isSynthetic()) {
	// 执行 #postProcessBeforeInitialization 方法
	wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
}

try {
	// 执行 InitializingBean#afterPropertiesSet() 或者自定义配置的 init 方法
	invokeInitMethods(beanName, wrappedBean, mbd);
}
catch (Throwable ex) {
}

if (mbd == null || !mbd.isSynthetic()) {
	// 执行 #postProcessAfterInitialization 方法
	wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
}
```


#### InstantiationAwareBeanPostProcessor
接口定义如下：  
``` java
/**
* 在目标对象实例化之前调用，该方法的返回值类型是Object，我们可以返回任何类型的值。
* 由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(比如代理对象)。
* 如果该方法的返回值代替原本该生成的目标对象，后续只有 postProcessAfterInitialization 方法会调用，其它方法不再调用；
* 否则按照正常的流程走
* @param beanClass: 表示目标对象的类型
* @param beanName: 是目标对象在 spring 容器中对应的 name
* @return 如果返回的是非 null 对象，接下来除了 postProcessAfterInitialization 方法会被执行以外，
* 其它 bean 构造的那些方法都不再执行。否则那些过程以及 postProcessAfterInitialization 方法都会执行
*/
Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

/**
* 在目标对象实例化之后调用，这个时候对象已经被实例化，但是该对象的属性和依赖的对象还未被设值
* 该方法在 AbstractAutowireCapableBeanFactory#populateBean 方法中设置属性值之前有被调用
* 如果该方法返回 false，会忽略属性值的设置
* 如果返回true，会按照正常流程设置属性值
*/
boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
			throws BeansException;

```

接口方法被调用的入口：  
`AbstractAutowireCapableBeanFactory#createBean` 在调用 #doCreateBean 创建真实的 bean 对象之前，先调用 `AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation` 方法，可以用来创建代理对象等

``` java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
  Object bean = null;
  if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
    // Make sure bean class is actually resolved at this point.
    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
      Class<?> targetType = determineTargetType(beanName, mbd);
      if (targetType != null) {
        // 调用 postProcessBeforeInstantiation
        bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
        if (bean != null) {
          // 如果创建了代理对象，则直接调用 postProcessAfterInstantiation
          bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
        }
      }
    }
    mbd.beforeInstantiationResolved = (bean != null);
  }
  return bean;
}
```
另外 `#postProcessPropertyValues()` 方法在
`AbstractAutowireCapableBeanFactory#populateBean` 中也被调用了（初始化依赖对象之前调用）

- InstantiationAwareBeanPostProcessor
  - postProcessBeforeInstantiation(Class<?> beanClass, String beanName)
  - postProcessAfterInstantiation(Object bean, String beanName)
  - postProcessPropertyValues

- SmartInstantiationAwareBeanPostProcessor
  - predictBeanType
  - determineCandidateConstructors
  - getEarlyBeanReference


参考：  
[Spring内部的BeanPostProcessor接口总结](https://fangjian0423.github.io/2017/06/20/spring-bean-post-processor/)
