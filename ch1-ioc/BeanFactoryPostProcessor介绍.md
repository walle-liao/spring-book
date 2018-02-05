# BeanFactoryPostProcessor 介绍


### BeanFactoryPostProcessor 介绍
- 这个接口用来在容器启动过程中对 `BeanDefinition` 进行修改的一个扩展点
- 在 `BeanDefinition` 解析之后，初始之前被调用
- 参考示例： `CustomScopeConfigurer`（扩展自定义的 scope） & `PlaceholderConfigurerSupport`（对配置文件中的占位符进行解析）都是通过扩展 `BeanFactoryPostProcessor` 接口来实现

![BeanFactoryPostProcessor 主要实现类](./images/000006.png)


### BeanDefinitionRegistryPostProcessor 介绍
- 是 `BeanFactoryPostProcessor` 的另外一个比较重要的子类
- 从命名上也可以看出 `BeanDefinitionRegistryPostProcessor` 接口是用来对 `BeanDefinition` 注册进行修改的一个扩展点
- 会在 `BeanFactoryPostProcessor` 调用之前被调用
- 参考示例： `ConfigurationClassPostProcessor`（用来对 `@Configuration` 注解进行处理）& `MapperScannerConfigurer`（mybatis 中用来对所有的 DAO 接口接口生成实例）都是通过扩展 `BeanDefinitionRegistryPostProcessor` 接口来实现

![BeanDefinitionRegistryPostProcessor 主要实现类](./images/000008.png)

![BeanFactoryPostProcessor & BeanDefinitionRegistryPostProcessor 之间的关系](./images/000007.png)


### 入口方法

``` java
// 1、AbstractApplicationContext#refresh
invokeBeanFactoryPostProcessors(beanFactory);

// 2、AbstractApplicationContext#invokeBeanFactoryPostProcessors
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
  PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
}

// 3、PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors
public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

  // 1）实例化并且初始化所有 BeanDefinitionRegistryPostProcessor 接口的实现类 （按照 PriorityOrdered, Ordered, and the rest 顺序调用）
  // 2）实例化并且初始化所有 BeanFactoryPostProcessor 接口的实现类（按照 PriorityOrdered, Ordered, and the rest 顺序调用）
  // 注：所有的 BeanDefinitionRegistryPostProcessor & BeanFactoryPostProcessor 都会在这里被提前实例化（beanFactory.getBean）
}

```

### 示例：PropertyPlaceholderConfigurer



### 示例：MapperScannerConfigurer




org.springframework.beans.factory.config.PropertyPlaceholderConfigurer
org.springframework.beans.factory.config.CustomScopeConfigurer
