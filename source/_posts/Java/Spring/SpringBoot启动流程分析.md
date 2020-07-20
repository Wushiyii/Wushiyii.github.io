---
title: SpringBoot启动流程分析
date: 2020-07-20 10:12:02
tags: [JAVA,Spring]
categories: JAVA
---

### 1.构造SpringBoot实例

一个`SpringBoot`的启动入口，往往是通过一个main函数+@SpringBootApplication启动，并且在这之上会有很多特性注解。

```java
@SpringBootApplication(scanBasePackages = {"com.xxx", "com.xxx"})
@MapperScan(value = "com.xxx", sqlSessionTemplateRef = "xxxSqlSessionTemplate")
@EnableTransactionManagement(proxyTargetClass = true)
@EnableRetry
@EnableAsync
@EnableDubbo(scanBasePackages = "com.xxx")
public class TestApplication {
    public static void main(String[] args) {
        SpringApplication.run(TestApplication.class, args);
    }
}
```

接着往下，SpringApplication.run 会去创建一个新的SpringApplication实例

```java
public static ConfigurableApplicationContext run(Class<?>[] primarySources,
      String[] args) {
   return new SpringApplication(primarySources).run(args);
}
```

Spring的构造函数，主要做了检验当前的应用环境（REACTIVE、SERVLET、NONE，见deduceFromClasspath方法）、设置应用的初始化器、监听器

```java
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
   this.resourceLoader = resourceLoader;
   Assert.notNull(primarySources, "PrimarySources must not be null");
   this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
  // 判断当前的应用环境
   this.webApplicationType = WebApplicationType.deduceFromClasspath();
  // 设置应用初始化启动器，会扫描所有jar包的 META-INF/spring.factories 中找到ApplicationContextInitializer 并实例化初始化器
   setInitializers((Collection) getSpringFactoriesInstances(
         ApplicationContextInitializer.class));
  // 设置应用初始化监听器，会扫描所有jar包的 META-INF/spring.factories 中找到ApplicationListener 并实例化初始化器
   setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
  // 根据当前Runtime对应的栈帧列表StackTraceElement，判断并设置启动springboot的main类
   this.mainApplicationClass = deduceMainApplicationClass();
}
```

这样，一个Springboot的实例就构造完了，接下来是启动的部分

### 2.启动

先看一下启动源码

```java
public ConfigurableApplicationContext run(String... args) {
   // 初始化一个计时器
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
   configureHeadlessProperty();
  // 获取SpringApplicationRunListener（如有配置）
   SpringApplicationRunListeners listeners = getRunListeners(args);
  
  //Springboot启动时，监听starting步骤
   listeners.starting();
   try {
      // 构造启动Springboot的参数对象
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
     
      ConfigurableEnvironment environment = prepareEnvironment(listeners,
            applicationArguments);
     // 配置spring.beaninfo.ignore的值
      configureIgnoreBeanInfo(environment);
     // 打印banner
      Banner printedBanner = printBanner(environment);
     // 创建ApplicationContext容器
      context = createApplicationContext();
     // 获取SpringBootExceptionReporter，收集上报exception用
      exceptionReporters = getSpringFactoriesInstances(
            SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
     // 执行创建完ApplicationContext的回调方法、执行各个应用的初始化initialize()方法
      prepareContext(context, environment, listeners, applicationArguments,
            printedBanner);
     // 刷新context容器(Beanfatroy刷新、BeanPostProcesser注册、JavaConfig解析等等..)
      refreshContext(context);
     // 执行完refresh的hook
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass)
               .logStarted(getApplicationLog(), stopWatch);
      }
      listeners.started(context);
     // 执行runner方法
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```

applicationContext的构造、加载与最终完成分为多个阶段

- SpringbootRunListener记录当前context执行到了starting阶段，并广播event到所有监听类

- createApplicationContext（创建applicationContext）

  - 判断当前应用环境，SERVLET(SpringMVC)、REACTIVE（WebFlux）、NONE(默认的非web)，并实例化对应的applicationContext

  - ```java
    protected ConfigurableApplicationContext createApplicationContext() {
       Class<?> contextClass = this.applicationContextClass;
       if (contextClass == null) {
          try {
            // 判断应用环境，创建对应的applicationContext对象
             switch (this.webApplicationType) {
             case SERVLET:
                contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                break;
             case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
             default:
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
             }
          }
          catch (ClassNotFoundException ex) {
             throw new IllegalStateException(
                   "Unable create a default ApplicationContext, please specify an ApplicationContextClass",ex);
          }
       }
       return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
    }
    ```

- prepaerContext（准备applicationContext）

  - 执行创建完applicationContext的回调方法

  - 执行各个应用的初始化器对应的initialize()方法（初始化器为META-INF/spring.factories中配置的ApplicationContextInitializer）

    - ```java
      protected void applyInitializers(ConfigurableApplicationContext context) {
        // 获取所有初始化器
         for (ApplicationContextInitializer initializer : getInitializers()) {
            Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
                  initializer.getClass(), ApplicationContextInitializer.class);
            Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
           // 执行
            initializer.initialize(context);
         }
      }
      ```

  - SpringbootRunListener记录当前context执行到了contextPrepared阶段，并广播event到所有监听类

  - 打印startUp信息`2020/07/20 15:28:57.604  main [INFO] XxxApplication (StartupInfoLogger.java:50) Starting XxxApplication on bogon with PID 31681 (/Users/workspace/java/xxx//target/classes started by xxx in /Users/xxx/workspace/java/xxx)`

  - 加载启动主类（XxxAppcalition）
    - 创建BeanDefinitionLoader
    - 加载XxxAppcalition
    
  - SpringbootRunListener记录当前context执行到了contextLoaded阶段，并广播event到所有监听类

- refreshContext（刷新容器，方法很多）

  - ```java
    public void refresh() throws BeansException, IllegalStateException {
       synchronized (this.startupShutdownMonitor) {
          prepareRefresh();
          ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
          prepareBeanFactory(beanFactory);
          try {
             postProcessBeanFactory(beanFactory);
             invokeBeanFactoryPostProcessors(beanFactory);
             registerBeanPostProcessors(beanFactory);
             initMessageSource();
             initApplicationEventMulticaster();
             onRefresh();
             registerListeners();
             finishBeanFactoryInitialization(beanFactory);
             finishRefresh();
          }
          catch (BeansException ex) {
             destroyBeans();
             cancelRefresh(ex);
             throw ex;
          }
          finally {
             resetCommonCaches();
          }
       }
    }
    ```

  - prepareRefresh（准备刷新）

    - ```java
      protected void prepareRefresh() {
         // 恢复到激活状态
         this.startupDate = System.currentTimeMillis();
         this.closed.set(false);
         this.active.set(true);
         // ...
         // 初始化placeholder对应的值
         initPropertySources();
         //验证环境必须属性
         getEnvironment().validateRequiredProperties();
         // ...
      }
      ```

  - obtainFreshBeanFactory（刷新beanFactory，步骤很简单，就是干掉所有单例bean、干掉beanFactory、重新创建beanFacotry、重新加载beanDefinitions）

    - ```java
      protected final void refreshBeanFactory() throws BeansException {
        // 如果存在beanFactory，则干掉所有bean、关闭beanFactory
         if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
         }
         try {
           // 创建一个listableBeanFactory
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            customizeBeanFactory(beanFactory);
           // 加载beanDefinitions
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
               this.beanFactory = beanFactory;
            }
         }
         catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
         }
      }
      ```

      

  - prepareBeanFactory（拿到重新创建的beanFacotry，配置一些默认参数、接口、classlorder）

    - ```java
      protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
         // 设置ClassLoarder、设置表达式解析器、添加属性编辑注册器
         beanFactory.setBeanClassLoader(getClassLoader());
         beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
         beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));
      
         // 设置ApplicationContextAwareProcessor, 以下6个ignoreDependencyInterface的目的是因为ApplicationContextAwareProcessor已经把这6个的工作做了
         beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
         beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
         beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
         beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
         beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
         beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
         beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
      
         // 设置几个特殊dependency类型bean
         beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
         beanFactory.registerResolvableDependency(ResourceLoader.class, this);
         beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
         beanFactory.registerResolvableDependency(ApplicationContext.class, this);
      
         // Register early post-processor for detecting inner beans as ApplicationListeners.
         beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));
      
         // ...
         // 设置一些默认变量
         if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
            beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
         }
         if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
            beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
         }
         if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
            beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
         }
      }
      ```

  - postProcessBeanFactory（设置一些需要提前创建的BeanPostProcessors类）

    - BeanPostProcessor的作用是在bean实例化前后都有对应的调用方法可以执行

    -  -->  Spring IOC容器实例化Bean
       -->  调用BeanPostProcessor的postProcessBeforeInitialization方法
       -->  调用bean实例的初始化方法
       -->  调用BeanPostProcessor的postProcessAfterInitialization方法

    - ```java
      @Override
      protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        // 将WebApplicationContextServletContextAwareProcessor设置到postProcesser中
         beanFactory.addBeanPostProcessor(
               new WebApplicationContextServletContextAwareProcessor(this));
         beanFactory.ignoreDependencyInterface(ServletContextAware.class);
         registerWebApplicationScopes();
      }
      ```

  - invokeBeanFactoryPostProcessors（实例化所有BeanPostProcessors类）

    - ```java
      protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
        	// 实例化所有BeanPostProcessors类
         PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
      
         // Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
         // (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
         if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
            beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
            beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
         }
      }
      ```

  - registerBeanPostProcessors

  - initMessageSource（初始化messageSource（国际化支持））

  - initApplicationEventMulticaster（初始化ApplicationEventMulticaster（事件广播器，用于通知事件到实现了ApplicationListener的类））

    - ```java
      protected void initApplicationEventMulticaster() {
        // 从ConfigurableListableBeanFactory中判断是否有applicationEventMulticaster，有则赋值、无准创建并注册到BeanFactory
         ConfigurableListableBeanFactory beanFactory = getBeanFactory();
         if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
            this.applicationEventMulticaster =
                  beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
         }
         else {
           // 创建
            this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
           // 注册单例对象
            beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
         }
      }
      ```

  - onRefresh（执行其他特殊beans的模板方法）

    - ```java
      @Override
      protected void onRefresh() {
         super.onRefresh();
         try {
           // ServletWebServerApplicationContext实现的是创建一个webServer
            createWebServer();
         }
         catch (Throwable ex) {
            throw new ApplicationContextException("Unable to start web server", ex);
         }
      }
      ```

    - ```java
      private void createWebServer() {
         WebServer webServer = this.webServer;
         ServletContext servletContext = getServletContext();
         if (webServer == null && servletContext == null) {
           // getWebServerFactory会从ServletWebServerFactory中取到当前环境的web容器bean（Tomcat、UnderTow、Jetty）
            ServletWebServerFactory factory = getWebServerFactory();
            this.webServer = factory.getWebServer(getSelfInitializer());
         }
         else if (servletContext != null) {
            try {
               getSelfInitializer().onStartup(servletContext);
            }
            catch (ServletException ex) {
               throw new ApplicationContextException("Cannot initialize servlet context",
                     ex);
            }
         }
         initPropertySources();
      }
      ```

  - registerListeners（初始化Springboot里的ApplicationListeners，并注册到ApplicationEventMulticaster里）

    - ```java
      protected void registerListeners() {
         // 注册所有ApplicationListeners到EventMulticaster中
         for (ApplicationListener<?> listener : getApplicationListeners()) {
            getApplicationEventMulticaster().addApplicationListener(listener);
         }
      	// ...
      
        // 如果存在earlyApplicationEvents，则直接广播出去
         Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
         this.earlyApplicationEvents = null;
         if (earlyEventsToProcess != null) {
            for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
               getApplicationEventMulticaster().multicastEvent(earlyEvent);
            }
         }
      }
      ```

  - finishBeanFactoryInitialization（实例化BeanFactory中已经被注册但是未实例化的所有实例(Lazy init除外)）

  - finishRefresh（完成refresh，执行LifecycleProcessor的onRefresh回调）

    - ```java
      protected void finishRefresh() {
         // 清除resource缓存
         clearResourceCaches();
      
         // 初始化LifecycleProcessor
         initLifecycleProcessor();
      
         // 执行onRefresh方法
         getLifecycleProcessor().onRefresh();
      
         // 发布容器refresh完成的通知
         publishEvent(new ContextRefreshedEvent(this));
      
         // Participate in LiveBeansView MBean, if active.
         LiveBeansView.registerApplicationContext(this);
      }
      ```

  - resetCommonCaches（恢复一些Util的缓存）

    - ```java
      protected void resetCommonCaches() {
         ReflectionUtils.clearCache();
         AnnotationUtils.clearCache();
         ResolvableType.clearCache();
         CachedIntrospectionResults.clearClassLoader(getClassLoader());
      }
      ```

- afterRefresh（预留方法，作为refresh后的hook调用）

- SpringbootRunListener记录当前context执行到了started阶段，并广播event到所有监听类

- callRunners

  - 获取所有实现ApplicationRunner、CommandLineRunner的类

  - 根据@Ordered注解排序调用Runner的run方法

  - ```java
    private void callRunners(ApplicationContext context, ApplicationArguments args) {
       List<Object> runners = new ArrayList<>();
      // 从listableBeanFactory获取所有实现ApplicationRunner、CommandLineRunner的类
       runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
       runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
      // 根据Ordered注解排序
       AnnotationAwareOrderComparator.sort(runners);
       for (Object runner : new LinkedHashSet<>(runners)) {
          if (runner instanceof ApplicationRunner) {
             callRunner((ApplicationRunner) runner, args);
          }
          if (runner instanceof CommandLineRunner) {
             callRunner((CommandLineRunner) runner, args);
          }
       }
    }
    ```

- SpringbootRunListener记录当前context执行到了running阶段，并广播event到所有监听类



至此就完成了Springboot的启动，不过这只是主体的启动流程，还有一些细枝末节的地方没有分析。