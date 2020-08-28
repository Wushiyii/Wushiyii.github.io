---
title: Java IO
date: 2019-10-11 11:23:54
tags: [distribute、framework、dubbo]
categories: framework
---
<!-- more -->

### IO分类

1、文件（file）：FileInputStream、FileOutputStream、FileReader、FileWriter

2、数组（[]）：

- 2.1、字节数组（byte[]）：ByteArrayInputStream、ByteArrayOutputStream
- 2.2、字符数组（char[]）：CharArrayReader、CharArrayWriter

3、管道操作：PipedInputStream、PipedOutputStream、PipedReader、PipedWriter

4、基本数据类型：DataInputStream、DataOutputStream

5、缓冲操作：BufferedInputStream、BufferedOutputStream、BufferedReader、BufferedWriter

6、打印：PrintStream、PrintWriter

7、对象序列化反序列化：ObjectInputStream、ObjectOutputStream

8、转换：InputStreamReader、OutputStreWriter

9、字符串（String）**Java8中已废弃**：~~StringBufferInputStream、StringBufferOutputStream、StringReader、StringWriter~~

[![adMgc6.png](https://s1.ax1x.com/2020/08/03/adMgc6.png)](https://imgchr.com/i/adMgc6)

### 字节流与字符流

- IO基本单位不同
  - 字节流以byte为单位
  - 字符流以char为单位（字符根据编码不同，字节大小也不同；比如中文编码是2字节，UTF-8是3字节）
- 用处不同
  - 字节流用于处理二进制数据（图像、文件、传输报文...）
  - 字符流用于处理编码文件、中文等

### 4个基本抽象类

Java IO的结构虽然看着复杂，不过都是从4个基本抽象类中衍生的，这四个分别是：`InputStream`、`OutputStream`、`Reader`、`Writer`。而这四个抽象类的方法中常用的就是`read`与`write`。



### 基本用法

获取一个InputStream

```java
@SneakyThrows
private static InputStream getInputStream() {
  		return new URL("http://www.baidu.com").openStream();
}
```



- InputStream

两种读取

```java
		// 单个bit 读取
    @SneakyThrows
    public static void bitRead() {
        // init a stream
        InputStream in = getInputStream();
        StringBuilder res = new StringBuilder();
        int temp;
        while ((temp = in.read()) != -1) {
            res.append((char)temp);
        }
        System.out.println(res.toString());
    }

    // 数组读取
    @SneakyThrows
    public static void arrayRead() {
        // init a stream
        InputStream in = getInputStream();
        StringBuilder res = new StringBuilder();

        byte[] b = new byte[1024];
        int len;
        while ((len = (in.read(b))) != -1) {
            res.append(new String(b, 0, len));
        }
        System.out.println(res.toString());
    }
```

- InputStreamReader

和InputStream没什么不同，区别在于数组读取的时候，用的是`char`类型数组，基本单位是字符

```java
@SneakyThrows
public static void bitRead() {
    InputStreamReader inputStreamReader = new InputStreamReader(getInputStream());
    StringBuilder res = new StringBuilder();
    int temp;
    while ((temp = inputStreamReader.read()) != -1) {
        res.append((char)temp);
    }
    System.out.println(res.toString());
}

@SneakyThrows
public static void arrayRead() {
    InputStreamReader inputStreamReader = new InputStreamReader(getInputStream());
    StringBuilder res = new StringBuilder();
    char[] buffer = new char[1024];
    int len;
    while ((len = inputStreamReader.read(buffer)) != -1) {
        res.append(new String(buffer, 0, len));
    }
    System.out.println(res.toString());
}
```

- OutputStream

```java
@SneakyThrows
public static void out() {
    InputStream is = getInputStream();
    OutputStream os = new FileOutputStream(new File("/Users/xxx/Desktop/a.txt"));

    byte[] buf = new byte[1024];
    int len;
    while ((len = is.read(buf)) != -1) {
        os.write(buf, 0, len);
    }
    is.close();
    
    os.flush();
    os.close();
}
```

- OutputReader

```java
@SneakyThrows
public static void out() {
    InputStreamReader is = new InputStreamReader(getInputStream());;
    OutputStreamWriter osw = new FileWriter("/Users/qudian/Desktop/a.txt");

    char[] buffer = new char[1024];
    int len;
    while ((len = is.read(buffer)) != -1) {
        osw.write(new String(buffer, 0, len));
    }
    is.close();
    osw.close();
}
```



### 装饰者模式在IO中的应用

装饰者`Decorator`有替代增强原始类的功能，在IO中就有体现。比如使用BufferedInputStream

```java
@SneakyThrows
private static void bufferedRead() {
    BufferedInputStream bis = new BufferedInputStream(getInputStream());
    byte[] buf = new byte[1024];
    int len;
    StringBuilder res = new StringBuilder();
    while ((len = bis.read(buf)) != -1) {
        res.append(new String(buf, 0, len));
    }
    System.out.println(res.toString());
}
```

观察BufferedInputStream的构造函数，实际上BufferInputStream调用了父类FilterInputStream的构造函数

```java
public BufferedInputStream(InputStream in, int size) {
    super(in);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

在真正调用的时候，调用read方法实际上就会调用到FilterInputStream里的read方法。