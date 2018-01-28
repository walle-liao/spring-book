# BeanDefinition 加载详解


### BeanDefinition 加载过程

- org.springframework.context.support.AbstractApplicationContext#refresh
- org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory
- org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory
- org.springframework.context.support.AbstractRefreshableApplicationContext#loadBeanDefinitions
  - org.springframework.context.support.AbstractXmlApplicationContext#loadBeanDefinitions  // 基于 xml 方式的 bean BeanDefinition 加载
  - org.springframework.web.context.support.AnnotationConfigWebApplicationContext#loadBeanDefinitions  // 基于注解形式的 bean BeanDefinition 加载
- org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String...)  // configResources 为 null，基于 configLocations 配置的 location 加载
- org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(java.lang.String, java.util.Set<org.springframework.core.io.Resource>)  // 根据 location 加载到具体的 Resource
- org.springframework.beans.factory.support.AbstractBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource...)
- org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.Resource)
- org.springframework.beans.factory.xml.XmlBeanDefinitionReader#loadBeanDefinitions(org.springframework.core.io.support.EncodedResource)  // 默认情况下 encoding & charset 都是 null
- org.springframework.beans.factory.xml.XmlBeanDefinitionReader#doLoadBeanDefinitions  // 根据 InputSource 加载 xml Document 树，Document 加载完成之后开始解析 BeanDefinition
- org.springframework.beans.factory.xml.XmlBeanDefinitionReader#registerBeanDefinitions  // documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#doRegisterBeanDefinitions  // parseBeanDefinitions(root, this.delegate);
- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseBeanDefinitions
  - org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#parseDefaultElement  // 解析 spring 默认的标签元素(e.g. `import`, `alias`, `bean`, `beans`)
  - org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseCustomElement(org.w3c.dom.Element)  // 解析扩展/自定义的标签元素（e.g. `<tx:xxx />`），参考自定义标签解析部分
- org.springframework.beans.factory.xml.DefaultBeanDefinitionDocumentReader#processBeanDefinition  // 解析 `<bean />` 标签
- org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionElement(org.w3c.dom.Element, org.springframework.beans.factory.config.BeanDefinition)
- org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionElement(org.w3c.dom.Element, java.lang.String, org.springframework.beans.factory.config.BeanDefinition)
- org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parseBeanDefinitionAttributes  // 解析 bean 的各种属性
- org.springframework.beans.factory.xml.BeanDefinitionParserDelegate#parsePropertyElement  // 解析 bean 的 property 属性

----------
PS:
org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory  
#customizeBeanFactory 方法
```
protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
    if (this.allowBeanDefinitionOverriding != null) {
        // 是否允许 BeanDefinitionOverriding （？）
        beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.allowCircularReferences != null) {
        // 设置 bean 之间是否可以循环引用
        beanFactory.setAllowCircularReferences(this.allowCircularReferences);
    }
}
```
