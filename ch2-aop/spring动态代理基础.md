# spring 动态代理基础

spring 实现了两种动态代理模式
- jdk 动态代理
- cglib 动态代理

### jdk 动态代理

示例

1、首先需要定义一个接口以及具体的实现类
``` java
// 首先需要定义一个接口
public interface IService {
    void sayHello();
}

// 接口的具体实现类
public class RealService implements IService {

    @Override
    public void sayHello() {
        System.out.println("hello world");
    }

}
```

2、jdk 动态代理类的实现
``` java
public class DynamicProxyService implements InvocationHandler {

    private Object target;

    public Object bind(Object target) {
        this.target = target;
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(), this);
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before say hello");
        return method.invoke(target, args);
    }

}
```

3、测试代理类
``` java
public class JdkProxyMain {

    public static void main(String[] args) {
        DynamicProxyService proxyService = new DynamicProxyService();
        IService service = (IService) proxyService.bind(new RealService());
        service.sayHello();
        System.out.println(service.getClass());
    }

}
```
4、运行结果
![jdk 动态代理测试运行结果](./images/000013.png)

可以看到 jdk 动态代理生成的类的类型为 `com.sun.proxy.$Proxy0`

PS：这里只是简单的 jdk 动态代理的使用示例，需要深入了解 jdk 动态代理的实现原理参考有道云笔记（）


-----------

### cglib 的动态代理
CGLIB是一个强大的、高性能的代码生成库。其被广泛应用于AOP框架（Spring、dynaop）中，用以提供方法拦截操作。  
CGLIB代理主要通过对字节码的操作，为对象引入间接级别，以控制对象的访问。我们知道Java中有一个动态代理也是做这个事情的，那我们为什么不直接使用Java动态代理，而要使用CGLIB呢？答案是CGLIB相比于JDK动态代理更加强大，JDK动态代理虽然简单易用，但是其有一个致命缺陷是，只能对接口进行代理。如果要代理的类为一个普通类、没有接口，那么Java动态代理就没法使用了。

![jdk 动态代理测试运行结果](./images/000014.png)

示例：

1、普通的 java 类
``` java
// 普通的类就可以代理，不需要实现接口
public class RealService {

    public void saveUser() {
        System.out.println("save user");
    }

    public void findUser() {
        System.out.println("find user");
    }

}
```

2、cglib 代理类
``` java
public class CglibProxyService implements MethodInterceptor {

    private Object target;

    public CglibProxyService(Object target) {
        this.target = target;
    }

    public Object getProxy() {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallbacks(new Callback[] {NoOp.INSTANCE, this});
        enhancer.setCallbackFilter(method -> {
            if(method.getName().startsWith("save"))  // 拦截所有以 save 开头的方法
                return 1;
            return 0;
        });
        return enhancer.create();
    }

    @Override
    public Object intercept(Object proxyObj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("CglibProxyService: method intercept log ....");
        return methodProxy.invoke(target, args);
    }

}
```

3、测试类
``` java
public class CglibProxyMain {

    public static void main(String[] args) {
        CglibProxyService proxyService = new CglibProxyService(new RealService());
        RealService proxy = (RealService) proxyService.getProxy();
        System.out.println(proxy);
        proxy.findUser();

        System.out.println();
        proxy.saveUser();  // 这个 save 方法会被aop拦截
    }

}
```

4、运行结果

![jdk 动态代理测试运行结果](./images/000015.png)


### cglib 与 jdk 动态代理的区别
- jdk 动态代理只能够对接口进行代理，不能对普通的类进行代理（因为所有生成的代理类的父类为 Proxy，jdk 类继承机制不允许多重继承）；CGLIB 能够代理普通类；
- jdk 动态代理使用 Java 原生的反射 API 进行操作，在生成类上比较高效；CGLIB 使用 ASM 框架直接对字节码进行操作，在类的执行过程中比较高效
- cglib 不能对 final 方法进行代理

cglib 动态代理，参考：  
http://blog.csdn.net/danchu/article/details/70238002

Java动态代理机制详解（JDK 和CGLIB，Javassist，ASM）
http://blog.csdn.net/luanlouis/article/details/24589193

### asm 动态生成字节码

示例：使用 asm 动态生成 class 文件
1、添加 asm 依赖 jar
``` xml
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm</artifactId>
    <version>5.2</version>
</dependency>
```

2、实现
``` java
public class MyGenerator {

    public static void main(String[] args) throws IOException {

        System.out.println();
        ClassWriter classWriter = new ClassWriter(0);
        // 通过visit方法确定类的头部信息
        classWriter.visit(Opcodes.V1_8,// java版本
                Opcodes.ACC_PUBLIC,// 类修饰符
                "Programmer", // 类的全限定名
                null, "java/lang/Object", null);

        //创建构造函数
        MethodVisitor mv = classWriter.visitMethod(Opcodes.ACC_PUBLIC, "<init>", "()V", null, null);
        mv.visitCode();
        mv.visitVarInsn(Opcodes.ALOAD, 0);
        mv.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/Object", "<init>","()V", false);
        mv.visitInsn(Opcodes.RETURN);
        mv.visitMaxs(1, 1);
        mv.visitEnd();

        // 定义code方法
        MethodVisitor methodVisitor = classWriter.visitMethod(Opcodes.ACC_PUBLIC, "code", "()V",
                null, null);
        methodVisitor.visitCode();
        methodVisitor.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out",
                "Ljava/io/PrintStream;");
        methodVisitor.visitLdcInsn("I'm a Programmer,Just Coding.....");
        methodVisitor.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println",
                "(Ljava/lang/String;)V", false);
        methodVisitor.visitInsn(Opcodes.RETURN);
        methodVisitor.visitMaxs(2, 2);
        methodVisitor.visitEnd();
        classWriter.visitEnd();

        // 使classWriter类已经完成
        // 将classWriter转换成字节数组写到文件里面去
        byte[] data = classWriter.toByteArray();
        File file = new File("D://Programmer.class");
        FileOutputStream fout = new FileOutputStream(file);
        fout.write(data);
        fout.close();
    }

}
```

3、class 文件反编译后
``` java
public class Programmer {
    public Programmer() {
    }

    public void code() {
        System.out.println("I'm a Programmer,Just Coding.....");
    }
}
```
