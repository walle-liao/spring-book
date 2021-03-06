# BeanPostProcessor 介绍


### spring BeanPostProcessor 接口体系

![spring BeanPostProcessor 接口体系](./images/000017.png)


#### InstantiationAwareBeanPostProcessor
接口定义如下：  
``` java
// postProcessBeforeInstantiation方法的作用在目标对象被实例化之前调用的方法，可以返回目标实例的一个代理用来代替目标实例
// beanClass参数表示目标对象的类型，beanName是目标实例在Spring容器中的name
// 返回值类型是Object，如果返回的是非null对象，接下来除了postProcessAfterInitialization方法会被执行以外，其它bean构造的那些方法都不再执行。否则那些过程以及postProcessAfterInitialization方法都会执行
Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

// postProcessAfterInstantiation方法的作用在目标对象被实例化之后并且在属性值被populate之前调用
// bean参数是目标实例(这个时候目标对象已经被实例化但是该实例的属性还没有被设置)，beanName是目标实例在Spring容器中的name
// 返回值是boolean类型，如果返回true，目标实例内部的返回值会被populate，否则populate这个过程会被忽视
boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

// postProcessPropertyValues方法的作用在属性中被设置到目标实例之前调用，可以修改属性的设置
// pvs参数表示参数属性值(从BeanDefinition中获取)，pds代表参数的描述信息(比如参数名，类型等描述信息)，bean参数是目标实例，beanName是目标实例在Spring容器中的name
// 返回值是PropertyValues，可以使用一个全新的PropertyValues代替原先的PropertyValues用来覆盖属性设置或者直接在参数pvs上修改。如果返回值是null，那么会忽略属性设置这个过程(所有属性不论使用什么注解，最后都是null)
PropertyValues postProcessPropertyValues(
    PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName)
    throws BeansException;
```

总结：
- InstantiationAwareBeanPostProcessor接口继承BeanPostProcessor接口，它内部提供了3个方法，再加上BeanPostProcessor接口内部的2个方法，所以实现这个接口需要实现5个方法。InstantiationAwareBeanPostProcessor接口的主要作用在于目标对象的实例化过程中需要处理的事情，包括实例化对象的前后过程以及实例的属性设置
- postProcessBeforeInstantiation方法是最先执行的方法，它在目标对象实例化之前调用，该方法的返回值类型是Object，我们可以返回任何类型的值。由于这个时候目标对象还未实例化，所以这个返回值可以用来代替原本该生成的目标对象的实例(比如代理对象)。如果该方法的返回值代替原本该生成的目标对象，后续只有postProcessAfterInitialization方法会调用，其它方法不再调用；否则按照正常的流程走
- postProcessAfterInstantiation方法在目标对象实例化之后调用，这个时候对象已经被实例化，但是该实例的属性还未被设置，都是null。如果该方法返回false，会忽略属性值的设置；如果返回true，会按照正常流程设置属性值
- postProcessPropertyValues方法对属性值进行修改(这个时候属性值还未被设置，但是我们可以修改原本该设置进去的属性值)。如果postProcessAfterInstantiation方法返回false，该方法不会被调用。可以在该方法内对属性值进行修改（参考示例：`CommonAnnotationBeanPostProcessor` & `AutowiredAnnotationBeanPostProcessor` 中处理 `@Resource` & `@Autowired` 这些注解属性的做法）
- 父接口BeanPostProcessor的2个方法postProcessBeforeInitialization和postProcessAfterInitialization都是在目标对象被实例化之后，并且属性也被设置之后调用的
- Instantiation表示实例化，Initialization表示初始化。实例化的意思在对象还未生成，初始化的意思在对象已经生成

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


### SmartInstantiationAwareBeanPostProcessor
接口的定义：
``` java
// 预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null
Class<?> predictBeanType(Class<?> beanClass, String beanName) throws BeansException;
// 选择合适的构造器，比如目标对象有多个构造器，在这里可以进行一些定制化，选择合适的构造器
// beanClass参数表示目标实例的类型，beanName是目标实例在Spring容器中的name
// 返回值是个构造器数组，如果返回null，会执行下一个PostProcessor的determineCandidateConstructors方法；否则选取该PostProcessor选择的构造器
Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException;
// 获得提前暴露的bean引用。主要用于解决循环引用的问题
// 只有单例对象才会调用此方法
Object getEarlyBeanReference(Object bean, String beanName) throws BeansException;
```

总结：
- SmartInstantiationAwareBeanPostProcessor接口继承InstantiationAwareBeanPostProcessor接口，它内部提供了3个方法，再加上父接口的5个方法，所以实现这个接口需要实现8个方法。SmartInstantiationAwareBeanPostProcessor接口的主要作用也是在于目标对象的实例化过程中需要处理的事情。它是InstantiationAwareBeanPostProcessor接口的一个扩展。主要在Spring框架内部使用
- predictBeanType方法用于预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null。主要在于BeanDefinition无法确定Bean类型的时候调用该方法来确定类型
- determineCandidateConstructors方法用于选择合适的构造器，比如类有多个构造器，可以实现这个方法选择合适的构造器并用于实例化对象。该方法在postProcessBeforeInstantiation方法和postProcessAfterInstantiation方法之间调用，如果postProcessBeforeInstantiation方法返回了一个新的实例代替了原本该生成的实例，那么该方法会被忽略
- getEarlyBeanReference主要用于解决循环引用问题。比如ReferenceA实例内部有ReferenceB的引用，ReferenceB实例内部有ReferenceA的引用。首先先实例化ReferenceA，实例化完成之后提前把这个bean暴露在ObjectFactory中，然后populate属性，这个时候发现需要ReferenceB。然后去实例化ReferenceB，在实例化ReferenceB的时候它需要ReferenceA的实例才能继续，这个时候就会去ObjectFactory中找出了ReferenceA实例，ReferenceB顺利实例化。ReferenceB实例化之后，ReferenceA的populate属性过程也成功完成，注入了ReferenceB实例。提前把这个bean暴露在ObjectFactory中，这个ObjectFactory获取的实例就是通过getEarlyBeanReference方法得到的
- 参考示例：`AbstractAutoProxyCreator`

#### BeanPostProcessor
接口定义如下：
``` java
// 实例化、依赖注入完成，初始化之前被调用
Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;

// 实例化、依赖注入完成，初始化之后被调用
Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
```


总结：
- postProcessBeforeInitialization是指bean在初始化之前需要调用的方法
- postProcessAfterInitialization是指bean在初始化之后需要调用的方法
- postProcessBeforeInitialization和postProcessAfterInitialization方法被调用的时候。这个时候bean已经被实例化，并且所有该注入的属性都已经被注入，是一个完整的bean
- 这2个方法的返回值可以是原先生成的实例bean，或者使用wrapper包装这个实例


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

![spring bean 实例化过程](./images/000016.png)

### DestructionAwareBeanPostProcessor
接口的定义：
``` java
// 该方法是bean在Spring在容器中被销毁之前调用
void postProcessBeforeDestruction(Object bean, String beanName) throws BeansException;
```

### MergedBeanDefinitionPostProcessor
接口的定义：
``` java
// 该方法是bean在合并Bean定义之后调用
void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName);
```

参考：  
1）[Spring内部的BeanPostProcessor接口总结](https://fangjian0423.github.io/2017/06/20/spring-bean-post-processor/)  
2）[Spring内置的BeanPostProcessor总结](https://fangjian0423.github.io/2017/06/24/spring-embedded-bean-post-processor/)
