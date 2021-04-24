---
layout:     post
title:      "SpringBoot专题-1-零配置及内嵌Tomcat原理"
subtitle:   " \"零配置及内嵌Tomcat原理\""
date:       2021-04-24 11:09:00 +0800
author:     "snowball"
header-img: "img/home-bg-o.jpg"
tags:
    - SpringBoot零配置
    - 内嵌Tomcat
    - SpringBoot学习笔记
    - 技术
typora-root-url: ..
---

<!--  “Spring Boot. ” -->

我们都知道SpringBoot是做到了零配置实现Web应用启动，根据以前开发Web应用的传统xml配置，有以下几个问题需要思考：

- XML的Bean配置被JavaConfig替代，内部的实现机制是什么？
- Tomcat内嵌方式，SpringBoot是如何在Tomcat启动后进行回调到SpringBoot的回调方法中？
- SpringMVC的DispatcherServlet怎么注册到Tomcat中的，默认配置怎么加载的？

## 零配置原理

基于Spring新特性JavaConfig，Spring JavaConfig是Spring社区的产品，使用Java代码配置Spring IoC容器。不需要使用XML配置。

**JavaConfig**的优点：

- 面向对象的配置。配置被定义为JavaConfig类，因此用户可以充分利用Java中的面向对象功能。一个配置类可以继承另一个，重写它的@Bean方法等。
- 减少或消除XML配置。许多开发人员不希望在XML和Java之间来回切换。JavaConfig为开发人员提供了一种纯Java方法来配置与XML配置概念相似的Spring容器。
- 类型安全和重构友好。JavaConfig提供了一种类型安全的方法来配置Spring容器。由于Java 5.0对泛型的支持，现在可以按类型而不是按名称检索bean，不需要任何强制转换或基于字符串的查找

Spring解析Java配置类的过程可参考之前的文章：[Spring解析配置类以及扫描源码解析](https://www.snowballzz.com/2021/04/08/Spring%E4%B8%93%E9%A2%98-8-Spring%E8%A7%A3%E6%9E%90%E9%85%8D%E7%BD%AE%E7%B1%BB%E4%BB%A5%E5%8F%8A%E6%89%AB%E6%8F%8F%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90/)



之前的web.xml：

ContextLoaderListener：监听servlet启动、销毁，初始化ioc完成依赖注入

DispatcherServlet：接收tomcat解析之后的http请求，匹配controller处理业务

代码替换：

```
AnnotationConfigWebApplicationContext ac = new AnnotationConfigWebApplicationContext();
ac.register(Start.class);
//ac.refresh();

DispatcherServlet servlet = new DispatcherServlet(ac);
```



之前的applicationContext.xml:

component-scan：扫描   注解替换： @ComponentScan

\<beans>  \<bean>  注解替换：  @Configuration +@Bean  

之前的springmvc.xml:

component-scan：扫描  只扫描@Controller   注解替换： @ComponentScan

视图解析器，json转换，国际化，编码。。。。  注解替换：  @Configuration +@Bean  

代码替换：

```
@Configuration
@EnableWebMvc
public class MyConfig implements WebMvcConfigurer{

    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        FastJsonHttpMessageConverter builder = new FastJsonHttpMessageConverter();
        converters.add(builder);
    }
}
```

## 内嵌Tomcat添加Web应用原理

apache 在tomcat7提供内嵌版本的tomcat  tomcat.jar

```
public static void createServer() throws Exception{
    Tomcat tomcat = new Tomcat();
    tomcat.setPort(8088);

    tomcat.addWebapp("/","D://tomcat");//完成监听

    tomcat.start();
    tomcat.getServer().await();
}
```

### 1、addWebapp解析

通过反射加载监听类LifecycleListener：ContextConfig

ContextConfig监听方法进行配置启动：configureStart()

- webConfig()：读取/WEB-INF/web.xml（如果有）
- processServletContainerInitializers()：完成spi类的扫描和加载（ServletContainerInitializer），存入initializerClassMap
- 遍历initializerClassMap将ServletContainerInitializer存入context（StandardContext的全局变量initializers）

### 2、start解析

```
public void start() throws LifecycleException {
    getServer();//初始化server容器及service子容器
    getConnector();//初始化Connector
    server.start();//调用startInternal()
}
```

StandardContext的startInternal()方法：遍历initializers，调用ServletContainerInitializer的onStartup()方法

```
for (Map.Entry<ServletContainerInitializer, Set<Class<?>>> entry :
                initializers.entrySet()) {
    try {
        entry.getKey().onStartup(entry.getValue(),
                                 getServletContext());
    } catch (ServletException e) {
        log.error(sm.getString("standardContext.sciFail"), e);
        ok = false;
        break;
    }
}
```

1. 加载监听器LifecycleListener
2. 监听器实现类：ContextConfig->lifecycleEvent   ->webConfig
3. 自定义类加载器加载ServletContainerInitializer实现类  扫描META-INF/services/  放入 initializerClassMap
4. initializerClassMap转换到StandardContext的全局变量initializers
5. tomcat启动start方法调用StandardContext.startInternal  遍历initializers  调用onstartup方法

## SpringBoot内嵌Tomcat

核心过程：

```
SpringApplication.run->refreshContext->applicationContext.refresh()->
ServletWebServerApplicationContext.onRefresh->createWebServer()->
this::selfInitialize->ServletContextInitializer.onStartup->
DispatcherServletRegistrationBean.onStartup
```

![tomcat内嵌](/img/in-post/post-springboot/tomcat内嵌.png)

## DispatcherServlet的装配

核心过程：

```
DispatcherServletRegistrationBean.onStartup->ServletRegistrationBean.addRegistration->
servletConfig.register
```

![DispatcherServlet装配过程](/img/in-post/post-springboot/DispatcherServlet装配过程.png)

