---
layout:     post
title:      "Spring专题-1.Spring框架核心逻辑"
subtitle:   " \"Spring框架核心逻辑\""
date:       2021-03-21 22:00:00
author:     "snowball"
header-img: "img/post-bg.jpg"
tags:
    - Spring Core
    - Spring
	- Spring学习笔记
---

<!-- > “Spring. ” -->

代码工程地址：
https://gitee.com/snowball2dev/spring-notice/

1. 了解Spring工作的大概流程，Spring启动，扫描，创建非懒加载的单例Bean，填充属性（byType，byName），回调Aware接口，初始化
2. 熟悉BeanDefinition，BeanFactory，Bean等基本概念
3. 熟悉Bean的生命周期，Bean的后置处理器等基本概念，BeanNameAware，InitializingBean
初始化前，初始化后
4. 熟悉常用注解，@Component，@ComponentScan，@Lazy，@Scope，@Autowired