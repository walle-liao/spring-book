# spring 自定义标签介绍


扩展一个 spring 自定义的标签配置大致需要以下几个步骤
- 创建一个 xsd 文件，定义自定义标签在 xml 文件中的配置规范
- 创建一个类实现 `BeanDefinitionParser` 接口，用来解析 xml 中自定义标签的配置
- 创建一个 Handler 类，扩展 `NamespaceHandlerSupport` 目的是将组件注册到 Spring 容器中
- 编写 spring.handlers 和 spring.schemas 文件
