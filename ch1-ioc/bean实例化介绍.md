# bean 实例化过程介绍


- org.springframework.context.support.AbstractApplicationContext#refresh  // finishBeanFactoryInitialization(beanFactory);
- org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization  // beanFactory.preInstantiateSingletons();
- org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons  // isFactoryBean & getBean(beanName);
- org.springframework.beans.factory.support.AbstractBeanFactory#doGetBean
  - Object sharedInstance = getSingleton(beanName);  // 先检查该 bean 是否已经实例化
  - parentBeanFactory.getBean(nameToLookup, requiredType);  // 再检查 parentBeanFactory 是否存在该 bean
  - String[] dependsOn = mbd.getDependsOn();  getBean(dependsOnBean);  // 先实例化该 bean 依赖的 bean
  - isSingleton()  new ObjectFactory() {createBean(beanName, mbd, args);}  // singleton bean 实例创建
  - isPrototype()  createBean(beanName, mbd, args);  // prototype bean 创建
  - customer scope
- org.springframework.beans.factory.support.DefaultSingletonBeanRegistry#getSingleton  // singletonObject = singletonFactory.getObject();
- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBean
  - Object bean = resolveBeforeInstantiation(beanName, mbdToUse);  // BeanPostProcessors 可以在这里返回代理对象
  - *Object beanInstance = doCreateBean(beanName, mbdToUse, args);  // 正常情况下*
- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#doCreateBean
  - *instanceWrapper = createBeanInstance(beanName, mbd, args);  // 创建 bean 实例*
  - populateBean(beanName, mbd, instanceWrapper);  // bean 属性初始化
  - exposedObject = initializeBean(beanName, exposedObject, mbd);  // bean 的初始化方法执行 BeanPostProcessor#postProcessBeforeInitialization & #afterPropertiesSet, initmethod & beanProcessor.postProcessAfterInitialization
- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#createBeanInstance
- org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#instantiateBean  // getInstantiationStrategy().instantiate(mbd, beanName, parent);
  - SimpleInstantiationStrategy  // 普通对象创建
  - CglibSubclassingInstantiationStrategy  // cglib 方式创建代理对象
- org.springframework.beans.factory.support.SimpleInstantiationStrategy#instantiate  // 获取无参的默认构造方法，然后创建实例
