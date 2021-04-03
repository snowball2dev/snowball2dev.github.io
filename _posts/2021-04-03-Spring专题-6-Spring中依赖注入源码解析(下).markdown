---
layout:     post
title:      "Spring专题-6.Spring中依赖注入源码解析(下)"
subtitle:   " \"Spring中依赖注入源码解析(下)\""
date:       2021-04-03 11:10:00 +0800
author:     "snowball"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - Spring IOC
    - 依赖注入
    - Spring Core
    - 技术
typora-root-url: ..
---

<!-- “Spring. ” -->

上篇文章写了Spring中的自动注入(byName,byType)和@Autowired注解的工作原理以及源码分析，这篇文章分析剩下的核心的方法：

AbstractAutowireCapableBeanFactory类

```
@Nullable
Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
        @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException;
```

该方法表示，传入一个依赖描述（DependencyDescriptor），该方法会根据该依赖描述从BeanFactory中找出对应的唯一的一个Bean对象。

下面来分析一下**DefaultListableBeanFactory**中**resolveDependency()**方法的具体实现，大概过程见下图

![Spring中根据Type找Bean的流程](/img/in-post/post-spring/Spring中根据Type找Bean的流程.png)

## findAutowireCandidates()实现

找出BeanFactory中类型为type的所有的Bean的名字，注意是名字，而不是Bean对象，因为我们可以根据BeanDefinition就能判断和当前type是不是匹配

把resolvableDependencies中key为type的对象找出来并添加到result中

遍历根据type找出的beanName，判断当前beanName对应的Bean是不是能够被自动注入

先判断beanName对应的BeanDefinition中的autowireCandidate属性，如果为false，表示不能用来进行自动注入，如果为true则继续进行判断

判断当前type是不是泛型，如果是泛型是会把容器中所有的beanName找出来的，如果是这种情况，那么在这一步中就要获取到泛型的真正类型，然后进行匹配，如果当前beanName和当前泛型对应的真实类型匹配，那么则继续判断

如果当前DependencyDescriptor上存在@Qualifier注解，那么则要判断当前beanName上是否定义了Qualifier，并且是否和当前DependencyDescriptor上的Qualifier相等，相等则匹配

经过上述验证之后，当前beanName才能成为一个可注入的，添加到result中

DefaultListableBeanFactory类:

```
// 根据requiredType寻找自动匹配的候选者bean
protected Map<String, Object> findAutowireCandidates(
		@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

	// 根据requiredType找到candidateNames，表示根据type找到了候选beanNames
	// Ancestors是祖先的意思，所以这个方法是去当前beanfactory以及祖先beanfactory中去找类型为requiredType的bean的名字
	String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
			this, requiredType, true, descriptor.isEager());

	Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);

	// 先从resolvableDependencies获取，这里是通过beanFactory.resolveDependency()注册
	for (Map.Entry<Class<?>, Object> classObjectEntry : this.resolvableDependencies.entrySet()) {
		...
	}

	// 对候选bean进行过滤，首先候选者不是自己，然后候选者是支持自动注入给其他bean的
	for (String candidate : candidateNames) {  // beanName orderSer1 order2 oser
		// isAutowireCandidate方法中会去判断候选者是否和descriptor匹配
		if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
			addCandidateEntry(result, candidate, descriptor, requiredType);
		}
	}

	// 如果过滤后的结果为空，再判断自己是否是候选者，是就添加到结果map
	if (result.isEmpty()) {
		...
	}
	return result;
}
```

### isAutowireCandidate(candidate, descriptor)

判断候选者是否和descriptor匹配

```
protected boolean isAutowireCandidate(String beanName, RootBeanDefinition mbd,
			DependencyDescriptor descriptor, AutowireCandidateResolver resolver) {

		String beanDefinitionName = BeanFactoryUtils.transformedBeanName(beanName);
		// 加载mbd中所指定的类
		resolveBeanClass(mbd, beanDefinitionName);
		if (mbd.isFactoryMethodUnique && mbd.factoryMethodToIntrospect == null) {
			new ConstructorResolver(this).resolveFactoryMethodIfPossible(mbd);
		}
		// 通过候选者解析器resolver检查BeanDefinitionHolder和descriptor匹配
		return resolver.isAutowireCandidate(
				new BeanDefinitionHolder(mbd, beanName, getAliases(beanDefinitionName)), descriptor);
}

AutowireCandidateResolver在Spring启动过程中默认设置为ContextAnnotationAutowireCandidateResolver
	->QualifierAnnotationAutowireCandidateResolver
		->GenericTypeAwareAutowireCandidateResolver
			->SimpleAutowireCandidateResolver
```

AutowireCandidateResolver会依次从父类开始检查是否匹配，如果不匹配直接返回，表示不通过筛选。

大概过程见下图：

![依赖注入流程](/img/in-post/post-spring/依赖注入流程.png)

## Qualifier应用

@Qualifier可以用来让程序员明确指定想要指定哪个bean，那有程序员就会想问，它和@Autowired和@Resource的区别是什么？

假设有如下bean定义：

```
 <bean id="user0" class="com.luban.entity.User">
        <property name="name" value="user0000"/>
    </bean>

    <bean id="user1" class="com.luban.entity.User" >
        <property name="name" value="user1111"/>
    </bean>

    <bean name="userService" class="com.luban.service.UserService" autowire="constructor"/>
```

user0和user1的类型都是user，UserService的通过构造方法自动注入。

```
public class UserService {

    private User user;

    public UserService(User user11) {
        this.user = user11;
    }

    public void test() {
        System.out.println(user.getName());
    }
}
```

main方法：

```
  public static void main(String[] args) {
        ClassPathXmlApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");

        UserService userService = applicationContext.getBean("userService", UserService.class);
        userService.test();
    }
```

现在运行main方法，会启动Spring，会报错：

```
Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.luban.entity.User' available: expected single matching bean but found 2: user0,user1
    at org.springframework.beans.factory.config.DependencyDescriptor.resolveNotUnique(DependencyDescriptor.java:223)
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.doResolveDependency(DefaultListableBeanFactory.java:1324)
    at org.springframework.beans.factory.support.DefaultListableBeanFactory.resolveDependency(DefaultListableBeanFactory.java:1255)
    at org.springframework.beans.factory.support.ConstructorResolver.resolveAutowiredArgument(ConstructorResolver.java:939)
    at org.springframework.beans.factory.support.ConstructorResolver.createArgumentArray(ConstructorResolver.java:842)
    ... 15 more
```

很明显，因为UserService会根据构造方法自动注入，会先根据构造方法参数类型User是找bean，找到了两个，一个user0, 一个user1，然后会根据构造方法参数名user11去进行过滤，但是过滤不出来，最终还是找到了两个，所以没法进行自动注入，所以报错。



怎么办？

要么给中某个bean设置primary为true，表示找到多个时**默认**用这个。或者给某个bean设置autowire-candidate为false，表示这个bean**不作为**自动注入的候选者。还有一种就是@Qualifier, 首先，给两个bean分别定义一个qualifier：

```
  <bean id="user0" class="com.luban.entity.User" autowire-candidate="false">
        <property name="name" value="user0000"/>
        <qualifier value="user00"/>
    </bean>

    <bean id="user1" class="com.luban.entity.User" >
        <property name="name" value="user1111"/>
        <qualifier value="user11"/>
    </bean>
```

同时在UserService中使用@Qualifier注解：

```
public UserService(@Qualifier("user00") User user11) {
        this.user = user11;
    }
```

还需要做一件事情，因为我们是使用的XML的方式使用Spring，所以还需要添加：

```
<context:annotation-config/>
```

这样@Qualifier注解才会生效，这样配置之后UserService中的user属性就会被赋值为@Qualifier("user00")所对应的user0这个bean。

@Qualifier注解是在使用端去指定想要使用的bean，而autowire-candidate和primary是在配置端。

@Qualifier的另外一种用法

定义一个：

```
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier
public @interface Genre {

    String value();
}
```

修改bean的定义

```
    <bean id="user0" class="com.luban.entity.User">
        <property name="name" value="user0000"/>
        <qualifier type="com.luban.annotation.Genre" value="user00"/>
    </bean>

    <bean id="user1" class="com.luban.entity.User" >
        <property name="name" value="user1111"/>
        <qualifier type="com.luban.annotation.Genre" value="user11"/>
    </bean>
```

UserService中使用Genre注解：

```
public class UserService {

    private User user;

    public UserService(@Genre("user00") User user11) {
        this.user = user11;
    }

    public void test() {
        System.out.println(user.getName());
    }
}
```

这种用法相当于自定义@Qualifier注解，使得可以处理更多的情况。

同样，还可以这么做，定义两个注解：

```
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier("random")
public @interface Random {

}
```

```
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Qualifier("roundRobin")
public @interface RoundRobin {
    
}
```

注意，是在自定义的Qualifier注解上指定了名字，这样就是可以直接使用自定义注解，而不用指定名字了。比如先修改bean的定义：

```
<bean id="user0" class="com.luban.entity.User">
        <property name="name" value="user0000"/>
        <qualifier type="com.luban.annotation.Random"/>
    </bean>

    <bean id="user1" class="com.luban.entity.User" >
        <property name="name" value="user1111"/>
        <qualifier type="com.luban.annotation.RoundRobin"/>
    </bean>
```

UserService中就可以这么用：

```
  public UserService(@RoundRobin User user11) {
        this.user = user11;
    }
```

## 自己注入自己

定义一个类，类里面提供了一个构造方法，用来设置name属性

```
public class UserService { 

    private String name;

    public UserService(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    @Autowired
    private UserService userService;


    public void test() {
        System.out.println(userService.getName());
    }

}
```

然后针对UserService定义两个Bean:

```
 	@Bean
    public UserService userService1() {
        return new UserService("userService1");
    }

    @Bean
    public UserService userService() {
        return new UserService("userService");
    }
```

按照正常逻辑来说，对于注入点：

```
@Autowired
private UserService userService;
```

会先根据UserService类型去找Bean，找到两个，然后根据属性名字“userService”找到一个beanName为userService的Bean，但是我们直接运行Spring，会发现注入的是“userService1”的那个Bean。

这是因为Spring中进行了控制，尽量“**自己不注入自己**”。

## @Resource注解底层原理

源码过程见下图：

![@Resource注解底层工作原理](/img/in-post/post-spring/@Resource注解底层工作原理.png)