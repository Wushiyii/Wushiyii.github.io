---
title: 从0开始实现一个简易MVC框架
date: 2022-02-09 12:10:00
tags: [MVC]
categories: java
---
从0开始实现一个简易MVC框架

<!-- more -->

## 基本概念

1. 框架基于`Servlet`实现，所有的请求都需要走代码中指定的`Servlet`，由一个`Servlet`做初始化与分发请求
2. 内嵌`Tomcat`
3. 需要一些反射的知识
4. 下方的代码只是把相对重要的部分展示，有很多工具类或者不太重要的地方请在Git查看（https://github.com/Wushiyii/MyMVC）



## 开始实现

### Pom

先导包，内嵌的`Tomcat`实际上已经把需要的`Servlet`相关的包导入了（主要的包就是`Tomcat`，其他都是些`Guava`、`Lombok`之类的）

```xml
 <dependency>
      <groupId>org.apache.tomcat.embed</groupId>
      <artifactId>tomcat-embed-jasper</artifactId>
      <version>8.5.31</version>
 </dependency>
```

### 实现请求入口

> 继承`HttpServlet`即可拦截所有的请求（Tomcat部分代码会进行mapping配置）

```java
@Slf4j
public class MyMVCServlet extends HttpServlet {

    //初始化
    @Override
    public void init(ServletConfig config) throws ServletException {

        //基本包路径
        String basePackage = MyMVCConfiguration.getBasePackage();
        log.info("MyMVC init basePackage:" + basePackage);

        //扫描注解
        Set<Class<?>> allPackageClass = ClassUtil.getPackageClass(basePackage);
        for (Class<?> clazz : allPackageClass) {

            for (Method declaredMethod : clazz.getDeclaredMethods()) {

                if (declaredMethod.isAnnotationPresent(GET.class) || declaredMethod.isAnnotationPresent(POST.class)) {

                    //注册执行器端点
                    EndpointManager.register(clazz, declaredMethod);
                }
            }
        }
    }

    //执行请求（所有的请求都会走这边）
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        RequestHandlerTemplate.handle(req, resp);
    }
}
```

EndPoint执行器注册（主要就是拿一个`Table`存路由信息）

```java
@Slf4j
public class EndpointManager {

    private final static Table<String, String, EndpointMetaInfo> endpointTable = HashBasedTable.create();

    // 不用考虑并发场景，只有启动时for循环注册一次
    @SneakyThrows
    public static void register(Class<?> clazz, Method declaredMethod) {
        String[] multiPath = {};
        String method = "";
        if (declaredMethod.isAnnotationPresent(GET.class)) {
            multiPath = declaredMethod.getAnnotation(GET.class).value();
            method = Constants.GET;
        } else if (declaredMethod.isAnnotationPresent(POST.class)) {
            multiPath = declaredMethod.getAnnotation(POST.class).value();
            method = Constants.POST;
        }

        Object endPointObject = clazz.newInstance();
        for (String path : multiPath) {
            path = StringUtils.trimSlash(path);
            EndpointMetaInfo metaInfo = EndpointMetaInfo.builder()
                    .path(path)
                    .endpointObject(endPointObject)
                    .method(declaredMethod)
                    .build();
            //只是把信息放到Guava提供的Table里面
            register0(method, path, metaInfo);
        }
    }

}
```

### 内嵌`Tomcat`实现

> 只需要端口号、`contextPath`即可，其他信息都可选。比较关键的是要指定`Servlet`的`Mapping`范围，比如下方的`/*`

```java
@Slf4j
public class MyEmbedServer implements EmbedServer {


    private final Tomcat tomcat;

    public MyEmbedServer(int port, String contextPath) {
        try {
            this.tomcat = new Tomcat();
            this.tomcat.setPort(port);

            File root = getRootFolder();
            File webContentFolder = new File(root.getAbsolutePath(), "src/main/resources/");
            if (!webContentFolder.exists()) {
                webContentFolder = Files.createTempDirectory("default-doc-base").toFile();
            }

            StandardContext standardContext = (StandardContext) this.tomcat.addWebapp(contextPath, webContentFolder.getAbsolutePath());
            standardContext.setParentClassLoader(this.getClass().getClassLoader());

            WebResourceRoot webResourceRoot = new StandardRoot(standardContext);
            standardContext.setResources(webResourceRoot);

            this.tomcat.addServlet(contextPath, "myMVCServlet", new MyMVCServlet()).setLoadOnStartup(0);

            //这边过滤了所有的请求，全部走这个servlet分发
            standardContext.addServletMappingDecoded("/*", "myMVCServlet");

        } catch (Exception e) {
            log.error("[MyMVC] Init Tomcat Server failed ", e);
            throw new RuntimeException(e);
        }


    }

    @SneakyThrows
    @Override
    public void start() {
        this.tomcat.start();
        log.info("[MyMVC] start success, address: http://{}:{}", this.tomcat.getServer().getAddress(), this.tomcat.getServer().getPort());
        this.tomcat.getServer().await();
    }
}
```

### 制作启动器

> 很简单的`Builder`模式，可以扩展为读取配置文件

```java
public class MyMVCBuilder {

    public static MyMVCBuilder of() {
        return new MyMVCBuilder();
    }

    public MyMVCBuilder port(int port) {
        MyMVCConfiguration.setPort(port);
        return this;
    }

    public MyMVCBuilder contextPath(String contextPath) {
        MyMVCConfiguration.setContextPath(contextPath);
        return this;
    }

    public void start(Class<?> startClass) {
        MyMVCConfiguration.setStartClass(startClass);
        new MyEmbedServer(MyMVCConfiguration.getPort(), MyMVCConfiguration.getContextPath()).start();
    }

}
```

### 请求流程模板代码

上方实现`MyMVCServlet`里面有重写`service`方法逻辑，`RequestHandlerTemplate.handle(req, resp)`这个就是处理请求的入口。实现也比较简单：

- 构造上下文
- 不同请求method的参数处理
- 反射调用执行器处理逻辑
- 渲染结果

> 这些步骤其实都可以用`SPI`扩展，有兴趣可以加

```java
@Slf4j
public class RequestHandlerTemplate {

    public static void handle(HttpServletRequest req, HttpServletResponse resp) throws IOException {

        try {
            //获取处理器
            RequestHandler handler = RequestHandlerFactory.getHandler(req.getMethod());

            //建立上下文
            RequestContext context = RequestContext.builder().req(req).resp(resp).build();

            //预处理
            handler.prepareContext(context);

            //解析参数
            context.setParamMap(handler.parseParam(context));

            //执行
            context.setOriginResult(handler.handleRequest(context));

            //处理结果
            handler.handleResponse(context);
        } catch (Exception e) {
            log.error("invoke method error, method={}", req.getMethod(), e);
            resp.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
        }
    }
}
```

### 解析入参

不同请求方式的解析处理，如`POST`请求的入参就有多种处理方式：

- `application/x-www-form-urlencoded`这种的请求会把参数放在parameterMap，解析同`GET`

```java
public static Map<String, Object> handleParameterMap(HttpServletRequest request) {

    Map<String, Object> paramMap = new HashMap<>();
    request.getParameterMap().forEach((paramName, paramsValues) -> {
        if (Objects.nonNull(paramsValues)) {
            paramMap.put(paramName, paramsValues[0]);
        }
    });
    return paramMap;

}
```

- `multipart/form-data`文件类型需要找一个临时文件夹放置

```java
public static Map<String, Object> handleMultiPart(HttpServletRequest request) {
    Map<String, Object> paramMap = new HashMap<>();

    ServletRequestContext servletRequestContext = new ServletRequestContext(request);
    String uploadPath = request.getServletContext().getRealPath("./") + File.separator + UPLOAD_DIRECTORY;

    try {
        List<FileItem> formItems = uploader.parseRequest(servletRequestContext);

        if (formItems != null && formItems.size() > 0) {

            for (FileItem item : formItems) {
                // 文件类型处理
                if (!item.isFormField()) {
                    String fileName = new File(item.getName()).getName();
                    String filePath = uploadPath + File.separator + fileName;
                    File storeFile = new File(filePath);
                    paramMap.put(item.getFieldName(), storeFile);
                } else {
                    paramMap.put(item.getFieldName(), item.getString());
                }
            }
        }
    } catch (Exception ex) {
        throw new RuntimeException("upload file error ", ex);
    }
    return paramMap;

}
```

- `application/json`JSON类型直接从Body获取，按照流方式读取

```java
public static Map<String, Object> handleJSON(HttpServletRequest req) throws IOException {

    // JSON 单行解析
    BufferedReader br = new BufferedReader(new InputStreamReader(req.getInputStream(), StandardCharsets.UTF_8));
    String line;
    StringBuilder sb = new StringBuilder();
    while ((line = br.readLine()) != null) {
        sb.append(line);
    }
    //将json字符串转换为json对象
    return JSONObject.parseObject(sb.toString(), MSS_TYPE_REFERENCE);
}
```

### 执行器调用

执行器调用，入口为上方模板代码里的`handler.handleRequest(context)`。

> 下方的`getReflectParameters`方法实现的是将`Map<String, Object>`这样的解析参数转化为需要反射调用的入参。至于为什么要用注解获取入参，这是由于JVM编译后，原入参名字无法保留，要开启的话也可以，会有性能损失

```java
@SneakyThrows
public static Object invokeEndpoint(EndpointMetaInfo endpoint, Map<String, Object> paramMap) {

	 // 通过反射组装endpoint需要的入参
    List<Object> parameters = getReflectParameters(endpoint, paramMap);
    Object result;
    //反射调用
    if (parameters.size() > 0) {
        result = endpoint.getMethod().invoke(endpoint.getEndpointObject(), parameters.toArray());
    } else {
        result = endpoint.getMethod().invoke(endpoint.getEndpointObject());
    }
    return result;
}


public static List<Object> getReflectParameters(EndpointMetaInfo endpoint, Map<String, Object> requestMap) {

    if (endpoint.getMethod().getParameters().length == 0) {
        return Collections.emptyList();
    }

    Parameter[] parameters = endpoint.getMethod().getParameters();
    Class<?>[] parameterTypes = endpoint.getMethod().getParameterTypes();
    List<Object> objectList = new ArrayList<>();

    checkRequestType(endpoint.getMethod());

    //这边主要实现GET、POST的不同参数赋值逻辑
    for (int i = 0; i < parameters.length; i++) {
        Parameter param = parameters[i];
        Class<?> type = parameterTypes[i];

        //通过标记注解反射赋值回入参
        PARAM pinedParam = param.getAnnotation(PARAM.class);
        BODY pinedBody = param.getAnnotation(BODY.class);

        if (Objects.nonNull(pinedParam) && requestMap.containsKey(pinedParam.value())) {
            objectList.add(ClassUtil.generateParameterObject(type, requestMap.get(pinedParam.value())));
        } else if (Objects.nonNull(pinedBody)) {
            objectList.add(ClassUtil.generateBodyObject(type, requestMap));
        } else {
            objectList.add(ClassUtil.generateDefaultObject(type));
        }
    }


    return objectList;
}
```



### 启动

完成大体的框架，可以实现GET、POST的请求解析了，启动方式：

```java
public class AppTest 
{

    @Test
    public void start() {
        MyMVCBuilder.of().start(App.class);
    }
}
```



## 总结

这仅仅是一个很简单的`Web`框架，功能还很不完善，不过大体的`MVC`处理流程都实现了，可以帮助读者理解实现`MVC`的几个核心点



## TODO

- Filter

- REST

- Cookie 

- Session

- CSRF

- Configuration

  



地址：https://github.com/Wushiyii/MyMVC