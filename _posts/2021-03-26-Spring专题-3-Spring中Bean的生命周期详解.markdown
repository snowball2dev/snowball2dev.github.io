---
layout:     post
title:      "Spring专题-3.Spring中Bean的生命周期详解"
subtitle:   " \"Spring中Bean的生命周期详解\""
date:       2021-03-26 16:30:00 +0800
author:     "snowball"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Spring Bean
    - Spring Core
    - 技术
typora-root-url: ..
---

<!-- “Spring. ” -->

## Spring中Bean的生命周期详解

Spring最重要的功能就是帮助程序员创建对象（也就是IOC），而启动Spring就是为创建Bean对象做准备，所以我们先明白Spring到底是怎么去创建Bean的，也就是先弄明白Bean的生命周期。



Bean的生命周期就是指：**在Spring中，一个Bean是如何生成的，如何销毁的？**

![Bean的生命周期流程](/img/in-post/post-spring/Bean的生命周期流程.png)

## Bean的生成过程

### 1. 生成BeanDefinition

Spring启动的时候会进行扫描，会先调用

```
Resource[] resources = getResourcePatternResolver().getResources(packageSearchPath);
```

拿到所指定的包路径下的所有文件资源（******.class文件）

然后会遍历每个Resource，为每个Resource生成一个MetadataReader对象，这个对象拥有三个功能：

1. 获取对应的Resource资源
2. 获取Resource对应的class的元数据信息，包括类的名字、是不是接口、是不是一个注解、是不是抽象类、有没有父类，父类的名字，所实现的所有接口的名字，内部类的类名等等。
3. 获取Resource对应的class上的注解信息，当前类上有哪些注解，当前类中有哪些方法上有注解



在生成MetadataReader对象时，会利用**ASM**技术解析class文件，得到类的元数据集信息合注解信息，在这个过程中也会利用ClassLoader去加载注解类（**ClassUtils.getDefaultClassLoader()所获得的类加载器**），但是不会加载本类。



有了MetadataReader对象，就相当于有了当前类的所有信息，但是当前类并没有加载，也是可以理解的，真正在用到这个类的时候才加载。



然后利用MetadataReader对象生成一个ScannedGenericBeanDefinition对象，**注意此时的BeanDefinition对象中的beanClass属性存储的是当前类的名字，而不是class对象**。（beanClass属性的类型是Object，它即可以存储类的名字，也可以存储class对象）

### 2. 合并BeanDefinition

如果某个BeanDefinition存在父BeanDefinition，那么则要进行合并