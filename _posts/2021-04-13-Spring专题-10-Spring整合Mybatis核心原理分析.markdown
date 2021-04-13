---
layout:     post
title:      "Spring专题-10.Spring整合Mybatis核心原理分析"
subtitle:   " \"Spring整合Mybatis核心原理分析\""
date:       2021-04-13 14:10:00 +0800
author:     "snowball"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Spring整合Mybatis
    - Spring Core
    - 技术
typora-root-url: ..
---

<!-- “Spring. ” -->

## 1.3.2版本

1. 通过@MapperScan导入了MapperScannerRegistrar类
2. MapperScannerRegistrar类实现了ImportBeanDefinitionRegistrar接口，所以Spring在启动时会调用MapperScannerRegistrar类中的registerBeanDefinitions方法
3. 在registerBeanDefinitions方法中定义了一个ClassPathMapperScanner对象，用来扫描mapper
4. 设置**ClassPathMapperScanner对象可以扫描到接口**，因为在Spring中是不会扫描接口的
5. 同时因为ClassPathMapperScanner中**重写了isCandidateComponent方法**，导致isCandidateComponent只会认为接口是备选者Component
6. 通过利用Spring的扫描后，会把接口扫描出来并且得到对应的BeanDefinition
7. 接下来把扫描得到的BeanDefinition进行修改，把BeanClass修改为MapperFactoryBean，把AutowireMode修改为byType
8. 扫描完成后，Spring就会基于BeanDefinition去创建Bean了，相当于每个Mapper对应一个FactoryBean（单例的）
9. 在MapperFactoryBean中的getObject方法中，调用了getSqlSession()去得到一个sqlSession对象，然后根据对应的Mapper接口生成一个代理对象
10. sqlSession对象是Mybatis中的，一个sqlSession对象需要SqlSessionFactory来产生
11. MapperFactoryBean的**AutowireMode为byType**，所以Spring会自动调用set方法，有两个set方法，一个setSqlSessionFactory，一个setSqlSessionTemplate，而这两个方法执行的前提是根据方法参数类型能找到对应的bean，所以Spring容器中要存在SqlSessionFactory类型的bean或者SqlSessionTemplate类型的bean。
12. 如果你定义的是一个SqlSessionFactory类型的bean，那么最终也会被包装为一个SqlSessionTemplate对象，并且赋值给sqlSession属性
13. 而在SqlSessionTemplate类中就存在一个getMapper方法，这个方法中就会利用SqlSessionFactory来生成一个代理对象。但是可以有多个@Autowired（required=false）,这种情况下，需要Spring从这些构造方法中去自动选择一个构造方法。

## 2.0.5版本

1. 通过@MapperScan导入了MapperScannerRegistrar类
2. MapperScannerRegistrar类实现了ImportBeanDefinitionRegistrar接口，所以Spring在启动时会调用MapperScannerRegistrar类中的registerBeanDefinitions方法
3. **在registerBeanDefinitions方法中生成了一个MapperScannerConfigurer类型的BeanDefinition**
4. **而MapperScannerConfigurer实现了实现了BeanDefinitionRegistryPostProcessor接口，所以Spring在启动过程中时会调用它的postProcessBeanDefinitionRegistry()方法**
5. 在postProcessBeanDefinitionRegistry方法中会生成一个ClassPathMapperScanner对象，然后进行扫描
6. 后续的逻辑和1.3.2版本一样。



带来的好处是，可以不使用@MapperScan注解，而可以直接定义一个Bean，比如：

```
@Bean
public MapperScannerConfigurer mapperScannerConfigurer() {
    MapperScannerConfigurer mapperScannerConfigurer = new MapperScannerConfigurer();
    mapperScannerConfigurer.setBasePackage("com.luban");
    return mapperScannerConfigurer;
}
```

