---
layout:     post
title:      "微服务调用组件Feign&Dubbo实战"
subtitle:   " \"微服务调用组件Feign&Dubbo实战\""
date:       2022-01-04 18:30:00 +0800
author:     "snowball"
header-img: "img/home-bg.jpg"
tags:
    - Spring Cloud
    - Feign
    - Dubbo
    - 微服务调用组件
typora-root-url: ..
---

<!--  “微服务组件-Feign&Dubbo实战. ” -->

### 主要内容

#### 1.Fegin的设计架构剖析

思考：JAVA 项目中如何实现接口调用？

**1.Httpclient**

HttpClient 是 Apache Jakarta Common 下的子项目，用来提供高效的、最新的、功能丰富的支持 Http 协
议的客户端编程工具包，并且它支持 HTTP 协议最新版本和建议。HttpClient 相比传统 JDK 自带的
URLConnection，提升了易用性和灵活性，使客户端发送 HTTP 请求变得容易，提高了开发的效率。

**2.Okhttp**
一个处理网络请求的开源项目，是安卓端最火的轻量级框架，由 Square 公司贡献，用于替代
HttpUrlConnection 和 Apache HttpClient。OkHttp 拥有简洁的 API、高效的性能，并支持多种协议
（HTTP/2 和 SPDY）。

**3.HttpURLConnection**
HttpURLConnection 是 Java 的标准类，它继承自 URLConnection，可用于向指定网站发送 GET 请求、
POST 请求。HttpURLConnection 使用比较复杂，不像 HttpClient 那样容易使用。

**4.RestTemplate**
RestTemplate 是 Spring 提供的用于访问 Rest 服务的客户端，RestTemplate 提供了多种便捷访问远程
HTTP 服务的方法，能够大大提高客户端的编写效率。



上面介绍的是最常见的几种调用接口的方法，我们下面要介绍的方法比上面的更简单、方便，它就是Feign。

##### 1. 什么是Feign

Feign是Netflix开发的声明式、模板化的HTTP客户端，其灵感来自Retrofit、JAXRS-2.0以及WebSocket。

Feign可帮助我们更加便捷、优雅地调用HTTP API。

Feign支持多种注解，例如Feign自带的注解或者JAX-RS注解等。Spring Cloud openfeign对Feign进行了增强，使其支持Spring MVC注解，另外还整合了Ribbon和Eureka，从而使得Feign的使用更加方便。

**1.1 优势**

Feign可以做到使用 HTTP 请求远程服务时就像调用本地方法一样的体验，开发者完全感知不到这是远程方
法，更感知不到这是个 HTTP 请求。它像 Dubbo 一样，consumer 直接调用接口方法调用 provider，而不
需要通过常规的 Http Client 构造请求再解析返回数据。它解决了让开发者调用远程接口就跟调用本地方法
一样，无需关注与远程的交互细节，更无需关注分布式环境开发。

**1.2 Feign的设计架构**

如下图：

![Feign架构](/img/in-post/post-micro-service-feign/Feign架构.png)

#### 2.Fegin使用详解

##### 2.1 Ribbon&Feign对比

2.1.1.Ribbon+RestTemplate进行微服务调用

```
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
	return new RestTemplate();
}

//调用方式
String url = "http://order-service/order/findOrderByUserId/"+id;
R result = restTemplate.getForObject(url,R.class);
```

2.1.2.Feign进行微服务调用

```
@FeignClient(value = "order-service", path = "order")
public interface OrderFeignApi {

    @RequestMapping("/findOrderByUserId/{userId}")
    Resp findOrderByUserId(@PathVariable("userId") String userId);

}

@Autowired
OrderFeignApi orderFeignApi;

//OpenFeign调用
Resp result = orderFeignApi.findOrderByUserId(userId);
```

feign使用起来更加简单，易维护

##### 2.2 Feign单独使用

案例项目工程地址：https://gitee.com/snowball2dev/spring-cloud-alibaba-notice

2.2.1.引入依赖

```
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-core</artifactId>
    <version>10.10.1</version>
</dependency>

<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-jackson</artifactId>
    <version>10.10.1</version>
</dependency>
```

2.2.2.编写Feign接口

```
public interface RemoteService {

    @Headers({/*"Content‐Type: application/json",*/"Accept: application/json"})
    @RequestLine("GET /order/findOrderByUserId/{userId}")
    Resp findOrderByUserId(@Param("userId") Integer userId);

}
```

2.2.3.构建代理对象调用

```
 //基于json
 // encoder指定对象编码方式，decoder指定对象解码方式
 RemoteService remoteService = Feign.builder().encoder(new JacksonEncoder())
     .decoder(new JacksonDecoder())
     .options(new Request.Options(1000, 3500))
     .retryer(new Retryer.Default(5000, 5000, 3))
     .target(RemoteService.class, "http://localhost:8085");

Resp resp = remoteService.findOrderByUserId(1);
System.out.println(resp);
```



#### 3.Fegin整合Spring Cloud Alibaba实战

3.1.引入依赖

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

3.2.编写调用接口+@FeignClient注解

```
@FeignClient(value = "order-service", path = "order")
public interface OrderFeignApi {

    @RequestMapping("/findOrderByUserId/{userId}")
    Resp findOrderByUserId(@PathVariable("userId") String userId);

}
```

3.3.调用端在启动类上添加@EnableFeignClients注解

```
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class UserApplication 
```

3.4.发起调用，像调用本地方式一样调用远程服务

```
@Autowired
OrderFeignApi orderFeignApi;

//OpenFeign调用
Resp result = orderFeignApi.findOrderByUserId(userId);
```

提示： Feign 的继承特性可以让服务的接口定义单独抽出来，作为公共的依赖，以方便使用。



#### 4.Fegin自定义相关配置使用详解

Feign 提供了很多的扩展机制，让用户可以更加灵活的使用。

##### 4.1 日志配置

有时候我们遇到 Bug，比如接口调用失败、参数没收到等问题，或者想看看调用性能，就需要配置 Feign 的
日志了，以此让 Feign 把请求信息输出来。

**4.1.1.定义一个配置类**，指定日志级别

```
// 此处配置@Configuration注解就会全局生效，如果想指定对应微服务生效，就不能配置
public class FeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel(){
        return Logger.Level.FULL;
    }
}
```

通过源码可以看到日志等级有 4 种，分别是：

- NONE【性能最佳，适用于生产】：不记录任何日志（默认值）。
- BASIC【适用于生产环境追踪问题】：仅记录请求方法、URL、响应状态代码以及执行时间。
- EADERS：记录BASIC级别的基础上，记录请求和响应的header。
- FULL【比较适用于开发及测试环境定位问题】：记录请求和响应的header、body和元数据。

**4.1.2.局部配置**

让调用的微服务生效，在@FeignClient 注解中指定使用的配置类

```
@FeignClient(value = "order-service", path = "order", configuration = FeignConfig.class)
public interface OrderFeignApi {

    @RequestMapping("/findOrderByUserId/{userId}")
    Resp findOrderByUserId(@PathVariable("userId") String userId);
}
```

**4.1.3.yml配置文件**

配置执行 Client 的日志级别才能正常输出日志，格式是"logging.level.feign接口包路径=debug"

```
logging:
  level:
    com.funny.user.feign: debug
    
feign:
	client:
		config:
			order-service: #对应微服务
				loggerLevel: FULL
```

补充：局部配置可以在yml中配置

对应属性配置类：org.springframework.cloud.openfeign.FeignClientProperties.FeignClientConfiguration

##### 4.2 契约配置

Spring Cloud 在 Feign 的基础上做了扩展，可以让 Feign 支持 Spring MVC 的注解来调用。原生的Feign 是不支持 Spring MVC 注解的，如果你想在 Spring Cloud 中使用原生的注解方式来定义客户端也是可以的，通过配置契约来改变这个配置，Spring Cloud 中默认的是 SpringMvcContract。**这块实际开发中基本采用 SpringMvcContract默认配置，自定义Contract了解即可**

**4.2.1.修改契约配置，支持Feign原生的注解**

```
@Bean
public Contract feignContract() {
	return new Contract.Default();
}
```

**注意：修改契约配置后，OrderFeignService 不再支持springmvc的注解，需要使用Feign原生的注解**

**4.2.2.OrderFeignApi中配置使用Feign原生的注解**

```
@FeignClient(value = "order-service", path = "order", configuration = FeignConfig.class)
public interface OrderFeignApi {

    @RequestLine("GET /findOrderByUserId/{userId}")
    Resp findOrderByUserId(@Param("userId") String userId);
    
}
```

也可以通过yml配置契约

```
feign:
  client:
    config:
      order-service:
        loggerLevel: FULL
        contract: feign.Contract.Default #指定Feign原生注解契约配置
```

##### 4.3 通过拦截器实现认证

通常我们调用的接口都是有权限控制的，很多时候可能认证的值是通过参数去传递的，还有就是通过请求头
去传递认证信息，比如 Basic 认证方式。

Feign 中我们可以直接配置 Basic 认证

```
@Bean
public BasicAuthRequestInterceptor basicAuthRequestInterceptor() {
	return new BasicAuthRequestInterceptor("test", "123456");
}
```

**扩展点： feign.RequestInterceptor**

每次 feign 发起http调用之前，会去执行拦截器中的逻辑。

```
public interface RequestInterceptor {

  /**
   * Called for every request. Add data using methods on the supplied {@link RequestTemplate}.
   */
  void apply(RequestTemplate template);
}
```

使用场景:
1. 统一添加 header 信息；
2. 对 body 中的信息做修改或替换；

```
public class FeignAuthRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 业务逻辑
        String access_token = UUID.randomUUID().toString();
        template.header("Authorization", access_token);
    }
}

@Bean
public FeignAuthRequestInterceptor feignAuthRequestInterceptor(){
	return new FeignAuthRequestInterceptor();
}
```

测试结果中，请求过程中拦截器添加了请求头，order-service端可以通过 @RequestHeader获取请求参数

##### 4.4 超时时间配置

通过 Options 可以配置连接超时时间和读取超时时间，Options 的第一个参数是连接的超时时间（ms），默认值是 2s；第二个是请求处理的超时时间（ms），默认值是 5s。

**全局配置**：

```
@Bean
public Request.Options options() {
	return new Request.Options(5000, 10000);
}
```

**yml中配置方式：**

```
feign:
  client:
    config:
      order-service:
        # 连接超时时间，默认2s
        connectTimeout: 5000
        # 请求处理超时时间，默认5s
        readTimeout: 10000
```

补充说明： Feign的底层用的是Ribbon，但超时时间以Feign配置为准

##### 4.5 客户端组件配置

Feign 中默认使用 JDK 原生的 URLConnection 发送 HTTP 请求，我们可以集成别的组件来替换URLConnection，比如 Apache HttpClient，OkHttp。

Feign发起调用真正执行逻辑：**feign.Client#execute （扩展点）**

```
@Override
public Response execute(Request request, Options options) throws IOException {
      HttpURLConnection connection = convertAndSend(request, options);
      return convertResponse(connection, request);
}
```

**4.5.1 配置Apache HttpClient**

```
 <!--         Apache HttpClient -->
 <dependency>
 	<groupId>org.apache.httpcomponents</groupId>
 	<artifactId>httpclient</artifactId>
 	<version>4.5.7</version>
 </dependency>

<dependency>
	<groupId>io.github.openfeign</groupId>
	<artifactId>feign-httpclient</artifactId>
	<version>10.10.1</version>
</dependency>
```

然后修改yml配置，将 Feign 的 Apache HttpClient启用 ：

```
feign:
  httpclient:
    enabled: true
```

关于配置可参考源码： org.springframework.cloud.openfeign.FeignAutoConfiguration

```
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(ApacheHttpClient.class)
@ConditionalOnMissingClass("com.netflix.loadbalancer.ILoadBalancer")
@ConditionalOnMissingBean(CloseableHttpClient.class)
@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
protected static class HttpClientFeignConfiguration {
```

测试：调用会进入feign.httpclient.ApacheHttpClient#execute

**4.5.2 配置OkHttp**

引入依赖

```
<dependency>
	<groupId>io.github.openfeign</groupId>
	<artifactId>feign‐okhttp</artifactId>
</dependency>
```

然后修改yml配置，将 Feign 的 HttpClient 禁用，启用 OkHttp，配置如下：

```
feign:
	httpclient:
		enabled: false
	okhttp:
		enabled: true
```

测试：调用会进入feign.okhttp.OkHttpClient#execute

##### 4.6 GZIP 压缩配置

开启压缩可以有效节约网络资源，提升接口性能，我们可以配置 GZIP 来压缩数据：

```
feign:
  # 配置 GZIP 来压缩数据
  compression:
    request:
      enabled: true
      # 配置压缩的类型
      mime-types: text/xml,application/xml,application/json
      # 最小压缩值
      min-request-size: 2048
    response:
      enabled: true
```

**注意：只有当 Feign 的 Http Client 不是 okhttp3 的时候，压缩才会生效，配置源码在**
**FeignAcceptGzipEncodingAutoConfiguration**

核心代码就是 @ConditionalOnMissingBean（type="okhttp3.OkHttpClient"），表示 Spring
BeanFactory 中不包含指定的 bean 时条件匹配，也就是没有启用 okhttp3 时才会进行压缩配置。

##### 4.7 编码器解码器配置

Feign 中提供了自定义的编码解码器设置，同时也提供了多种编码器的实现，比如 Gson、Jaxb、Jackson。
我们可以用不同的编码解码器来处理数据的传输。如果你想传输 XML 格式的数据，可以自定义 XML 编码解
码器来实现获取使用官方提供的 Jaxb。

**扩展点：Encoder & Decoder**

```
public interface Encoder {
	void encode(Object object, Type bodyType, RequestTemplate template) throws EncodeException;
}

public interface Decoder {
	Object decode(Response response, Type type) throws IOException, DecodeException, FeignException;
}
```

Java配置方式

配置编码解码器只需要在 Feign 的配置类中注册 Decoder 和 Encoder 这两个类即可:

```
@Bean
public Decoder decoder() {
	return new JacksonDecoder();
}

@Bean
public Encoder encoder() {
	return new JacksonEncoder();
}
```

yml配置方式

```
feign:
  client:
    config:
      order-service:
        # 连接超时时间，默认2s
        connectTimeout: 5000
        # 请求处理超时时间，默认5s
        readTimeout: 10000
        loggerLevel: FULL
        # 配置编解码器
        encoder: feign.jackson.JacksonEncoder
        decoder: feign.jackson.JacksonDecoder
```

#### 5.Spring Cloud Alibaba整合Dubbo实战

##### 1.provider端配置

**引入依赖**

```
<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring‐cloud‐starter‐dubbo</artifactId>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

**application.yml**

```
dubbo:
  scan:
    base-packages: com.funny.dubbo.user.service
  application:
    name: ${spring.application.name}
  protocol:
    name: dubbo
    # dubbo 协议端口（ ‐1 表示自增端口，从 20880 开始）
    port: -1
  # registry:
  # #挂载到 Spring Cloud 注册中心 高版本可选
  # address: spring‐cloud://127.0.0.1:8848


spring:
  application:
    name: spring_cloud_dubbo_provider_user
  main:
    allow-bean-definition-overriding: true
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
```

**服务实现类上配置@DubboService暴露服务**

```
@DubboService
public class UserServiceImpl implements UserService  {

    @Override
    public List<User> list() {
        List<User> list = new ArrayList<>();
        list.add(new User(1, "test"));
        list.add(new User(1, "admin"));
        return list;
    }

    @Override
    public User getById(Integer id) {
        return new User(id, "test");
    }
}
```

##### 2.consumer端配置

**引入依赖**

```
  <dependency>
  	<groupId>org.springframework.boot</groupId>
  	<artifactId>spring-boot-starter-web</artifactId>
  </dependency>

<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-dubbo</artifactId>
	<version>2.2.5.RELEASE</version>
</dependency>

<dependency>
	<groupId>com.alibaba.cloud</groupId>
	<artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

<dependency>
	<groupId>com.funny</groupId>
	<artifactId>spring-cloud-dubbo-api</artifactId>
</dependency>
```

**application.yml**

```
dubbo:
  cloud:
    # 指定需要订阅的服务提供方，默认值*，会订阅所有服务，不建议使用
    subscribed‐services: spring_cloud_dubbo_provider_user
  application:
    name: ${spring.application.name}
  protocol:
    name: dubbo
    # dubbo 协议端口（ ‐1 表示自增端口，从 20880 开始）
    port: -1
  # registry:
  # #挂载到 Spring Cloud 注册中心 高版本可选
  # address: spring‐cloud://127.0.0.1:8848


spring:
  application:
    name: spring_cloud_dubbo_consumer_user
  main:
    allow-bean-definition-overriding: true
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
```

当应用使用属性dubbo.cloud.subscribed­services为默认值时，日志中将会输出警告。

**服务消费方通过@DubboReference引入服务**

```
@RestController
@RequestMapping("/user")
public class UserController {

    @DubboReference
    UserService userService;

    @RequestMapping("/list")
    public List<User> list(){
        return userService.list();
    }
}
```

#### 6.如何实现Feign到Dubbo的无缝迁移

Dubbo Spring Cloud 提供了方案，即 @DubboTransported 注解，支持在类，方法，属性上使用。能够帮助服务消费端的 Spring Cloud Open Feign 接口以及 @LoadBalanced RestTemplate Bean 底层走 Dubbo 调用（可切换 Dubbo 支持的协议），而服务提供方则只需在原有 @RestController 类上追加 Dubbo @Servce 注解（需要抽取接口）即可，换言之，在不调整 Feign 接口以及 RestTemplate URL 的前提下，实现无缝迁移。

**服务消费端引入依赖，增加OpenFeign**

```
<!--    openfeign    -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

不建议同时使用feign和dubbo，因为使用feign需要provider提供web接口访问，dubbo netty也会单独开启一个接口，相当于启动了2个端口

**修改服务提供者**

```
@RestController
@RequestMapping("/user")
@DubboService
public class UserServiceImpl implements UserService  {

    @Override
    @RequestMapping("/list")
    public List<User> list() {
        List<User> list = new ArrayList<>();
        list.add(new User(1, "test"));
        list.add(new User(2, "admin"));
        return list;
    }

    @Override
    @RequestMapping("/getById/{id}")
    public User getById(@PathVariable("id")Integer id) {
        return new User(id, "test");
    }
}
```

**feign的实现，启动类上添加@EnableFeignClients**

```
@SpringBootApplication
@EnableFeignClients
public class ConsumerUserApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerUserApplication.class);
    }
}
```

**feign接口添加 @DubboTransported 注解**

```
@FeignClient(value = "spring-cloud-dubbo-provider-user", path = "user")
public interface UserFeignApi {

  @RequestMapping("/list")
  List<User> list();

  @RequestMapping("/getById/{id}")
  User getById(@PathVariable("id") Integer id);
}

@DubboTransported(protocol = "dubbo")
@FeignClient(value = "spring-cloud-dubbo-provider-user", path = "/user")
public interface UserDubboFeignApi {

  @RequestMapping("/list")
  List<User> list();

  @RequestMapping("/getById/{id}")
  User getById(@PathVariable("id") Integer id);

}
```

**调用对象添加 @DubboTransported 注解**

```
@RestController
@RequestMapping("/user")
public class UserController {

//    @DubboReference
//    UserService userService;

    //使用httpUrlConnection，断点Client.execute
    @Autowired
    //添加DubboTransported注解，协议是否生效？无效
    @DubboTransported
    UserFeignApi userService;

    //使用dubbo Invoker
    //@DubboTransported(protocol = "dubbo")
    //添加DubboTransported注解，协议是否生效？无效
    @Autowired
    UserDubboFeignApi userDubboFeignApi;

    @RequestMapping("/list")
    public List<User> list() {
        return userService.list();
    }

    @RequestMapping("/list2")
    public List<User> list2() {
        return userDubboFeignApi.list();
    }

    @RequestMapping("/getById/{id}")
    public User getById(@PathVariable Integer id) {
        return userService.getById(id);
    }

    @Autowired
    RestTemplate restTemplate;

    @Bean
    @LoadBalanced
    @DubboTransported
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @RequestMapping("/list3")
    public List<User> list3() {
        String url = "http://spring-cloud-dubbo-provider-user/user/list";
        return restTemplate.getForObject(url, List.class);
    }

}
```
