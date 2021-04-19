---
layout:     post
title:      "SpringMVC专题-2.SpringMVC框架核心逻辑"
subtitle:   " \"SpringMVC框架核心逻辑\""
date:       2021-04-19 17:06:00 +0800
author:     "snowball"
header-img: "img/home-bg-o.jpg"
tags:
    - SpringMVC
    - Controller
    - Spring学习笔记
    - 技术
typora-root-url: ..
---

<!--  “Spring MVC. ” -->

## 框架功能分析

SpringMVC模块：

**处理器映射器**（HandlerMapping）：做业务处理的组件，类、方法跟url有映射关系

功能过程分析：使用内嵌tomcat作为servlet容器，打开IOC扫描，后置处理器，IOC取出处理器   

处理器类型：

@Controller、普通的bean、Map<url，Object处理器>、springmvc注解：@RequestMapping(url)方法



处理器映射器对应的执行机制：

 @Controller @RequestMapping(url)方法，反射执行   

 Map<url,method>，反射

 servlet.service()，映射路径  /  if  else

 Controller.handleRequest，(Controller)controller.handleRequest

 HttpRequestHandler.handleRequest，(HttpRequestHandler)handler.handleRequest

 /beanName  --->  bean处理器  </id,处理器>

 

**处理器适配器**（HandlerAdapter）：跟映射器一一对应，处理器是由适配器执行的

统一的接口  

判断适配是否成功   Object处理器  instanseOf  Controller

执行处理逻辑    (Controller)处理器 调用 handleRequest



**Tomcat  SPI：服务发现机制**

特定的目录(目录路径是定死的，相对路径  META-INF/service +  接口全限定名)下：实现类(字符串)

自定义类加载器：隔离应用

webapp目录：放多个war



**JDK  SPI**

自定义类加载器：打破双亲委派



**Spring SPI**

Servlet3.0的核心接口：ServletContainerInitializers接口  调用 onStartup

springmvc启动：

1、启动tomcat

2、完成IOC创建，初始化，扫描

tomcat.sh -->  webapp目录  war（spring）  servlet规范

ServletContainerInitializers接口实现类，任意jar

ServletContainerInitializers接口实现类放入standardContext全局变量initializers

## 执行流程

### Tomcat SPI

![TomcatSPI](/img/in-post/post-springmvc/TomcatSPI.png)

### DispatcherServlet

![Dispatcher](/img/in-post/post-springmvc/Dispatcher.png)