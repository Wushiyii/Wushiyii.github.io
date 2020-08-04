---
title: Dubbo SPI
date: 2020-08-01 12:23:34
tags: [distribute、framework、dubbo]
categories: framework
---
SPI全称为`Service Provider Interface`,即服务发现。类似于Spring的IOC一样，SPI注入的类可以被加载到runtime中。

<!-- more -->

SPI 的本质是将接口实现类的全限定名配置在文件中（文件名称即是接口的全限定名），并由服务加载器读取配置文件，加载实现类。这样可以在运行时，动态为接口替换实现类。

### SPI简易用法与原理

首先定义一个需要实现SPI的接口，比如这边定一个`Animal`接口作为规范。

```java
public interface Animal {
    void sayHello();
}
```

再增加几个实现类

```java
public class Monkey implements Animal {
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Monkey.");
    }
}

public class Snake implements Animal {
    @Override
    public void sayHello() {
        System.out.println("Hello, I am Snake.");
    }
}
```

在`resource`文件夹下建立`META-INF/services`文件夹（为什么是这个文件夹？原理部分有说明），放入接口全限定名为名称的文件，并在文件里写上实现类的全限定名，文件夹结构类似这样

```properties
java
	-com
		-wushiyii
			-messy
				-grammer
					-spi
						-Animal.class
						-Monkey.class
						-Snake.class
resource
	- META-INF
		- services
			com.wushiyii.messy.grammar.spi.Animal
```

文件`com.wushiyii.messy.grammar.spi.Animal`里的内容是

```properties
com.wushiyii.messy.grammar.spi.Monkey
com.wushiyii.messy.grammar.spi.Snake
```

另外在写一个SPI Test 来看结果

```java
public class SpiTest {
    @Test
    public void sayHello() {
        ServiceLoader<Animal> serviceLoader = ServiceLoader.load(Animal.class);
        serviceLoader.forEach(Animal::sayHello);
    }
}

// Out:
// Hello, I am Monkey.
// Hello, I am Snake.
```

`SPI`的原理是什么？在SPI test中，Animal类是通过ServiceLoader来进行加载发现，点开ServiceLoader看看

```java
public final class ServiceLoader<S>
    implements Iterable<S>
{
		
    private static final String PREFIX = "META-INF/services/";

    // 需要加载的interface
    private final Class<S> service;

    // 加载SPi的类加载器
    private final ClassLoader loader;

    // 上下文
    private final AccessControlContext acc;

    // 实现SPI的类实例
    private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

    // 当前线程的懒加载迭代器
    private LazyIterator lookupIterator;
  
	  // ...
}
```

ServiceLoader已经固定写死了，只能从`"META-INF/services"`文件夹下读取SPI实现类。再来看一下整个SPI的运行加载原理，首先从`ServiceLoader.load(Animal.class)`开始。

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
			  //获取当前线程的类加载器
        ClassLoader cl = Thread.currentThread().getContextClassLoader();
        return ServiceLoader.load(service, cl);
}

public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader){
  return new ServiceLoader<>(service, loader);
}

private ServiceLoader(Class<S> svc, ClassLoader cl) {
  		  // service对象在此被赋值，即要实现的interface
        service = Objects.requireNonNull(svc, "Service interface cannot be null");
  			// 判断当前classLoader是否为空
        loader = (cl == null) ? ClassLoader.getSystemClassLoader() : cl;
  			// 获取AccessController的当前上下文快照
        acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
  			//载入
        reload();
}

// reload方法的意图很简单，生成一个懒加载机制的迭代器，等到真正去用的时候才会去`META-INF/services`下寻找并实例化实现类
public void reload() {
        providers.clear();
        lookupIterator = new LazyIterator(service, loader);
}

// 在使用的时候，会进行实例化等操作
public boolean hasNext() {
            if (acc == null) {
                return hasNextService();
            } else {
                PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                    public Boolean run() { return hasNextService(); }
                };
                return AccessController.doPrivileged(action, acc);
            }
}

private boolean hasNextService() {
  					// 记录下一个实例类名
            if (nextName != null) {
                return true;
            }
  
            if (configs == null) {
                try {
                  	// 组装路径名+全限定接口名
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
  					// 判断刚刚读取到的`Enumeration<URL> configs`是否存在下一个元素，是的话就解析
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
}


private Iterator<String> parse(Class<?> service, URL u)
        throws ServiceConfigurationError
    {
        InputStream in = null;
        BufferedReader r = null;
        ArrayList<String> names = new ArrayList<>();
        try {
          	// 打开输入流
            in = u.openStream();
          	// 转为Reader读取
            r = new BufferedReader(new InputStreamReader(in, "utf-8"));
            int lc = 1;
            // 读取解析是通过`parseLine`来实现的，每次读取一行，并放置在names中
            while ((lc = parseLine(service, u, r, lc, names)) >= 0);
        } catch (IOException x) {
            fail(service, "Error reading configuration file", x);
        } finally {
            try {
                if (r != null) r.close();
                if (in != null) in.close();
            } catch (IOException y) {
                fail(service, "Error closing configuration file", y);
            }
        }
        return names.iterator();
    }
```

拿到服务是通过iterator的next方法进行读取的，而next方法会执行nextService方法，实例化就是在nextService中发生的。

```java
private S nextService() {
    if (!hasNextService())
        throw new NoSuchElementException();
    // hasNext中传递下来的nextName
    String cn = nextName;
    nextName = null;
    Class<?> c = null;
    try {
      	// 尝试加载读取到的类
        c = Class.forName(cn, false, loader);
    } catch (ClassNotFoundException x) {
        fail(service,
             "Provider " + cn + " not found");
    }
    if (!service.isAssignableFrom(c)) {
        fail(service,
             "Provider " + cn  + " not a subtype");
    }
    try {
      	// 实例化
        S p = service.cast(c.newInstance());
      	// 将实例化的类放到之前的linkedHashMap中
        providers.put(cn, p);
        return p;
    } catch (Throwable x) {
        fail(service,
             "Provider " + cn + " could not be instantiated",
             x);
    }
    throw new Error();          // This cannot happen
}
```

至此，SPI的加载与实例化都完成了，所以在SPI test中，可以直接调用`sayHello()`方法。

通过SPI可以使第三方服务模块的逻辑与业务代码相分离，而不耦合在一起，应用程序可以根据实际业务进行扩展。但是SPI也存在缺点：

- 做不到按需加载，如果要拿到自己的实现，最坏情况需要一直执行next，遍历完整个iterator才能找到需要的实现类
- 不能提供类似IOC的功能
- 扩展很难和其他的框架集成，比如扩展里面依赖了一个Spring bean，原生的Java SPI不支持



### Dubbo里的SPI

Dubbo 并未使用 Java SPI，而是重新实现了一套功能更强的 SPI 机制，Dubbo实现SPI的逻辑封装在ExtensionLoader类中，Dubbo SPi的文件放在`META-INF/dubbo`下，配置文件例如

```properties
monkey=com.alibaba.dubbo.common.extensionloader.Monkey
snake=com.alibaba.dubbo.common.extensionloader.Snake
```

配置文件是以键值进行配置的，如下面的test案例，可以实现按需加载了。

```java
@Test
public void sayHello() throws Exception {
    ExtensionLoader<Animal> extensionLoader =
            ExtensionLoader.getExtensionLoader(Animal.class);
    Animal monkey = extensionLoader.getExtension("monkey");
    monkey.sayHello();
    Animal snake = extensionLoader.getExtension("snake");
    snake.sayHello();
}
```

#### Dubbo SPI 源码分析

首先先看ExtensionLoader的对象参数，可以发现Dubbo读取的SPI扩展类是在META-INF/dubbo下读取

```java
public class ExtensionLoader<T> {

    private static final Logger logger = LoggerFactory.getLogger(ExtensionLoader.class);

    private static final String SERVICES_DIRECTORY = "META-INF/services/";

    private static final String DUBBO_DIRECTORY = "META-INF/dubbo/";

    private static final String DUBBO_INTERNAL_DIRECTORY = DUBBO_DIRECTORY + "internal/";

    private static final Pattern NAME_SEPARATOR = Pattern.compile("\\s*[,]+\\s*");

    private static final ConcurrentMap<Class<?>, ExtensionLoader<?>> EXTENSION_LOADERS = new ConcurrentHashMap<Class<?>, ExtensionLoader<?>>();

    private static final ConcurrentMap<Class<?>, Object> EXTENSION_INSTANCES = new ConcurrentHashMap<Class<?>, Object>();
 
 		/...
}
```

getExtensionLoader 方法用于从缓存中获取与拓展类对应的 ExtensionLoader，若缓存未命中，则创建一个新的实例

```java
@SuppressWarnings("unchecked")
public static <T> ExtensionLoader<T> getExtensionLoader(Class<T> type) {
    if (type == null)
        throw new IllegalArgumentException("Extension type == null");
  	// class type必须为接口
    if (!type.isInterface()) {
        throw new IllegalArgumentException("Extension type(" + type + ") is not interface!");
    }
    // 接口必须有@SPI注解
    if (!withExtensionAnnotation(type)) {
        throw new IllegalArgumentException("Extension type(" + type +
                ") is not extension, because WITHOUT @" + SPI.class.getSimpleName() + " Annotation!");
    }
		// 从缓存中获取ExtensionLoader，如未获取到则创建一个
    ExtensionLoader<T> loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    if (loader == null) {
        EXTENSION_LOADERS.putIfAbsent(type, new ExtensionLoader<T>(type));
        loader = (ExtensionLoader<T>) EXTENSION_LOADERS.get(type);
    }
    return loader;
}
```

getExtension是获取扩展类的核心方法

```java
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
      // 获取默认的拓展实现类
        return getDefaultExtension();
    }
   //Holder的作用是持有一个对象，内部存放一个volatile的value值
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
  	// 双检锁，为空则新建一个扩展，并放入holder中
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //新建扩展
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

创建扩展类

```java
private T createExtension(String name) {
    //加载所有扩展类
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        injectExtension(instance);
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        return instance;
    } catch (Throwable t) {
        throw new IllegalStateException("Extension instance(name: " + name + ", class: " +
                type + ")  could not be instantiated: " + t.getMessage(), t);
    }
}
```

createExtension首先会调用getExtensionClasses来加载所有扩展类；loadExtensionClasses 方法总共做了两件事情，一是对 SPI 注解进行解析，二是调用 loadDirectory 方法加载指定文件夹配置文件。

```java
private Map<String, Class<?>> getExtensionClasses() {
    Map<String, Class<?>> classes = cachedClasses.get();
    if (classes == null) {
        synchronized (cachedClasses) {
            classes = cachedClasses.get();
            if (classes == null) {
              	// 加载扩展类
                classes = loadExtensionClasses();
                cachedClasses.set(classes);
            }
        }
    }
    return classes;
}

private Map<String, Class<?>> loadExtensionClasses() {
        final SPI defaultAnnotation = type.getAnnotation(SPI.class);
  			// 从SPI注解里获取需要默认加载的key
        if (defaultAnnotation != null) {
            String value = defaultAnnotation.value();
            if ((value = value.trim()).length() > 0) {
                String[] names = NAME_SEPARATOR.split(value);
                if (names.length > 1) {
                    throw new IllegalStateException("more than 1 default extension name on extension " + type.getName() + ": " + Arrays.toString(names));
                }
                if (names.length == 1) cachedDefaultName = names[0];
            }
        }
				
        Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
  			//加载`META-INF/services/`、`META-INF/dubbo/`、`META-INF/dubbo/internal`下所有SPI实现类
        loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
        loadDirectory(extensionClasses, DUBBO_DIRECTORY);
        loadDirectory(extensionClasses, SERVICES_DIRECTORY);
        return extensionClasses;
    }
```

loadDirectory 方法先通过 classLoader 获取所有资源链接，然后再通过 loadResource 方法加载资源。

```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
    String fileName = dir + type.getName();
    try {
        Enumeration<java.net.URL> urls;
        ClassLoader classLoader = findClassLoader();
      	// 使用classloader获取资源URL
        if (classLoader != null) {
            urls = classLoader.getResources(fileName);
        } else {
            urls = ClassLoader.getSystemResources(fileName);
        }
        if (urls != null) {
            while (urls.hasMoreElements()) {
                java.net.URL resourceURL = urls.nextElement();
              	//根据资源URL解析class
                loadResource(extensionClasses, classLoader, resourceURL);
            }
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class(interface: " +
                type + ", description file: " + fileName + ").", t);
    }
}
```

loadResource主要用于读取配置文件并解析，加载实现类，并通过loadClass进行校验与缓存

```java
private void loadResource(Map<String, Class<?>> extensionClasses, ClassLoader classLoader, java.net.URL resourceURL) {
    try {
        BufferedReader reader = new BufferedReader(new InputStreamReader(resourceURL.openStream(), "utf-8"));
        try {
            String line;
          // 读取一行
            while ((line = reader.readLine()) != null) {
              // # 号分割，截取 # 之前的字符串，# 之后的内容为注释，需要忽略
                final int ci = line.indexOf('#');
                if (ci >= 0) line = line.substring(0, ci);
                line = line.trim();
                if (line.length() > 0) {
                    try {
                        String name = null;
                      	// =号分割，截取key、value
                        int i = line.indexOf('=');
                        if (i > 0) {
                            name = line.substring(0, i).trim();
                            line = line.substring(i + 1).trim();
                        }
                        if (line.length() > 0) {
                          	// 装载SPI实现类，并通过 loadClass 方法对类进行缓存
                            loadClass(extensionClasses, resourceURL, Class.forName(line, true, classLoader), name);
                        }
                    } catch (Throwable t) {
                        IllegalStateException e = new IllegalStateException("Failed to load extension class(interface: " + type + ", class line: " + line + ") in " + resourceURL + ", cause: " + t.getMessage(), t);
                        exceptions.put(line, e);
                    }
                }
            }
        } finally {
            reader.close();
        }
    } catch (Throwable t) {
        logger.error("Exception when load extension class(interface: " +
                type + ", class file: " + resourceURL + ") in " + resourceURL, t);
    }
}
```



```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
  	// 判断加载的类是否为接口的实现类
    if (!type.isAssignableFrom(clazz)) {
        throw new IllegalStateException("Error when load extension class(interface: " +
                type + ", class line: " + clazz.getName() + "), class "
                + clazz.getName() + "is not subtype of interface.");
    }
    //判断实现类时否含有@Adaptive注解
    if (clazz.isAnnotationPresent(Adaptive.class)) {
        if (cachedAdaptiveClass == null) {
          // 设置 cachedAdaptiveClass缓存
            cachedAdaptiveClass = clazz;
        } else if (!cachedAdaptiveClass.equals(clazz)) {
            throw new IllegalStateException("More than 1 adaptive class found: "
                    + cachedAdaptiveClass.getClass().getName()
                    + ", " + clazz.getClass().getName());
        }
      // 判断加载类是否是Wrapper类
    } else if (isWrapperClass(clazz)) {
        Set<Class<?>> wrappers = cachedWrapperClasses;
        if (wrappers == null) {
            cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
            wrappers = cachedWrapperClasses;
        }
        wrappers.add(clazz);
      // 加载类为普通扩展类
    } else {
      // 检测 clazz 是否有默认的构造方法，如果没有，则抛出异常
        clazz.getConstructor();
        if (name == null || name.length() == 0) {
          // 如果 name 为空，则尝试从 Extension 注解中获取 name，或使用小写的类名作为name
            name = findAnnotationName(clazz);
            if (name.length() == 0) {
                throw new IllegalStateException("No such extension name for the class " + clazz.getName() + " in the config " + resourceURL);
            }
        }
      	//切分配置
        String[] names = NAME_SEPARATOR.split(name);
        if (names != null && names.length > 0) {
            Activate activate = clazz.getAnnotation(Activate.class);
            if (activate != null) {
              // 如果类上有 Activate 注解，则使用 names 数组的第一个元素作为键，
		          // 存储 name 到 Activate 注解对象的映射关系
                cachedActivates.put(names[0], activate);
            }
            for (String n : names) {
                if (!cachedNames.containsKey(clazz)) {
                  // 存储 Class 到名称的映射关系
                    cachedNames.put(clazz, n);
                }
                Class<?> c = extensionClasses.get(n);
                if (c == null) {
                  // 存储名称到 Class 的映射关系
                    extensionClasses.put(n, clazz);
                } else if (c != clazz) {
                    throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                }
            }
        }
    }
}
```

loadCLass方法也不复杂，主要是设置cachedAdaptiveClass、cachedActivates、cachedNames（Class与名称映射关系） 这三个缓存，以上几步就完成了从META-INF中装载dubbo扩展类到runtime中。