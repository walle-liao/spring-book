# spring xml 配置 bean 自定义属性值如何被赋值

平时开发中我们经常会使用到 xml 中配置的 bean 属性是通过其他 properties 文件指定的情况，那么 spring 到底是如何将 properties 文件中配置的值设置到 bean 对应的属性中的呢？

本文就 spring xml 配置中 bean 自定义属性的加载进行详细讲解。

spring 对自定属性进行赋值的关键实现类为 `PropertyPlaceholderConfigurer`。

### 先从一个简单示例代码入手

`User.java` 一个普通的 java bean，我们测试通过 properties 文件来配置该 bean 的属性
``` java
public class User {

    private String username;
    private String password;

    // getter and setter...

    @Override
    public String toString() {
        return "User{" +
                "username='" + username + '\'' +
                ", password='" + password + '\'' +
                '}';
    }
}
```

`MD5EncryptedPropertyPlaceholderConfigurer.java` 扩展自默认的 `PropertyPlaceholderConfigurer`，用于后面讲解对 properties 属性的特殊处理
``` java
public class MD5EncryptedPropertyPlaceholderConfigurer extends PropertyPlaceholderConfigurer {

    private static final String PASSWORD = "password";

    @Override
    protected String convertProperty(String propertyName, String propertyValue) {
        if(PASSWORD.equals(propertyName)) {
            // 对配置文件中 password 属性进行加密的功能，配置文件中配置的是明文，但是设置到的 bean 属性中时是密文
            return MD5Utils.encrypted(propertyValue);
        }
        return super.convertProperty(propertyName, propertyValue);
    }

}
```

applicationContext.xml 配置文件
``` xml
<!-- 使用我们自定义的 PropertyPlaceholder -->
<bean id="propertyConfigurer" class="com.qfang.examples.spring.ioc.property.MD5EncryptedPropertyPlaceholderConfigurer">
    <property name="location" value="ioc/property/system.properties" />
    <property name="fileEncoding" value="UTF-8" />
</bean>

<bean id="user" class="com.qfang.examples.spring.ioc.property.User">
    <property name="username" value="${username}"/>
    <property name="password" value="${password}"/>
</bean>
```

system.properties 配置
```
username=test
password=123456
```

测试类
``` java
public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("ioc/property/applicationContext.xml");
    User user = context.getBean(User.class);
    System.out.println(user.toString());
}
```

运行结果：
> User{username='test', password='e10adc3949ba59abbe56e057f20f883e'}   
> 两个属性都被正确设值，并且 password 属性对读取的值进行了 MD5 加密


### 分析 `PropertyPlaceholderConfigurer` 如何对配置文件中的占位符属性进行替换

1、`PropertyPlaceholderConfigurer` 是继承自 `BeanFactoryPostProcessor` 接口，那其对应的接口实现方法会在容器启动（`#refresh`方法）的 `invokeBeanFactoryPostProcessors(beanFactory);` 被调用

![PropertyPlaceholderConfigurer 继承关系](./images/000011.png)

2、接下来我们看下 `PropertyPlaceholderConfigurer` 中 `#postProcessBeanFactory` 接口实现方法（BeanFactoryPostProcessor 接口对应的实现方法）

``` java
// PropertyResourceConfigurer#postProcessBeanFactory
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
  try {
    // 加载并读取 properties 配置文件
    // 检查否允许（localOverride 属性）本地应用的一些配置来替换 jar 包里面配置等，如果允许则会将本地配置值替换默认的配置值
    Properties mergedProps = mergeProperties();

    // 用来将配置文件中配置的值做一些转换
    // 例如：一些密码在配置文件中配置的是密文，那么可以在这里将密文转换成对应的明文
    convertProperties(mergedProps);

    // 对配置文件中的占位符属性进行替换的关键代码
    // 真正意义上就是将 BeanDefinition 中对应的 PropertyValue 里的 value 属性值进行替换
    processProperties(beanFactory, mergedProps);
  }
  catch (IOException ex) {
    throw new BeanInitializationException("Could not load properties", ex);
  }
}
```

BeanDefinition 中 PropertyValue 替换之前的值，可以看到 username 属性对应的值为 `TypedStringValue` 类型，在替换之前的值为 `${username}`

![PropertyValue 替换之前的值](./images/000009.png)


3、`#processProperties` 方法，将 `TypedStringValue` 替换成具体值的关键方法

org.springframework.beans.factory.config.PropertyPlaceholderConfigurer#processProperties
``` java
protected void processProperties(ConfigurableListableBeanFactory beanFactoryToProcess, Properties props)
    throws BeansException {

  StringValueResolver valueResolver = new PlaceholderResolvingStringValueResolver(props);
  doProcessProperties(beanFactoryToProcess, valueResolver);
}
```

org.springframework.beans.factory.config.PlaceholderConfigurerSupport#doProcessProperties
``` java
protected void doProcessProperties(ConfigurableListableBeanFactory beanFactoryToProcess,
    StringValueResolver valueResolver) {

  // visitor 模式
  BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);

  String[] beanNames = beanFactoryToProcess.getBeanDefinitionNames();
  for (String curName : beanNames) {
    if (!(curName.equals(this.beanName) && beanFactoryToProcess.equals(this.beanFactory))) {
      // 对所有的 BeanDefinition 进行属性替换
      BeanDefinition bd = beanFactoryToProcess.getBeanDefinition(curName);
      visitor.visitBeanDefinition(bd);
    }
  }
}
```

org.springframework.beans.factory.config.BeanDefinitionVisitor#visitBeanDefinition
``` java
// 这个方法的实现就决定了 BeanDefinition 中那些对方可以使用占位符属性
// 从这个方法我们就可以知道 spring bean 的配置中，不光光只有属性可以使用占位符方式配置，其他很多属性都可以使用占位符动态配置
public void visitBeanDefinition(BeanDefinition beanDefinition) {
  visitParentName(beanDefinition);
  visitBeanClassName(beanDefinition);
  visitFactoryBeanName(beanDefinition);
  visitFactoryMethodName(beanDefinition);
  visitScope(beanDefinition);
  // 对属性占位符值进行替换的地方
  visitPropertyValues(beanDefinition.getPropertyValues());
  ConstructorArgumentValues cas = beanDefinition.getConstructorArgumentValues();
  visitIndexedArgumentValues(cas.getIndexedArgumentValues());
  visitGenericArgumentValues(cas.getGenericArgumentValues());
}
```

org.springframework.beans.factory.config.BeanDefinitionVisitor#visitPropertyValues
``` java
protected void visitPropertyValues(MutablePropertyValues pvs) {
  PropertyValue[] pvArray = pvs.getPropertyValues();
  for (PropertyValue pv : pvArray) {
    // 将 BeanDefinition 中的 PropertyValue 解析成具体的值
    Object newVal = resolveValue(pv.getValue());
    if (!ObjectUtils.nullSafeEquals(newVal, pv.getValue())) {
      pvs.add(pv.getName(), newVal);
    }
  }
}
```

org.springframework.beans.factory.config.BeanDefinitionVisitor#resolveValue
``` java
protected Object resolveValue(Object value) {
		if (value instanceof BeanDefinition) {
			visitBeanDefinition((BeanDefinition) value);
		}
		else if (value instanceof BeanDefinitionHolder) {
			visitBeanDefinition(((BeanDefinitionHolder) value).getBeanDefinition());
		}
		else if (value instanceof RuntimeBeanReference) {
			RuntimeBeanReference ref = (RuntimeBeanReference) value;
			String newBeanName = resolveStringValue(ref.getBeanName());
			if (!newBeanName.equals(ref.getBeanName())) {
				return new RuntimeBeanReference(newBeanName);
			}
		}
		else if (value instanceof RuntimeBeanNameReference) {
			RuntimeBeanNameReference ref = (RuntimeBeanNameReference) value;
			String newBeanName = resolveStringValue(ref.getBeanName());
			if (!newBeanName.equals(ref.getBeanName())) {
				return new RuntimeBeanNameReference(newBeanName);
			}
		}
		else if (value instanceof Object[]) {
			visitArray((Object[]) value);
		}
		else if (value instanceof List) {
			visitList((List) value);
		}
		else if (value instanceof Set) {
			visitSet((Set) value);
		}
		else if (value instanceof Map) {
			visitMap((Map) value);
		}
		else if (value instanceof TypedStringValue)
      // 一般情况下属性占位符的值都是这种类型 {name: username, value: ${username}}，替换之后会将 ${username} 替换成真正的配置文件中的值
			TypedStringValue typedStringValue = (TypedStringValue) value;
			String stringValue = typedStringValue.getValue();
			if (stringValue != null) {
				String visitedString = resolveStringValue(stringValue);
				typedStringValue.setValue(visitedString);
			}
		}
		else if (value instanceof String) {
			return resolveStringValue((String) value);
		}
		return value;
	}
```

4、再回到 `PropertyResourceConfigurer#postProcessBeanFactory` 方法中的 `#convertProperties` 方法，该方法可以在 properties 属性值设置给 bean 属性之前，对属性值进行一些处理，例如：如果配置文件中配置的密码为明文，而想要设置给 bean 属性中的密码进行不同方式地加密，那么久可以通过覆盖 `#convertProperty` 方法在属性设置到 bean 之前对属性值进行一些处理

``` java
protected void convertProperties(Properties props) {
  // props = {username: test, password: 123456}
  Enumeration<?> propertyNames = props.propertyNames();
  while (propertyNames.hasMoreElements()) {
    String propertyName = (String) propertyNames.nextElement();  // password
    String propertyValue = props.getProperty(propertyName);   // 123456
    // convertProperty 方法，调用到  MyExtPropertyPlaceholderConfigurer#convertProperty 方法，对属性名称为 password 的属性进行加密处理，返回的 value 为 MD5 加密后的值
    String convertedValue = convertProperty(propertyName, propertyValue);  // e10adc3949ba59abbe56e057f20f883e
    if (!ObjectUtils.nullSafeEquals(propertyValue, convertedValue)) {
      props.setProperty(propertyName, convertedValue);
    }
  }
}
```

自定义 PropertyPlaceholder 来实现对属性文件中的 password 属性加密功能
``` java
public class MyExtPropertyPlaceholderConfigurer extends PropertyPlaceholderConfigurer {

    private static final String PASSWORD = "password";

    @Override
    protected String convertProperty(String propertyName, String propertyValue) {
        if(PASSWORD.equals(propertyName)) {
            // 对配置文件中的明文进行加密的功能
            // 配置文件中配置的是明文，但是真正替换的 bean 属性中时是密文
            return MD5Utils.encrypted(propertyValue);
        }
        return super.convertProperty(propertyName, propertyValue);
    }

}
```
