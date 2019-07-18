---
title: 能不能自己写个类叫 java.lang.System？
categories:
  - 技术
tags:
  - JVM
abbrlink: 814dc607
date: 2019-07-18 14:54:53
---
这是一个很经典的问题，但是网上提供的答案容易误导别人。接下来通过详细说明和实例来看看实际情况是怎么样的。

# 错误答案的详细解释
为了不让我们写 System 类，类加载采用委托机制，这样可以保证 parent 优先，parent 能找到的类，儿子就没有机会加载。而 System 类是 Bootstrap 加载器加载的，就算自己重写，也总是使用 Java 系统提供的 System，自己写的 System 类根本没有机会得到加载。但是，我们可以自己定义一个类加载器来达到这个目的，为了避免双亲委托机制，这个类加载器也必须是特殊的。由于系统自带的三个类加载器都加载特定目录下的类，如果我们自己的类加载器放在一个特殊的目录，那么系统的加载器就无法加载，也就是最终还是由我们自己的加载器加载。

说明一下上面提到的一些概念：

类加载器可分为两类：一是启动类加载器(Bootstrap ClassLoader)，是 C++ 实现的，是 JVM 的一部分；另一种是其它的类加载器，是 Java 实现的，独立于 JVM，全部都继承自抽象类 java.lang.ClassLoader。jdk 自带了三种类加载器，分别是启动类加载器（Bootstrap ClassLoader），扩展类加载器（Extension ClassLoader），应用程序类加载器（Application ClassLoader）。后两种加载器是继承自抽象类 java.lang.ClassLoader。

# 类加载器使用的原理
类加载器的加载过程一般是：自定义类加载器 >> 应用程序类加载器 >> 扩展类加载器 >> 启动类加载器。

上面的层次关系被称为双亲委派模型(Parents Delegation Model)。除了最顶层的启动类加载器外，其余的类加载器都有对应的父类加载器。

再简单说下双亲委托机制：如果一个类加载器收到了类加载的请求，它首先不会自己尝试去加载这个类，而是把这个请求委派给父类加载器，每一个层次的类加载器都是如此，因此所有的加载请求最终到达顶层的启动类加载器，只有当父类加载器反馈自己无法完成加载请求时（指它的搜索范围没有找到所需的类），子类加载器才会尝试自己去加载。

再回去看下解释内容，我相信前面的部分大家应该很看懂了，也没什么大问题。最后的如果部分“如果我们自己的类加载器放在一个特殊的目录，那么系统的加载器就无法加载，也就是最终还是由我们自己的加载器加载。” ，逻辑完全不通。我想它的本意可能是，将自己的 java.lang.System 类放置在特殊目录，然后系统自带的加载器无法加载，这样最终还是由我们自己的加载器加载（因为我们自己的加载器知道其所在的特殊目录）。这种说法好像逻辑上没有问题，那么我们就来实验下了。

# 代码验证
```
// 自定义类加载器
public class MyClassLoader extends ClassLoader{

    public MyClassLoader() {
        super(null);
    }

    @Override
    public Class<?> loadClass(String name) throws ClassNotFoundException {
        try{
            String className = null;
            if(name.startsWith("java.lang")){
                className = "/" + name.replace('.', '/') + ".class";
            }else{
                className = name.substring(name.lastIndexOf('.') + 1) + ".class";
            }
            System.out.println(className);
            InputStream is = getClass().getResourceAsStream(className);
            System.out.println(is);
            if(is == null)
                return super.loadClass(name);

            byte[] b = new byte[is.available()];
            is.read(b);
            return defineClass(name, b, 0, b.length);
        }catch (Exception e) {
            e.printStackTrace();
            throw new ClassNotFoundException();
        }
    }
}
```

Math 类：
```
import java.lang


public final class Math {
    
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}


public class MyMath {
    
    public static void main(String[] args) {
        System.out.println("hello world");
    }
}
```

ClassLoader 测试类：
```
public class ClassLoaderTest {
 
    public static void main(String[] args) throws ClassNotFoundException, InstantiationException, IllegalAccessException {
        ClassLoader myLoader = new MyClassLoader();
        Object obj = myLoader.loadClass("java.lang.Math").newInstance();
        System.out.println(obj);
    }
    
}
```

上面的测试代码没用自定义java.lang.System类，因为测试代码用到了JDK自带的System类进行输出打印，会冲突，所以改用为自定义的java.lang.Math类。如果自定义的Math类能加载，那么自定义的System类同样能加载。

我们先直接运行下 Math 类，输出如下：
```
错误: 在类 java.lang.Math 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application
```

提示Math类没有 main 方法。首先大家要明白一个概念，当类首次主动使用时，必须进行类的加载，这部分工作是由类加载器来完成的。根据双亲委托原则，Math 类首先由启动类加载器去尝试加载，很显然，它找到 rt.jar 中的 java.lang.Math 类并加载进内存（并不会加载我们自定义的Math类），然后执行 main 方法时，发现不存在该方法，所以报方法不存在错误。也就是说，默认情况下 JVM 不会加载我们自定义的 Math 类。

再直接运行MyMath类，输出如下：

```
java.lang.SecurityException: Prohibited package name: java.lang
at java.lang.ClassLoader.preDefineClass(ClassLoader.java:479)
at java.lang.ClassLoader.defineClassCond(ClassLoader.java:625)
at java.lang.ClassLoader.defineClass(ClassLoader.java:615)
at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:141)
at java.net.URLClassLoader.defineClass(URLClassLoader.java:283)
at java.net.URLClassLoader.access$000(URLClassLoader.java:58)
at java.net.URLClassLoader$1.run(URLClassLoader.java:197)
at java.security.AccessController.doPrivileged(Native Method)
at java.net.URLClassLoader.findClass(URLClassLoader.java:190)
at java.lang.ClassLoader.loadClass(ClassLoader.java:306)
at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:301)
at java.lang.ClassLoader.loadClass(ClassLoader.java:247)
Exception in thread "main" 
```

由堆栈异常信息可知道，当应用程序类加载器类（AppClassLoader）尝试加载MyMath类时，ClassLoader.java的479行抛出了SecurityException。

**禁止使用包名：java.lang。**

直接查看抽象类 java.lang.ClassLoader 的 preDefineClass 方法代码，摘抄如下：

```java
private ProtectionDomain preDefineClass(String name,
                                            ProtectionDomain pd)
{
    if (!checkName(name))
        throw new NoClassDefFoundError("IllegalName: " + name);

    // Note:  Checking logic in java.lang.invoke.MemberName.checkForTypeAlias
    // relies on the fact that spoofing is impossible if a class has a name
    // of the form "java.*"
    if ((name != null) && name.startsWith("java.")) {
        throw new SecurityException
            ("Prohibited package name: " +
             name.substring(0, name.lastIndexOf('.')));
    }
    if (pd == null) {
        pd = defaultDomain;
    }

    if (name != null) checkCerts(name, pd.getCodeSource());

    return pd;
}
```

可以看到如果加载的类全名称以“java.”开头时，将会抛出SecurityException，这也是为什么直接执行MyMath类会出现 SecurityException。

照这样，我们自定义的类加载器必须继承自ClassLoader，其loadClass()方法里调用了父类的defineClass()方法，并最终调到 preDefineClass() 方法，因此我们自定义的类加载器也是不能加载以 “java.” 开头的 java 类的。我们继续运行下 ClassLoaderTest 类，输出如下：

```

/java/lang/Math.class
sun.net.www.protocol.jar.JarURLConnection$JarURLInputStream@a981ca
java.lang.SecurityException: Prohibited package name: java.lang
at java.lang.ClassLoader.preDefineClass(ClassLoader.java:479)
at java.lang.ClassLoader.defineClassCond(ClassLoader.java:625)
at java.lang.ClassLoader.defineClass(ClassLoader.java:615)
at java.lang.ClassLoader.defineClass(ClassLoader.java:465)
at com.tq.MyClassLoader.loadClass(MyClassLoader.java:28)
at com.tq.ClassLoaderTest.main(ClassLoaderTest.java:8)
Exception in thread "main" java.lang.ClassNotFoundException
at com.tq.MyClassLoader.loadClass(MyClassLoader.java:31)
at com.tq.ClassLoaderTest.main(ClassLoaderTest.java:8)
```

通过代码实例及源码分析可以看到，对于自定义的类加载器，强行用defineClass()方法去加载一个以"java."开头的类也是会抛出异常的。

# 总结
**不能自己写以"java."开头的类，其要么不能加载进内存，要么即使你用自定义的类加载器去强行加载，也会收到一个SecurityException。**

# 参考
* [java能不能自己写一个类叫java.lang.System/String正确答案](https://blog.csdn.net/tang9140/article/details/42738433)