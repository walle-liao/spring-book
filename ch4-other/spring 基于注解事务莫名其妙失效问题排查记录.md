# spring 基于注解事务莫名其妙失效问题排查记录


### 问题描述
有同事反馈我们线上系统的事务不起作用，加了 `@Transactional` 注解的方法里面出现异常之后已保存的数据不能正常回滚。
我们系统采用的是基于注解的事务配置，如下：
``` xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
	<property name="dataSource" ref="dataSource" />
</bean>

<tx:annotation-driven transaction-manager="transactionManager" />
```

### 问题分析
事务不起作用，一般怀疑就是 service 代理对象创建出了问题，断点调试 spring 容器启动代码 `AbstractApplicationContext#refresh` 发现在执行到 `registerBeanPostProcessors(beanFactory);` 代码时已经开始创建系统内部 bean (com.qfang 包下的类对象)，继续 debug 进该方法，跟踪到了 `PostProcessorRegistrationDelegate#registerBeanPostProcessors` 方法，该方法的主要作用是注册系统中所有的 `BeanPostProcessor` 接口，关于该方法的详见介绍，参考后面的示例

> 需要强调的一点是 service 事务代理对象的创建也是通过 `BeanPostProcessor` 接口来实现的，具体可以参考 `AopAutoProxyConfigurer#configureAutoProxyCreator` 里面的代码，第一行就是  `AopNamespaceUtils.registerAutoProxyCreatorIfNecessary(parserContext, element);` 用来注册 `AbstractAdvisorAutoProxyCreator`，而所有的 `AbstractAutoProxyCreator` 都是实现了 `BeanPostProcessor` 接口  

继续 debug 该方法，发现该方法内会实例化 `com.qfang.service.center.entry.collect.Collector` 这个类对象，该类的定义
``` java
public class Collector extends AbstractBeanFactoryPointcutAdvisor implements
ApplicationListener<ApplicationEvent>,BeanPostProcessor,InitializingBean {}
```

这个类实现了 `BeanPostProcessor` 接口，在这里被实例化非常正常，并且这个类是继承了 `AbstractBeanFactoryPointcutAdvisor` 抽象类，该抽象类又实现了 `Ordered` 接口，所以 `Collector` 类的实例化会在第二部分（实例化所有实现了 `Ordered` 接口 `BeanPostProcessor` 时）被实例化，问题的关键是，该类引用了 `ExecutorServiceHolder` 实例，`ExecutorServiceHolder` 类又继承了 `BaseServiceImpl`, `BaseServiceImpl` 中有又依赖了 `CityTypeTools`、`SystemUserInfo`、`CurrentCity`，而 `SystemUserInfo` 又依赖了 `PersonService`，`PersonService` 又依赖了很多其他的 service，这样导致问题的关键就来了，**在创建 `Collector` 类实例的时候会因为依赖关系创建很大一部分类的实例，这其中涉及到很多 service 类的对象，** 而此时由于用来创建事务代理对象的 `BeanPostProcessor` (`org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator`) 实例还没有被创建，所有导致此时创建的 service 对象无法生成被代理。
> 通常为 service 对象生成事务代理对象的入口的代码调用链 `AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization`   
-> `AbstractAutoProxyCreator#postProcessAfterInitialization`   
-> `AbstractAutoProxyCreator#wrapIfNecessary` 这个方法可以用来返回一个代理对象，用来替换原 service 对象

而用于生成 service 事务代理对象的 `BeanPostProcessor` 需要在 `Collector` 类被实例化之后才会被实例化，但是由于 spring 的单例模式，service 对象一但被创建出来后，下次在其他类中再被引用时，会直接获取已经创建的对象实例（那些在 `Collector` 实例被创建时，被关联创建出来的 service 都是没有代理的，特别是被 `SystemUserInfo` 关联的 `PersonService`，以及被 `PersonService` 关联的，以及后续传递的关联的所有的对象都不会被代理），这也就是为什么注释掉 `BaseServiceImpl` 中 `SystemUserInfo` 的 `@Autowired` 事务就会正常起作用的原因，而且即使不注释掉 `BaseServiceImpl` 中 `SystemUserInfo` 的 `@Autowired` 也不是所有的 service 的事务都会失效，而是被 `PersonService` 关联，以及关联对象所关联的 service 不会有事务 *（如果你新创建一个 service 在系统的其他地方都没有被引用，去测试其事务特性，一样可以正常工作）*，

**原因总结：**
`com.qfang.service.center.entry.collect.Collector` 类是一个 `BeanPostProcessor`，而在 `Collector` 中又注入了系统中的类（com.qfang），有用实例化依赖对象的传递关系时，导致很多系统中的类被过早地实例化，而没有被正确地创建事务代理对象，然后因为 spring 单例的缘故，实例化之后不会再次实例化，所以导致很多 service 对象的事务代理对象没有正常生成。

### 解决办法
找到引起问题的原因，其实要解决问题就很简单了，直接修改 `Collector` 类，不要 `@Autowired ExecutorServiceHolder executorServiceHolder;` 而是在运行时通过容器中取获取该类的实例就可以了，具体修改见 svn....


### 总结
在自己实现 `BeanPostProcessor` 接口时，要非常小心，尽量不要去引用注入系统中的类，因为在 spring 容器启动过程中 `BeanPostProcessor` 类的实例化是非常靠前的，这些类是属于容器的一些基础功能类，所以一般会早于系统类实例化之前被实例化 *(关于spring容器启动过程详细步骤说明，参考：[spring 容器启动过程](../ch1-ioc/spring容器启动过程.md))*，而且系统中的类的依赖关系非常复杂，很可能因为一个依赖对象的创建导致很多其他关联对象被创建，如果这些关联的对象被过早的时候，很可能不是完整功能的实例。
所以除非你非常清楚依赖直接的传递，最好不要在自己扩展的 `BeanPostProcessor` 类中依赖注入系统的类，同样其他 spring 容器扩展的一些基础接口类在使用过程中也会有类似的问题。


### 附录
#### PostProcessorRegistrationDelegate#registerBeanPostProcessors 方法说明
``` java
// 1. 获取所有 BeanPostProcessor 接口对应的 bean name
String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

// Separate between BeanPostProcessors that implement PriorityOrdered,
// Ordered, and the rest.
// 2. 判断这些 BeanPostProcessor，区分实现了 PriorityOrdered，Ordered，未实现 Ordered 接口，分别放到不同的 List 中
// 优先实例化实现了 PriorityOrdered 接口的 BeanPostProcessor，实例化之后进行排序，然后注册到 beanFactory 中
List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
List<BeanPostProcessor> internalPostProcessors = new ArrayList<BeanPostProcessor>();
List<String> orderedPostProcessorNames = new ArrayList<String>();
List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
for (String ppName : postProcessorNames) {
	if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
		BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
		priorityOrderedPostProcessors.add(pp);
		if (pp instanceof MergedBeanDefinitionPostProcessor) {
			internalPostProcessors.add(pp);
		}
	}
	else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
		orderedPostProcessorNames.add(ppName);
	}
	else {
		nonOrderedPostProcessorNames.add(ppName);
	}
}

sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

// 3. 实例化实现了 Ordered 接口的 BeanPostProcessor，排序后注册
List<BeanPostProcessor，排序后注册> orderedPostProcessors = new ArrayList<BeanPostProcessor>();
for (String ppName : orderedPostProcessorNames) {
	BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
	orderedPostProcessors.add(pp);
	if (pp instanceof MergedBeanDefinitionPostProcessor) {
		internalPostProcessors.add(pp);
	}
}
sortPostProcessors(beanFactory, orderedPostProcessors);
registerBeanPostProcessors(beanFactory, orderedPostProcessors);

// 4、实例化剩下的未实现任何 Ordered 接口的 BeanPostProcessor 后注册
List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanPostProcessor>();
for (String ppName : nonOrderedPostProcessorNames) {
	BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
	nonOrderedPostProcessors.add(pp);
	if (pp instanceof MergedBeanDefinitionPostProcessor) {
		internalPostProcessors.add(pp);
	}
}
registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

// 对 MergedBeanDefinitionPostProcessor 重新排序，并注册
sortPostProcessors(beanFactory, internalPostProcessors);
registerBeanPostProcessors(beanFactory, internalPostProcessors);

beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
```

简单来看该方法的主要步骤包括：
- 获取所有 `BeanPostProcessor` 接口对应的 bean name
- 区分对待实现了 `PriorityOrdered`，`Ordered`，未实现 `Ordered` 接口，分别放到不同的 `List` 中，然后先后实例化
- 优先实例化实现了 `PriorityOrdered` 接口的 ` BeanPostProcessor`，实例化之后进行排序，然后注册到 beanFactory 中
- 实例化实现了 `Ordered` 接口的 `BeanPostProcessor`，排序后注册
- 实例化剩下的未实现任何 `Ordered` 接口的 `BeanPostProcessor` 后注册



> 关于 spring 容器启动过程代码分析，参考：[spring 容器启动过程](../ch1-ioc/spring容器启动过程.md)  
> 关于 tx:annotation-driven 标签的解析，参考：[spring 自定义标签介绍](../ch1-ioc/spring自定义标签介绍.md)]
> 关于 spring 中基于注解事务的详细原理参考：[spring 基于注解事务详见](../ch2-aop\spring基于注解事务.md)
