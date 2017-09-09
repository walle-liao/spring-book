# spring 自定义标签介绍

我们使用 spring xml 配置时，经常会使用到类似 `<context:component-scan base-package="" />`、`<context:property-placeholder location="" />`、`<tx:annotation-driven transaction-manager="" />` 这样的配置，本文主要剖析这些标签到底是怎样取作用的，并且通过一个示例来实际讲解下如何定制自己的 spring 的标签

没一个 spring 的自定义标签都可以找到一个对应的 `NamespaceHandler`，例如 `<context:xxx />` 标签对应的是 `ContextNamespaceHandler`，`<tx:xx />` 标签对应的是 `TxNamespaceHandler`，


扩展一个 spring 自定义的标签配置大致需要以下几个步骤
- 创建一个 xsd 文件，定义自定义标签在 xml 文件中的配置规范
- 创建一个类实现 `BeanDefinitionParser` 接口，用来解析 xml 中自定义标签的配置
- 创建一个 Handler 类，扩展 `NamespaceHandlerSupport` 目的是将组件注册到 Spring 容器中
- 编写 spring.handlers 和 spring.schemas 文件
