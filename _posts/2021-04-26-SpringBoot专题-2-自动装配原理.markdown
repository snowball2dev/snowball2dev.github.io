---
layout:     post
title:      "SpringBoot专题-2-自动装配原理"
subtitle:   " \"自动装配原理\""
date:       2021-04-26 14:09:00 +0800
author:     "snowball"
header-img: "img/home-bg-o.jpg"
tags:
    - SpringBoot自动装配
    - 自动配置
    - SpringBoot学习笔记
    - 技术
typora-root-url: ..
---

<!--  “Spring Boot. ” -->

## @SpringBootApplication

![springboot自动装配](/img/in-post/post-springboot/springboot自动装配.png)

```
@Inherited//修饰自定义注解，该自定义注解注解的类，被继承时，子类也会拥有该自定义注解
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
```

#### @SpringBootConfiguration

相当于Configuration，表明启动类也是一个配置类，可以配置@Bean

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;//默认使用CGLIB代理该类
}
```

#### @EnableAutoConfiguration

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    Class<?>[] exclude() default {};

    String[] excludeName() default {};

}
```

#### @AutoConfigurationPackage

保存自动配置类以供之后的使用，比如给`JPA entity`扫描器用来扫描开发人员通过注解`@Entity`定义的`entity`类

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)//手工注册bean
public @interface AutoConfigurationPackage {
```

#### AutoConfigurationImportSelector

```
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 获取注解的属性值
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 从META-INF/spring.factories文件中获取EnableAutoConfiguration所对应的configurations
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 去重
    configurations = removeDuplicates(configurations);
    // 从注解的exclude/excludeName属性中获取排除项
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    // 对于不属于AutoConfiguration的exclude报错
    checkExcludedClasses(configurations, exclusions);
    // 从configurations去除exclusions
    configurations.removeAll(exclusions);
    // 由所有AutoConfigurationImportFilter类的实例再进行一次筛选
    configurations = filter(configurations, autoConfigurationMetadata);
    // 把AutoConfigurationImportEvent绑定在所有AutoConfigurationImportListener子类实例上
    fireAutoConfigurationImportEvents(configurations, exclusions);
    // 返回(configurations, exclusions)组
    return new AutoConfigurationEntry(configurations, exclusions);
}
```

## 自定义starter

1. 新建两个模块：命名规范，springboot自带的spring-boot-starter-xxx，自定义的xxx-spring-boot-starter

​	xxx-spring-boot-autoconfigure：自动配置核心代码

​	xxx-spring-boot-starter：管理依赖，如果不需要将自动配置代码和依赖项管理分离开来，则可以将它们组合到一个模块中。

2. 使用@ConfigurationProperties注入需要的配置

3. @Configuration + @Bean注册需要的bean,使用@EnableConfigurationProperties开启配置注入

4. 利用META-INF/spring.factories导入@Configuration配置类（完全无侵入，但缺乏灵活性）
   1. 启动类使用@Import注解导入@Configuration配置类
   2. `EnableXXX` 自定义注解使用@Import注解导入@Configuration配置类，启动类添加该注解

## Spring  SPI

![SpringSPI](/img/in-post/post-springboot/SpringSPI.png)

## 自动装配

![自动装配](/img/in-post/post-springboot/自动装配.png)