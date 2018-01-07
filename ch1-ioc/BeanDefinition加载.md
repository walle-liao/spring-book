# BeanDefinition 加载详解


### BeanDefinition 加载时机

org.springframework.context.support.AbstractApplicationContext#refresh
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory

org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory
两个关键代码：
    customizeBeanFactory(beanFactory);
    loadBeanDefinitions(beanFactory);

#customizeBeanFactory
```
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    if (this.allowBeanDefinitionOverriding != null) {
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.allowCircularReferences != null) {
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```

#loadBeanDefinitions(beanFactory);
两个实现
org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)
org.springframework.web.context.support.AnnotationConfigWebApplicationContext#loadBeanDefinitions
