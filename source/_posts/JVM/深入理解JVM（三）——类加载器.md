---
title: 深入理解JVM（三） --- 类加载器
date: 2019-09-01 13:10:34
tags: [JAVA,JVM]
categories: JAVA

---

Defenite a classloader

<!-- more -->

# 深入理解JVM（三）—类加载器



#### 不同类加载器加载的类，并不相同

每一个类加载器，都拥有一个独立的类名称空间，所以不同的类加载器即使加载同一个类，这两个类也并不相同（equals、isAssignableFrom、isIntance）

```java
public class MyClassLoader {

    public static void main(String[] args) throws Exception {
        ClassLoader myClassLoader = new ClassLoader() {
            @Override
            public Class<?> loadClass(String name) throws ClassNotFoundException {
                try {
                    String fileName = name.substring(name.lastIndexOf(".") + 1) + ".class";
                    InputStream is = getClass().getResourceAsStream(fileName);
                    if (is == null) {
                        return super.loadClass(name);
                    }
                    byte[] b = new byte[is.available()];
                    is.read(b);
                    return defineClass(name,b,0,b.length);
                }catch (IOException e) {
                    throw new ClassNotFoundException(name);
                }
            }
        };
        Object object = myClassLoader.loadClass("com.wushiyii.jvm.demo.classLoader.MyClassLoader").newInstance();
        System.out.println(object.getClass());
        System.out.println(object instanceof com.wushiyii.jvm.demo.classLoader.MyClassLoader);
    }
}
```



#### 双亲委派模型

在Java虚拟机中，只有两种类加载器：

- 启动类加载器：使用C++实现，属于虚拟机的一部分
- 其他类加载器：由JAVA实现，独立于虚拟机外部，并且全部继承自`java.lang.ClassLoader`



从Java角度看，类加载器实际上可以划分的更细致一些，绝大部分Java程序都会使用到以下3种系统提供的类加载器。

- 启动类加载器（`Bootstrap ClassLoader`）:负责将<JAVA_HOME>/lib 、-Xbootclasspath且能被虚拟机识别的类库加载到虚拟机内存中。启动类加载器无法被Java程序直接调用，编写自定义加载器时，直接将加载请求委派给引导类加载器，即使用null即可。

- 扩展类加载器（`Extension ClassLoader`）:这个加载器由`sun.misc.Launcher$ExtClassLoader`实现，负责加载<JAVA_HOME>/lib/ext活java.ext.dirs系统变量指定的路径中的所有类库。
- 应用程序类加载器（`Application ClassLoader`）:由`sum.misc.Launch$AppClassLoader`实现，也称为系统类加载器。它负责加载用户类路径（ClassPath）上指定的类库，一般如果程序没有自定义过类加载器，默认就是使用系统类加载器。

#### 类加载器关系

![类加载器关系](/Users/qudian/Desktop/类加载器关系.png)



图中各个类加载器之间的关系，称为类加载器的双亲委派模型（`Parents Delegation Model`）。双亲委派模型要求除了顶层的启动类加载器，其余的类加载器都应当由自己的父类加载器。并且类加载器之间的父子关系不会以继承（`Inheritance`）的关系实现，而是使用组合（`Composition`）关系来复用父类加载器代码。

双亲委派模型的**工作过程**是:如果一个类加载器收到了类加载的请求，它首先不会加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都会传送到顶层的启动类加载器中，只有父类加载器反馈无法加载时，子加载器才会尝试加载。

使用双亲委派模型有一个显而易见的好处就是Java类随着类加载器一起具备类一种带优先级的层次关系，比如一个Object类，它存放在rt.jar中，无论在哪一个类加载器中加载，最终都会到启动类加载器中进行加载，因此Object在程序中的各个环境里都是同一个类。

以下为双亲委派模型的实现：

```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 先检查类是否已经被加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    //如果父类加载器无法加载类，则会抛出ClassNotFoundException异常
                }

                if (c == null) {
                    // 父类加载器无法加载时，会调用本身的findClass方法进行类加载
                    long t1 = System.nanoTime();
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }
```

