# spring 启动过程详细介绍

我们以 `ClassPathXmlApplicationContext` 容器为示例，来研究下容器启动过程中到底发生了什么，首先我们来看下当我们 `new ClassPathXmlApplicationContext("classpath:applicationContext.xml")` 时会发生什么，我们看下 `ClassPathXmlApplicationContext` 对应的构造方法如下

``` java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
    throws BeansException {
  // 设置父容器
  super(parent);
  // 解析配置文件路径
  setConfigLocations(configLocations);
  if (refresh) {
    // 关键方法，容器启动的主要逻辑就在这个方法里面
    refresh();
  }
}

```

其中容器启动的关键逻辑都在这个 `#refresh()` 方法中，该方法主要完成的逻辑：
- 加载 BeanDefinition
- 执行 BeanFactoryPostProcessor & BeanPostProcessor
- 初始化事件
- 初始化所有非 lazy 的单例 bean


下面是该方法的源代码
``` java
@Override
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
        // 初始化系统 placeholder，校验必须要设置的 Property 是否已经设置
		prepareRefresh();

		// 创建 BeanFactory
    // 完成所有 BeanDefinition 的加载
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// 设置 classloader, 注册 ApplicationContextAwareProcessor 等
		prepareBeanFactory(beanFactory);

		try {
			// AbstractRefreshableWebApplicationContext 实现类在这里注册 (request, session, global session, application) scope 实现
			postProcessBeanFactory(beanFactory);

			// 实例化并且初始化所有 BeanDefinitionRegistryPostProcessor 接口的实现类 （按照 PriorityOrdered, Ordered, and none order 顺序调用）
      // 实例化并且初始化所有 BeanFactoryPostProcessor 接口的实现类（按照 PriorityOrdered, Ordered, and none order 顺序调用）
			invokeBeanFactoryPostProcessors(beanFactory);

			// 注册所有的 BeanPostProcessor
			registerBeanPostProcessors(beanFactory);

			// 定义资源文件，国际化
			initMessageSource();

			// 初始化事件广播器
			initApplicationEventMulticaster();

			// 预留扩展接口
			onRefresh();

			// 注册 ApplicationEvent 事件监听器
			registerListeners();

			// 初始化所有单例并且非 lazy bean 的实例
			finishBeanFactoryInitialization(beanFactory);

			// 完成整个容器的启动，并且发布 ContextRefreshedEvent 事件
			finishRefresh();
		}
		catch (BeansException ex) {
			// ignore ...
		}
		finally {
			resetCommonCaches();
		}
	}
}
```
