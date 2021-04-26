---
layout:     post
title:      "SpringBoot专题-3-启动流程分析"
subtitle:   " \"启动流程分析\""
date:       2021-04-26 18:09:00 +0800
author:     "snowball"
header-img: "img/home-bg-o.jpg"
tags:
    - SpringBoot启动流程
    - SpringBoot学习笔记
    - 技术
typora-root-url: ..
---

<!--  “Spring Boot. ” -->

## 1.SpringApplication类

### 构造器

```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader;//此时为null,可以通过此参数指定类加载器
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    //推断应用类型,reactive,servlet
    this.webApplicationType = WebApplicationType.deduceFromClasspath();
    //初始化classpath下 META-INF/spring.factories中配置的ApplicationContextInitializer
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
    //初始化classpath下所有配置的 ApplicationListener(META-INF/spring.factories)
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    //根据调用栈，推断出 main 方法的类名
    this.mainApplicationClass = deduceMainApplicationClass();
}
```

初始化并设置了ApplicationContextInitializer、ApplicationListener的bean

### spring.factories

```
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
    return getSpringFactoriesInstances(type, new Class<?>[] {});
}
//通过getClassLoader 从META-INF/spring.factories获取指定的Spring的工厂实例
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
    //默认为Thread.currentThread().getContextClassLoader()/ClassLoader.getSystemClassLoader()
    ClassLoader classLoader = getClassLoader();
    //读取 key 为 type.getName() 的 value
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
    //反射创建Bean
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
    AnnotationAwareOrderComparator.sort(instances);
    return instances;
}
```

获取指定类权限名对应的实例，可以自定义事件监听器和初始化干预

### 反射创建Bean

整个 springBoot 框架中获取factories的统一方式

```
private <T> List<T> createSpringFactoriesInstances(Class<T> type,Class<?>[] parameterTypes, ClassLoader                                 classLoader, Object[] args,Set<String> names) {
    List<T> instances = new ArrayList<>(names.size());
    for (String name : names) {
        try {
            //装载class文件到内存
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);
            Assert.isAssignable(type, instanceClass);
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);
            //通过反射创建实例
            T instance = (T) BeanUtils.instantiateClass(constructor, args);
            instances.add(instance);
        } catch (Throwable ex) {
            throw new IllegalArgumentException(
                "Cannot instantiate " + type + " : " + name, ex);
        }
    }
    return instances;
}
```

## 2.run方法

运行spring应用，并刷新ApplicationContext(ConfigurableApplicationContext)

```
public ConfigurableApplicationContext run(String... args) {
    StopWatch stopWatch = new StopWatch(); //记录运行时间
    stopWatch.start();
    ConfigurableApplicationContext context = null;
    Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
    configureHeadlessProperty();//java.awt.headless是J2SE的一种模式用于在缺少显示屏、键盘或者鼠标时的系统配置，很多监控工具如jconsole 需要将该值设置为true，系统变量默认为true
    
    //从META-INF/spring.factories中获取监听器  SpringApplicationRunListeners
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting();//遍历回调SpringApplicationRunListeners的starting方法
    try {   
        //封装命令行参数
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
        //构造应用上下文环境，完成后回调SpringApplicationRunListeners的environmentPrepared方法
        ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
        configureIgnoreBeanInfo(environment);//处理需要忽略的Bean
        Banner printedBanner = printBanner(environment);//打印banner
        //根据是否web环境创建相应的IOC容器
        context = createApplicationContext();
        //实例化SpringBootExceptionReporter，用来支持报告关于启动的错误
        exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                          new Class[] { ConfigurableApplicationContext.class}, context);
        
        //准备上下文环境，将environment保持到IOC容器中
        //执行applyInitializers，遍历回调ApplicationContextInitializer的initialize方法
        //遍历回调SpringApplicationRunListeners的contextPrepared方法
        //遍历回调SpringApplicationRunListeners的contextLoaded方法
        prepareContext(context, environment, listeners, applicationArguments, printedBanner);
        refreshContext(context);//刷新应用上下文,组件扫描、创建、加载
        
        //从IOC容器获取所有的ApplicationRunner（先调用）和CommandLinedRunner进行回调
        afterRefresh(context, applicationArguments);
        stopWatch.stop();//时间记录停止
        if (this.logStartupInfo) {
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
        }
        listeners.started(context);//发布容器启动完成事件
        callRunners(context, applicationArguments);
    }catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, listeners);
        throw new IllegalStateException(ex);
    }
    try {
        listeners.running(context);
    }catch (Throwable ex) {
        handleRunFailure(context, ex, exceptionReporters, null);
        throw new IllegalStateException(ex);
    }
    return context;
}
```

1. 获取并启动监听器
2. 构造容器环境
3. 初始化容器，创建容器
4. 报告错误信息
5. 刷新应用上下文前的准备阶段
6. 刷新容器
7. 刷新容器后的扩展接口

### 获取并启动监听器

#### 获取监听器 

getRunListeners：加载spring.factories中的监听器EventPublishingRunListener

```
public EventPublishingRunListener(SpringApplication application, String[] args) {
    this.application = application;
    this.args = args;
    this.initialMulticaster = new SimpleApplicationEventMulticaster();
    for (ApplicationListener<?> listener : application.getListeners()) {
        this.initialMulticaster.addApplicationListener(listener);
    }
}
//SimpleApplicationEventMulticaster extends  AbstractApplicationEventMulticaster
public void addApplicationListener(ApplicationListener<?> listener) {
    synchronized (this.retrievalMutex) {
        //避免重复  放入set，去掉代理对象
        Object singletonTarget = AopProxyUtils.getSingletonTarget(listener);
        if (singletonTarget instanceof ApplicationListener) {
            this.defaultRetriever.applicationListeners.remove(singletonTarget);
        }
        //内部类对象，保存所有的监听器
        this.defaultRetriever.applicationListeners.add(listener);
        this.retrieverCache.clear();
    }
}
```

SimpleApplicationEventMulticaster父类AbstractApplicationEventMulticaster中。关键代码为this.defaultRetriever.applicationListeners.add(listener);，这是一个内部类，用来保存所有的ApplicationListener监听器。也就是在这一步，将spring.factories中的监听器传递到SimpleApplicationEventMulticaster中。

#### 启动监听器

```
public void starting() {
    //关键代码，这里是创建application启动事件:ApplicationStartingEvent发布启动事件
    this.initialMulticaster.multicastEvent(new ApplicationStartingEvent(this.application, this.args));
}
```

`EventPublishingRunListener`这个是springBoot框架中最早执行的监听器，在该监听器执行`started()`方法时，会继续发布事件，也就是事件传递。这种实现主要还是基于spring的事件机制。继续跟进`SimpleApplicationEventMulticaster`，有个核心方法：

```
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    //找出匹配该事件的监听器
    for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        //获取线程池，如果为空则同步处理。这里线程池为空，还未初始化。
        Executor executor = getTaskExecutor();
        if (executor != null) {
            //异步发送事件
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            //同步发送事件
            invokeListener(listener, event);
        }
    }
}
protected Collection<ApplicationListener<?>> getApplicationListeners(
    ....构建缓存
    Collection<ApplicationListener<?>> listeners = retrieveApplicationListeners(eventType, sourceType,                                                                      retriever);
    this.retrieverCache.put(cacheKey, retriever);
    return listeners;
    ...
}
    
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
    ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable ListenerRetriever retriever) {
    List<ApplicationListener<?>> allListeners = new ArrayList<>();
    Set<ApplicationListener<?>> listeners;
    Set<String> listenerBeans;
    synchronized (this.retrievalMutex) {
        listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
        listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
    }
    //找出适配监听事件的监听器
    for (ApplicationListener<?> listener : listeners) {
        if (supportsEvent(listener, eventType, sourceType)) {
            if (retriever != null) {
                retriever.applicationListeners.add(listener);
            }
            allListeners.add(listener);
        }
    }
    ...
```

这是springBoot启动过程中，第一处根据类型，执行监听器的地方，根据发布的事件类型从所有监听器中选择对应的监听器进行事件发布

### 构造容器环境

#### prepareEnvironment

首先是创建并按照相应的应用类型配置相应的环境，然后根据用户的配置，配置系统环境，然后启动监听器，并加载系统配置文件。

```
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                   ApplicationArguments applicationArguments) {
    //创建并配置相应的环境,获取对应的ConfigurableEnvironment
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    //根据用户配置，配置 environment系统环境
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    //发布监听事件， ConfigFileApplicationListener 就是加载项目配置文件的监听器。
    listeners.environmentPrepared(environment);
    bindToSpringApplication(environment);
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
                                                    deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
//根据环境创建对应ConfigurableEnvironment
private ConfigurableEnvironment getOrCreateEnvironment() {
    if (this.environment != null) {
        return this.environment;
    }
    switch (this.webApplicationType) {
        case SERVLET:
            return new StandardServletEnvironment();//Web程序
        case REACTIVE:
            return new StandardReactiveWebEnvironment();//响应式web环境
        default:
            return new StandardEnvironment();//普通程序
    }
}
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
    if (this.addConversionService) {
        ConversionService conversionService = ApplicationConversionService.getSharedInstance();
        environment.setConversionService((ConfigurableConversionService) conversionService);
    }
    // 将main 函数的args封装成 SimpleCommandLinePropertySource 加入environment中。
    configurePropertySources(environment, args);
    // 激活相应的配置文件
    configureProfiles(environment, args);
}
```

#### environmentPrepared

配置文件的配置信息加入environment，最终调用事件发布

### 创建容器

```
protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
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
        }catch (ClassNotFoundException ex) {
            throw new IllegalStateException(
             "Unable create a default ApplicationContext, please specify an ApplicationContextClass", ex);
        }
    }
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

### 报告错误信息

该类主要是在项目启动失败之后，打印log

```
SpringBootExceptionReporter exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
                   new Class[] { ConfigurableApplicationContext.class }, context);
                   
private void reportFailure(Collection<SpringBootExceptionReporter> exceptionReporters,
                           Throwable failure) {
    try {
        for (SpringBootExceptionReporter reporter : exceptionReporters) {
            if (reporter.reportException(failure)) {
                //上报错误log
                registerLoggedException(failure);
                return;
            }
        }
    }catch (Throwable ex) {
        // Continue with normal handling of the original failure
    }
    if (logger.isErrorEnabled()) {
        logger.error("Application run failed", failure);
        registerLoggedException(failure);
    }
}
```

### 刷新容器前的准备阶段

prepareContext：将启动类注入容器，为后续开启自动化配置奠定基础。

```
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
                            SpringApplicationRunListeners listeners, ApplicationArguments                                                   applicationArguments, Banner printedBanner) {
    //设置容器环境
    context.setEnvironment(environment);
    postProcessApplicationContext(context);//执行容器后置处理
    //执行容器中的 ApplicationContextInitializer 包括spring.factories和通过三种方式自定义的
    applyInitializers(context);
    //向各个监听器发送容器已经准备好的事件
    listeners.contextPrepared(context);
    if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
    }
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    //将main函数中的args参数封装成单例Bean，注册进容器
    beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
    if (printedBanner != null) {
         //将 printedBanner 封装成单例，注册进容器
        beanFactory.registerSingleton("springBootBanner", printedBanner);
    }
    if (beanFactory instanceof DefaultListableBeanFactory) {
        ((DefaultListableBeanFactory) beanFactory)
        .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
    }
    if (this.lazyInitialization) {
        context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
    }
    
    Set<Object> sources = getAllSources();//拿到启动类
    Assert.notEmpty(sources, "Sources must not be empty");
    //加载启动类，将启动类注入容器
    load(context, sources.toArray(new Object[0]));
    //发布容器已加载事件
    listeners.contextLoaded(context);
}
```

#### 配置bean 生成器及资源加载器

```
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
    //如果设置了是实例命名生成器，注册到Spring容器中
    if (this.beanNameGenerator != null) {
        context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
            this.beanNameGenerator);
    }
    // 如果设置了资源加载器，设置到Spring容器中
    if (this.resourceLoader != null) {
        if (context instanceof GenericApplicationContext) {
            ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
        }
        if (context instanceof DefaultResourceLoader) {
            ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());
        }
    }
    if (this.addConversionService) {
        context.getBeanFactory().setConversionService(ApplicationConversionService.getSharedInstance());
    }
}
```

这里默认不执行任何逻辑，因为`beanNameGenerator`和`resourceLoader`默认为空。springBoot预留的扩展处理方式，配置上下文的 bean 生成器及资源加载器

#### 加载启动类

```
protected void load(ApplicationContext context, Object[] sources) {
    if (logger.isDebugEnabled()) {
        logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));
    }
    //创建 BeanDefinitionLoader    将上下文context强转为BeanDefinitionRegistry
    BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);
    if (this.beanNameGenerator != null) {
        loader.setBeanNameGenerator(this.beanNameGenerator);
    }
    if (this.resourceLoader != null) {
        loader.setResourceLoader(this.resourceLoader);
    }
    if (this.environment != null) {
        loader.setEnvironment(this.environment);
    }
    loader.load();
}
//createBeanDefinitionLoader进入构造器
BeanDefinitionLoader(BeanDefinitionRegistry registry, Object... sources) {
    Assert.notNull(registry, "Registry must not be null");
    Assert.notEmpty(sources, "Sources must not be empty");
    this.sources = sources;
    //注解形式的Bean定义读取器 比如：@Configuration @Bean @Component @Controller @Service等等
    this.annotatedReader = new AnnotatedBeanDefinitionReader(registry);
    this.xmlReader = new XmlBeanDefinitionReader(registry);//XML形式的Bean定义读取器
    if (isGroovyPresent()) {
        this.groovyReader = new GroovyBeanDefinitionReader(registry);
    }
    this.scanner = new ClassPathBeanDefinitionScanner(registry); //类路径扫描器
    this.scanner.addExcludeFilter(new ClassExcludeFilter(sources));//扫描器添加排除过滤器
}
private int load(Object source) {
    Assert.notNull(source, "Source must not be null");
    //如果是class类型，启用注解类型
    if (source instanceof Class<?>) {
        return load((Class<?>) source);
    }
    //如果是resource类型，启用xml解析
    if (source instanceof Resource) {
        return load((Resource) source);
    }
    //如果是package类型，启用扫描包，例如：@ComponentScan
    if (source instanceof Package) {
        return load((Package) source);
    }
    //如果是字符串类型，直接加载
    if (source instanceof CharSequence) {
        return load((CharSequence) source);
    }
    throw new IllegalArgumentException("Invalid source type " + source.getClass());
}
```

#### 发布ApplicationPreparedEvent事件

```
public void contextLoaded(ConfigurableApplicationContext context) {
    for (ApplicationListener<?> listener : this.application.getListeners()) {
        if (listener instanceof ApplicationContextAware) {
            ((ApplicationContextAware) listener).setApplicationContext(context);
        }
        context.addApplicationListener(listener);
    }
    this.initialMulticaster.multicastEvent(new ApplicationPreparedEvent(this.application, this.args, context));
}
```

上面会将spring.factories中的DelegatingApplicationListener放入容器上下文，从而该实例可以监听容器上下文发布的事件，也可以监听springboot中的事件(委派对象)

### Spring刷新容器

context.refresh()

### 刷新容器后的扩展接口

```
protected void afterRefresh(ConfigurableApplicationContext context,
                            ApplicationArguments args) {
}
```

扩展接口，设计模式中的模板方法，默认为空实现。如果有自定义需求，可以重写该方法。比如打印一些启动结束log，或者一些其它后置处理。

### 发布ApplicationStartedEvent事件

## 事件驱动

![事件驱动](/img/in-post/post-springboot/事件驱动.png)

## 启动流程

![启动流程](/img/in-post/post-springboot/启动流程.png)