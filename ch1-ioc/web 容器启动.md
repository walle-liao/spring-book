# web 容器启动时如何启动 spring 容器


web.xml 中的配置
``` xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

入口方法 ContextLoaderListener#contextInitialized
- ContextLoaderListener#contextInitialized
- ContextLoader#initWebApplicationContext
  - ContextLoader#createWebApplicationContext // 默认创建的是 XmlWebApplicationContext
    - ContextLoader#determineContextClass // 根据配置参数 `contextClass` 指定，或者使用默认的配置文件 `ContextLoader.properties`（就在ContextLoader所在jar相同包下）中指定的 `XmlWebApplicationContext` 作为默认的 spring web 容器。ConfigurableWebApplicationContext 接口两个重要的实现类：XmlWebApplicationContext & AnnotationConfigWebApplicationContext
  - ContextLoader#configureAndRefreshWebApplicationContext  // 配置并调用 spring 容器的 refresh 方法
      - ConfigurableWebApplicationContext#refresh  // #refresh 方法调用，启动 spring 容器


``` java
if (this.context == null) {
  // 创建 ConfigurableWebApplicationContext
  this.context = createWebApplicationContext(servletContext);
}
if (this.context instanceof ConfigurableWebApplicationContext) {
  ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
  if (!cwac.isActive()) {
    if (cwac.getParent() == null) {
      ApplicationContext parent = loadParentContext(servletContext);
      cwac.setParent(parent);
    }
    // 这里就是配置 spring 容器并且调用容器 #refresh 方法的地方
    configureAndRefreshWebApplicationContext(cwac, servletContext);
  }
}
```

createWebApplicationContext 方法

org.springframework.web.context.ContextLoader#createWebApplicationContext

org.springframework.web.context.ContextLoader#determineContextClass  // 该方法返回 WebApplicationContext 具体的类型，返回的是 org.springframework.web.context.support.XmlWebApplicationContext
``` java
protected Class<?> determineContextClass(ServletContext servletContext) {
  // 通过配置文件中 contextClass 参数来指定
  String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
  if (contextClassName != null) {
    try {
      return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
    }
    catch (ClassNotFoundException ex) {
      throw new ApplicationContextException(
          "Failed to load custom context class [" + contextClassName + "]", ex);
    }
  }
  else {
    // jar 文件中 org/springframework/web/context/ContextLoader.properties 配置中指定
    // org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
    contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
    try {
      return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
    }
    catch (ClassNotFoundException ex) {
      throw new ApplicationContextException(
          "Failed to load default context class [" + contextClassName + "]", ex);
    }
  }
}
```
