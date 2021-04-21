---
layout:     post
title:      "SpringMVC专题-4.HandlerMapping和HandlerAdapter"
subtitle:   " \"HandlerMapping和HandlerAdapter\""
date:       2021-04-21 09:30:00 +0800
author:     "snowball"
header-img: "img/home-bg-o.jpg"
tags:
    - HandlerAdapter
    - HandlerMapping
    - Spring学习笔记
    - 技术
typora-root-url: ..
---

<!--  “Spring MVC. ” -->

SpringMVC框架支持4种请求处理器：

1. @Controller+@RequestMapping （类里面支持多个请求路径的对应方法）
2. 实现HttpRequestHandler接口
3. 实现Controller接口
4. 继承HttpServlet，web.xml配置或者Bean配置ServletRegistrationBean

## **HandlerMapping**

默认情况下，Spring MVC会加载在当前系统中所有实现了HandlerMapping接口的bean，再进行按优先级排序。如果只期望Spring MVC只加载指定的HandlerMapping，可以修改web.xml中的DispatcherServlet的初始化参数，将detectAllHandlerMappings的值设置为false。这样，Spring MVC就只会查找名为“handlerMapping”的bean，并作为当前系统的唯一的HandlerMapping。所以在DispatcherServlet的initHandlerMappings()方法中，优先判断detectAllHandlerMappings的值，如果没有定义HandlerMapping的话，Spring MVC就会按照DispatcherServlet.properties所定义的内容来加载默认的HandlerMapping。

- SimpleUrlHandlerMapping 支持映射bean实例和映射bean名称，需要手工维护urlmap，通过key指定访问路径
- BeanNameUrlHandlerMapping 支持映射bean的name属性值，扫描以”/“开头的beanname，通过id指定访问路径
- RequestMappingHandlerMapping  支持@Controller和@RequestMapping注解，通过注解定义访问路径

作用是将请求映射到处理程序，以及预处理和处理后的拦截器列表，映射是基于一些标准的，其中的细节因不同的实现而不相同。这是官方文档上一段描述，该接口只有一个方法getHandler(request)，返回一个HandlerExecutionChain对象

```
public interface HandlerMapping {
    // 返回请求的一个处理程序handler和拦截器interceptors
    @Nullable
    HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
```

initHandlerMappings

```
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;
    //默认为true，可通过DispatcherServlet的init-param参数进行设置
    if (this.detectAllHandlerMappings) {
        //在ApplicationContext中找到所有的handlerMapping, 包括父级上下文
        Map<String, HandlerMapping> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
           //排序，可通过指定order属性进行设置，order的值为int型，数越小优先级越高
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    } else {
        try {
            //从ApplicationContext中取id（或name）="handlerMapping"的bean,此时为空
            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
            // 将hm转换成list，并赋值给属性handlerMappings
            this.handlerMappings = Collections.singletonList(hm);
        } catch (NoSuchBeanDefinitionException ex) {
        }
    }

    //如果没有自定义则使用默认的handlerMappings
    //默认的HandlerMapping在DispatcherServlet.properties属性文件中定义，
    // 该文件是在DispatcherServlet的static静态代码块中加载的
    // 默认的是：BeanNameUrlHandlerMapping和RequestMappingHandlerMapping
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);//默认策略
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerMappings declared for servlet '" + getServletName() +
                         "': using default strategies from DispatcherServlet.properties");
        }
    }
}
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
   String key = strategyInterface.getName();//HandlerMappings
    //获取HandlerMappings为key的value值,defaultStrategies在static块中读取DispatcherServlet.properties
   String value = defaultStrategies.getProperty(key);
   if (value != null) {
       //逗号，分割成数组
      String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
      List<T> strategies = new ArrayList<>(classNames.length);
      for (String className : classNames) {
         try {
             //反射加载并存储strategies
            Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
             //通过容器创建bean 触发后置处理器    
            Object strategy = createDefaultStrategy(context, clazz);
            strategies.add((T) strategy);
         }
         。。。
      }
      return strategies;
   }
   else {
      return new LinkedList<>();
   }
}

protected Object createDefaultStrategy(ApplicationContext context, Class<?> clazz) {
    //通过容器创建bean
    return context.getAutowireCapableBeanFactory().createBean(clazz);
}
```

DispatcherServlet.properties中的三个HandlerMapping

### AbstractHandlerMapping

```
protected void initApplicationContext() throws BeansException {
    // 提供给子类去重写的，不过Spring并未去实现,提供扩展
    extendInterceptors(this.interceptors);
    // 加载拦截器
    detectMappedInterceptors(this.adaptedInterceptors);
    // 归并拦截器
    initInterceptors();
}

/**
 * 空实现
 */
protected void extendInterceptors(List<Object> interceptors) {
}

/**
 * 从上下文中加载MappedInterceptor类型的拦截器，比如我们在配置文件中使用
 * <mvc:interceptors></mvc:interceptors>
 * 标签配置的拦截器
 */
protected void detectMappedInterceptors(List<HandlerInterceptor> mappedInterceptors) {
    mappedInterceptors.addAll(
            BeanFactoryUtils.beansOfTypeIncludingAncestors(
                    obtainApplicationContext(), MappedInterceptor.class, true, false).values());
}

/**
 * 合并拦截器，即将<mvc:interceptors></mvc:interceptors>中的拦截器与HandlerMapping中通过属性interceptors设置的拦截器进行合并
 */
protected void initInterceptors() {
    if (!this.interceptors.isEmpty()) {
        for (int i = 0; i < this.interceptors.size(); i++) {
            Object interceptor = this.interceptors.get(i);
            if (interceptor == null) {
                throw new IllegalArgumentException("Entry number " + i + " in interceptors array is null");
            }
            // 适配后加入adaptedInterceptors
            this.adaptedInterceptors.add(adaptInterceptor(interceptor));
        }
    }
}

/**
 * 适配HandlerInterceptor和WebRequestInterceptor
 */
protected HandlerInterceptor adaptInterceptor(Object interceptor) {
    if (interceptor instanceof HandlerInterceptor) {
        return (HandlerInterceptor) interceptor;
    } else if (interceptor instanceof WebRequestInterceptor) {
        return new WebRequestHandlerInterceptorAdapter((WebRequestInterceptor) interceptor);
    } else {
        throw new IllegalArgumentException("Interceptor type not supported: " +                                                                                 interceptor.getClass().getName());
    }
}

/**
 * 返回请求处理的HandlerExecutionChain，从AbstractHandlerMapping中的adaptedInterceptors和mappedInterceptors属性中获取
 */
@Override
@Nullable
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // getHandlerInternal()为抽象方法，具体需子类实现
    Object handler = getHandlerInternal(request);
    if (handler == null) {
        handler = getDefaultHandler();
    }
    if (handler == null) {
        return null;
    }
    // Bean name or resolved handler?
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }
    
    // 将请求处理器封装为HandlerExectionChain
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    // 对跨域的处理
    if (CorsUtils.isCorsRequest(request)) {
        CorsConfiguration globalConfig = this.globalCorsConfigSource.getCorsConfiguration(request);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        CorsConfiguration config = (globalConfig != null ? globalConfig.combine(handlerConfig) : handlerConfig);
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }
    return executionChain;
}

/**
 * 钩子函数，需子类实现
 */
@Nullable
protected abstract Object getHandlerInternal(HttpServletRequest request) throws Exception;

/**
 * 构建handler处理器的HandlerExecutionChain，包括拦截器
 */
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
            (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));

    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request);
    // 迭代添加拦截器
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
        // 如果拦截器是MappedInterceptor，判断是否对该handler进行拦截，是的情况下添加
        if (interceptor instanceof MappedInterceptor) {
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        } else { // HandlerInterceptor直接添加，即通过HandingMapping属性配置的拦截器
            chain.addInterceptor(interceptor);
        }
    }
    return chain;
}
```

### RequestMappingHandlerMapping

处理注解@RequestMapping及@Controller

- 实现InitializingBean接口，增加了bean初始化的能力，也就是说在bean初始化时可以做一些控制
- 实现EmbeddedValueResolverAware接口，即增加了读取属性文件的能力

继承自AbstractHandlerMethodMapping

```
//RequestMappingHandlerMapping
@Override
public void afterPropertiesSet() {
    this.config = new RequestMappingInfo.BuilderConfiguration();
    this.config.setUrlPathHelper(getUrlPathHelper());
    this.config.setPathMatcher(getPathMatcher());
    this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
    this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
    this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
    this.config.setContentNegotiationManager(getContentNegotiationManager());

    super.afterPropertiesSet();
}

//AbstractHandlerMethodMapping
@Override
public void afterPropertiesSet() {
    initHandlerMethods();
}
protected void initHandlerMethods() {
    //获取上下文中所有bean的name，不包含父容器
    for (String beanName : getCandidateBeanNames()) {
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            processCandidateBean(beanName);
        }
    }
    //日志记录HandlerMethods的总数量
    handlerMethodsInitialized(getHandlerMethods());
}


protected void processCandidateBean(String beanName) {
    Class<?> beanType = null;
    try {
        //根据name找出bean的类型
        beanType = obtainApplicationContext().getType(beanName);
    } catch (Throwable ex) {
        if (logger.isTraceEnabled()) {
            logger.trace("Could not resolve type for bean '" + beanName + "'", ex);
        }
    }
    //处理Controller和RequestMapping
    if (beanType != null && isHandler(beanType)) {
        detectHandlerMethods(beanName);
    }
}

//RequestMappingHandlerMapping
@Override
protected boolean isHandler(Class<?> beanType) {
    //获取@Controller和@RequestMapping
    return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
            AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
}

//整个controller类的解析过程 
protected void detectHandlerMethods(Object handler) {
    //根据name找出bean的类型
    Class<?> handlerType = (handler instanceof String ?
                obtainApplicationContext().getType((String) handler) : handler.getClass());

    if (handlerType != null) {
            //获取真实的controller，如果是代理类获取父类
        Class<?> userType = ClassUtils.getUserClass(handlerType);
            //对真实的controller所有的方法进行解析和处理  key为方法对象，T为注解封装后的对象RequestMappingInfo
        Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
                (MethodIntrospector.MetadataLookup<T>) method -> {
                        try {
                            // 调用子类RequestMappingHandlerMapping的getMappingForMethod方法进行处理，即根据RequestMapping注解信息创建匹配条件RequestMappingInfo对象
                            return getMappingForMethod(method, userType);
                        }catch (Throwable ex) {
                            throw new IllegalStateException("Invalid mapping on handler class [" +
                                    userType.getName() + "]: " + method, ex);
                        }
                    });
        if (logger.isTraceEnabled()) {
            logger.trace(formatMappings(userType, methods));
        }
        methods.forEach((method, mapping) -> {
                //找出controller中可外部调用的方法
            Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
                //注册处理方法
                registerHandlerMethod(handler, invocableMethod, mapping);
        });
    }
}

//requestMapping封装成RequestMappingInfo对象
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    RequestMappingInfo info = createRequestMappingInfo(method);//解析方法上的requestMapping
    if (info != null) {
        //解析方法所在类上的requestMapping
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
        if (typeInfo != null) {
            info = typeInfo.combine(info);//合并类和方法上的路径,比如Controller类上有@RequestMapping("/demo")，方法的@RequestMapping("/demo1")，结果为"/demo/demo1"
        }
        String prefix = getPathPrefix(handlerType);//合并前缀
        if (prefix != null) {
            info = RequestMappingInfo.paths(prefix).options(this.config).build().combine(info);
        }
    }
    return info;
}

private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
    //找到方法上的RequestMapping注解
    RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element,                                                                         RequestMapping.class);
    //获取自定义的类型条件(自定义的RequestMapping注解)
    RequestCondition<?> condition = (element instanceof Class ?
                                            getCustomTypeCondition((Class<?>) element) :                                                                  getCustomMethodCondition((Method) element));
    return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}

protected RequestMappingInfo createRequestMappingInfo(
    RequestMapping requestMapping, @Nullable RequestCondition<?> customCondition) {

    //获取RequestMapping注解的属性,封装成RequestMappingInfo对象
    RequestMappingInfo.Builder builder = RequestMappingInfo
        .paths(resolveEmbeddedValuesInPatterns(requestMapping.path()))
        .methods(requestMapping.method())
        .params(requestMapping.params())
        .headers(requestMapping.headers())
        .consumes(requestMapping.consumes())
        .produces(requestMapping.produces())
        .mappingName(requestMapping.name());
    if (customCondition != null) {
        builder.customCondition(customCondition);
    }
    return builder.options(this.config).build();
}

//再次封装成对应的对象    面向对象编程  每一个属性都存在多个值得情况需要排重封装
@Override
public RequestMappingInfo build() {
    ContentNegotiationManager manager = this.options.getContentNegotiationManager();

    PatternsRequestCondition patternsCondition = new PatternsRequestCondition(
        this.paths, this.options.getUrlPathHelper(), this.options.getPathMatcher(),
        this.options.useSuffixPatternMatch(), this.options.useTrailingSlashMatch(),
        this.options.getFileExtensions());

    return new RequestMappingInfo(this.mappingName, patternsCondition,
                                  new RequestMethodsRequestCondition(this.methods),
                                  new ParamsRequestCondition(this.params),
                                  new HeadersRequestCondition(this.headers),
                                  new ConsumesRequestCondition(this.consumes, this.headers),
                                  new ProducesRequestCondition(this.produces, this.headers, manager),
                                  this.customCondition);
}
```

```
protected void registerHandlerMethod(Object handler, Method method, T mapping) {
   this.mappingRegistry.register(mapping, handler, method);
}
//mapping是RequestMappingInfo对象   handler是controller类的beanName   method为接口方法
public void register(T mapping, Object handler, Method method) {
    ...
    this.readWriteLock.writeLock().lock();
    try {
        //beanName和method封装成HandlerMethod对象
        HandlerMethod handlerMethod = createHandlerMethod(handler, method);
        //验证RequestMappingInfo是否有对应不同的method，有则抛出异常
        validateMethodMapping(handlerMethod, mapping);
        //RequestMappingInfo和handlerMethod绑定
        this.mappingLookup.put(mapping, handlerMethod);

        List<String> directUrls = getDirectUrls(mapping);//可以配置多个url
        for (String url : directUrls) {
            //url和RequestMappingInfo绑定   可以根据url找到RequestMappingInfo，再找到handlerMethod
            this.urlLookup.add(url, mapping);
        }

        String name = null;
        if (getNamingStrategy() != null) {
            name = getNamingStrategy().getName(handlerMethod, mapping);
            addMappingName(name, handlerMethod);//方法名和Method绑定
        }

        CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
        if (corsConfig != null) {
            this.corsLookup.put(handlerMethod, corsConfig);
        }
        //将RequestMappingInfo  url  handlerMethod绑定到MappingRegistration对象  放入map
        this.registry.put(mapping, new MappingRegistration<>(mapping, handlerMethod, directUrls, name));
    }
    finally {
        this.readWriteLock.writeLock().unlock();
    }
}
```

### BeanNameUrlHandlerMapping

处理Bean的id必须以/开头

```
public class HelloController implements Controller {
    public ModelAndView handleRequest(HttpServletRequest request,
                                      HttpServletResponse response) throws Exception {
        ModelAndView mav = new ModelAndView("index");
        mav.addObject("message", "默认的映射处理器示例");
        return mav;
    }
<bean id="resourceView"
    class="org.springframework.web.servlet.view.InternalResourceViewResolver">
    <property name="prefix" value="/"></property>
    <property name="suffix" value=".jsp"></property>
</bean>
<bean id="/name.do" class="com.mvc.controller.HelloController"></bean>
```

```
//向上继承自ApplicationObjectSupport实现ApplicationContextAware接口
public abstract class ApplicationObjectSupport implements ApplicationContextAware {
    protected final Log logger = LogFactory.getLog(this.getClass());
    @Nullable
    private ApplicationContext applicationContext;
    ...

    public final void setApplicationContext(@Nullable ApplicationContext context) throws BeansException {
        if (context == null && !this.isContextRequired()) {
            this.applicationContext = null;
            this.messageSourceAccessor = null;
        } else if (this.applicationContext == null) {
            if (!this.requiredContextClass().isInstance(context)) {
                throw new ApplicationContextException("Invalid application context: needs to be of type [" + this.requiredContextClass().getName() + "]");
            }

            this.applicationContext = context;
            this.messageSourceAccessor = new MessageSourceAccessor(context);
            this.initApplicationContext(context);//模板方法，调子类
        } else if (this.applicationContext != context) {
            throw new ApplicationContextException("Cannot reinitialize with different application context: current one is [" + this.applicationContext + "], passed-in one is [" + context + "]");
        }

    }
    ...
}

@Override
public void initApplicationContext() throws ApplicationContextException {
    super.initApplicationContext();//加载拦截器
    // 处理url和bean name，具体注册调用父类AbstractUrlHandlerMapping类完成
    detectHandlers();
}

//建立当前ApplicationContext中controller和url的对应关系
protected void detectHandlers() throws BeansException {
    // 获取应用上下文
    ApplicationContext applicationContext = obtainApplicationContext();
    //获取ApplicationContext中的所有bean的name(也是id，即@Controller的属性值)
    String[] beanNames = (this.detectHandlersInAncestorContexts ?
                          BeanFactoryUtils.beanNamesForTypeIncludingAncestors(applicationContext,                                      Object.class) : applicationContext.getBeanNamesForType(Object.class));

    //遍历所有beanName
    for (String beanName : beanNames) {
        // 通过模板方法模式调用BeanNameUrlHandlerMapping子类处理
        String[] urls = determineUrlsForHandler(beanName);//判断是否以/开始
        if (!ObjectUtils.isEmpty(urls)) {
            //调用父类AbstractUrlHandlerMapping将url与handler存入map
            registerHandler(urls, beanName);
        }
    }
}

protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {
    Assert.notNull(urlPath, "URL path must not be null");
    Assert.notNull(handler, "Handler object must not be null");
    Object resolvedHandler = handler;

    // Eagerly resolve handler if referencing singleton via name.
    if (!this.lazyInitHandlers && handler instanceof String) {
        String handlerName = (String) handler;
        ApplicationContext applicationContext = obtainApplicationContext();
        if (applicationContext.isSingleton(handlerName)) {
            resolvedHandler = applicationContext.getBean(handlerName);
        }
    }
    //已存在且指向不同的Handler抛异常
    Object mappedHandler = this.handlerMap.get(urlPath);
    if (mappedHandler != null) {
        if (mappedHandler != resolvedHandler) {
            throw new IllegalStateException(
                "Cannot map " + getHandlerDescription(handler) + " to URL path [" + urlPath +
                "]: There is already " + getHandlerDescription(mappedHandler) + " mapped.");
        }
    }
    else {
        if (urlPath.equals("/")) {
            if (logger.isTraceEnabled()) {
                logger.trace("Root mapping to " + getHandlerDescription(handler));
            }
            setRootHandler(resolvedHandler);//设置根处理器，即请求/的时候
        }
        else if (urlPath.equals("/*")) {
            if (logger.isTraceEnabled()) {
                logger.trace("Default mapping to " + getHandlerDescription(handler));
            }
            setDefaultHandler(resolvedHandler);//默认处理器
        }
        else {
            this.handlerMap.put(urlPath, resolvedHandler);//注册进map
            if (logger.isTraceEnabled()) {
                logger.trace("Mapped [" + urlPath + "] onto " + getHandlerDescription(handler));
            }
        }
    }
}
```

- 处理器bean的id/name为一个url请求路径，前面有"/"；
- 如果多个url映射同一个处理器bean，那么就需要定义多个bean，导致容器创建多个处理器实例，占用内存空间；
- 处理器bean定义与url请求耦合在一起。

### SimpleUrlHandlerMapping

间接实现了org.springframework.web.servlet.HandlerMapping接口，直接实现该接口的是org.springframework.web.servlet.handler.AbstractHandlerMapping抽象类，映射Url与请求handler bean。支持映射bean实例和映射bean名称

```
public class SimpleUrlHandlerMapping extends AbstractUrlHandlerMapping {
    // 存储url和bean映射
    private final Map<String, Object> urlMap = new LinkedHashMap<>();
    // 注入property的name为mappings映射
    public void setMappings(Properties mappings) {
        CollectionUtils.mergePropertiesIntoMap(mappings, this.urlMap);
    }
    // 注入property的name为urlMap映射
    public void setUrlMap(Map<String, ?> urlMap) {
        this.urlMap.putAll(urlMap);
    }
    public Map<String, ?> getUrlMap() {
        return this.urlMap;
    }
    // 实例化本类实例入口
    @Override
    public void initApplicationContext() throws BeansException {
        // 调用父类AbstractHandlerMapping的initApplicationContext方法，只要完成拦截器的注册
        super.initApplicationContext();
        // 处理url和bean name，具体注册调用父类完成
        registerHandlers(this.urlMap);
    }
    // 注册映射关系，及将property中的值解析到map对象中，key为url，value为bean id或name
    protected void registerHandlers(Map<String, Object> urlMap) throws BeansException {
        if (urlMap.isEmpty()) {
            logger.warn("Neither 'urlMap' nor 'mappings' set on SimpleUrlHandlerMapping");
        } else {
            urlMap.forEach((url, handler) -> {
                // 增加以"/"开头
                if (!url.startsWith("/")) {
                    url = "/" + url;
                }
                // 去除handler bean名称的空格
                if (handler instanceof String) {
                    handler = ((String) handler).trim();
                }
                // 调用父类AbstractUrlHandlerMapping完成映射
                registerHandler(url, handler);
            });
        }
    }

}
```

SimpleUrlHandlerMapping类主要接收用户设定的url与handler的映射关系，其实际的工作都是交由其父类来完成的。

在创建初始化SimpleUrlHandlerMapping类时，调用其父类的initApplicationContext()方法，该方法完成拦截器的初始化

```
@Override
protected void initApplicationContext() throws BeansException {
    // 空实现。子类可重写此方法以注册额外的拦截器
    extendInterceptors(this.interceptors);
    // 从上下文中查询拦截器并添加到拦截器列表中
    detectMappedInterceptors(this.adaptedInterceptors);
    // 初始化拦截器
    initInterceptors();
}

// 查找实现了MappedInterceptor接口的bean，并添加到映射拦截器列表
protected void detectMappedInterceptors(List<HandlerInterceptor> mappedInterceptors) {
    mappedInterceptors.addAll(BeanFactoryUtils.beansOfTypeIncludingAncestors(
                    obtainApplicationContext(), MappedInterceptor.class, true, false).values());
}

// 将自定义bean设置到适配拦截器中，bean需实现HandlerInterceptor或WebRequestInterceptor
protected void initInterceptors() {
    if (!this.interceptors.isEmpty()) {
        for (int i = 0; i < this.interceptors.size(); i++) {
            Object interceptor = this.interceptors.get(i);
            if (interceptor == null) {
                throw new IllegalArgumentException("Entry number " + i + " in interceptors array is null");
            }
            this.adaptedInterceptors.add(adaptInterceptor(interceptor));
        }
    }
}
```

在创建初始化SimpleUrlHandlerMapping类时，调用AbstractUrlHandlerMapping类的registerHandler(urlPath,handler)方法

```
protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {
    Assert.notNull(urlPath, "URL path must not be null");
    Assert.notNull(handler, "Handler object must not be null");
    Object resolvedHandler = handler;

    // 不是懒加载，默认为false，即不是，通过配置SimpleUrlHandlerMapping属性lazyInitHandlers的值进行控制
    // 如果不是懒加载并且handler为单例，即从上下文中查询实例处理，此时resolvedHandler为handler实例对象；
    // 如果是懒加载或者handler不是单例，即resolvedHandler为handler逻辑名
    if (!this.lazyInitHandlers && handler instanceof String) {
        String handlerName = (String) handler;
        ApplicationContext applicationContext = obtainApplicationContext();
        // 如果handler是单例，通过bean的scope控制
        if (applicationContext.isSingleton(handlerName)) {
            resolvedHandler = applicationContext.getBean(handlerName);
        }
    }

    Object mappedHandler = this.handlerMap.get(urlPath);
    if (mappedHandler != null) {
        if (mappedHandler != resolvedHandler) {
            throw new IllegalStateException(
                    "Cannot map " + getHandlerDescription(handler) + " to URL path [" + urlPath +
                    "]: There is already " + getHandlerDescription(mappedHandler) + " mapped.");
        }
    }
    else {
        if (urlPath.equals("/")) {
            if (logger.isInfoEnabled()) {
                logger.info("Root mapping to " + getHandlerDescription(handler));
            }
            setRootHandler(resolvedHandler);
        }
        else if (urlPath.equals("/*")) {
            if (logger.isInfoEnabled()) {
                logger.info("Default mapping to " + getHandlerDescription(handler));
            }
            setDefaultHandler(resolvedHandler);
        }
        else {
            // 把url与handler（名称或实例）放入map，以供后续使用
            this.handlerMap.put(urlPath, resolvedHandler);
            if (logger.isInfoEnabled()) {
                logger.info("Mapped URL path [" + urlPath + "] onto " + getHandlerDescription(handler));
            }
        }
    }
}
```

![handlermapping](/img/in-post/post-springmvc/handlermapping.png)

## HandlerAdapters

Spring MVC为我们提供了多种处理用户的处理器（Handler），Spring实现的处理器类型有Servlet、Controller、HttpRequestHandler以及注解类型的处理器，即我们可以通过实现这些接口或者注解我们的类来使用这些处理器，那么针对不同类型的处理器，如何将用户请求转发到相应类型的处理器方法中的呢，这就需求Spring MVC的处理器适配器来完成适配操作，这就是处理器适配器要完成的工作。

- SimpleServletHandlerAdapter 适配Servlet处理器
- HttpRerquestHandlerAdapter 适配HttpRequestHandler处理器
- RequestMappingHandlerAdapter 适配注解处理器
- SimpleControllerHandlerAdapter 适配Controller处理器

Spring MVC默认使用的处理器适配器为：HttpRequestHandlerAdapter、SimpleServletHandlerAdapter、RequestMappingHandlerAdapter三种。

与initHandlerMappings的处理一致，读取DispatcherServlet.properties中的HandlerAdapters，但不是通过后置处理器触发

```
private void initHandlerAdapters(ApplicationContext context) {
    this.handlerAdapters = null;

    if (this.detectAllHandlerAdapters) {
        // 从应用上下文中查找HandlerAdapter
        Map<String, HandlerAdapter> matchingBeans =
            BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
        if (!matchingBeans.isEmpty()) {
            this.handlerAdapters = new ArrayList<>(matchingBeans.values());
            AnnotationAwareOrderComparator.sort(this.handlerAdapters);
        }
    } else {
        try {
         //如果在web.xml配了detectAllHandlerAdapters=false，此时spring会加载名称为handlerAdapter的bean为处理器适配器
            HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
            // 转化为集合赋给handlerAdapters属性
            this.handlerAdapters = Collections.singletonList(ha);
        } catch (NoSuchBeanDefinitionException ex) {
        }
    }

    //如果未配置，读取DispatcherServlet.properties中的HandlerAdapters,放入this.handlerAdapters
    if (this.handlerAdapters == null) {
        this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
        if (logger.isTraceEnabled()) {
            logger.trace("No HandlerAdapters declared for servlet '" + getServletName() +
                         "': using default strategies from DispatcherServlet.properties");
        }
    }
}
```

DispatcherServlet的doDispatch方法会遍历handlerAdapters，找到handler对应的适配器

```
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) {//判断是否适配成功
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler +
                               "]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
}
```

HandlerAdapter的接口中定义了三个方法：

- boolean supports(Object handler); 判断适配器是否适配Handler
- ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)  使用适配的Handler处理用户请求
- long getLastModified(HttpServletRequest request, Object handler); 返回资源的最后修改时间，如果handler实现类不支持可以返回-1

### SimpleServletHandlerAdapter

SimpleSerlvetHandlerAdapter是Spring使用HandlerAdapter最简单的方式，此方式是为了在Spring中支持Servlet方式开发，即把Servlet适配为处理器handler。

是一个Servlet的适配器，其最终执行的方法是Servlet的service方法，非默认提供（DispatcherServlet.properties中没有），需要自己导入bean。

- supports方法就是判断handler是否是Servlet
- getLastModified直接返回-1
- handle方法本质是执行Servlet.service方法。

```
public class SimpleServletHandlerAdapter implements HandlerAdapter {

    //判断handler是否实现Servlet接口
    @Override
    public boolean supports(Object handler) {
        return (handler instanceof Servlet);
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        ((Servlet) handler).service(request, response);
        return null;
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        return -1;
    }

}
```

### SimpleControllerHandlerAdapter

是Controller实现类的适配器类，其本质是执行Controller中的handleRequest方法。

- supports方法就是判断handler是否是Controller
- getLastModified直接返回-1
- handle方法本质是执行Controller.handleRequest方法。

```
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

    @Override
    public boolean supports(Object handler) {
        return (handler instanceof Controller);
    }

    @Override
    @Nullable
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        return ((Controller) handler).handleRequest(request, response);
    }

    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        if (handler instanceof LastModified) {
            return ((LastModified) handler).getLastModified(request);
        }
        return -1L;
    }

}
```

### HttpRequestHandlerAdapter

- supports方法就是判断handler是否是HttpRequestHandler
- getLastModified直接返回-1
- handle方法本质是执行HttpRequestHandler.handleRequest方法。

```
public class HttpRequestHandlerAdapter implements HandlerAdapter {
    @Override
    public boolean supports(Object handler) {
        //判断是否是HttpRequestHandler子类
        return (handler instanceof HttpRequestHandler);
    }
    @Override
    public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
            throws Exception {
        //执行HttpRequestHandler的handleRequest方法
        ((HttpRequestHandler) handler).handleRequest(request, response);
        return null;
    }
    @Override
    public long getLastModified(HttpServletRequest request, Object handler) {
        //返回modified值
        if (handler instanceof LastModified) {
            return ((LastModified) handler).getLastModified(request);
        }
        return -1L;
    }
}
```

### RequestMappingHandlerAdapter

通过继承抽象类AbstractHandlerMethodAdapter实现了HandlerAdapter接口

- 请求适配给`@RequestMapping`类型的Handler处理。
- 采用反射机制调用url请求对应的Controller中的方法（这其中还包括参数处理），返回执行结果值，完成HandlerAdapter的使命
- getLastModified直接返回-1

```
@Override
protected long getLastModifiedInternal(HttpServletRequest request, HandlerMethod handlerMethod) {
    return -1;
}

//通过父类调用
@Override
protected ModelAndView handleInternal(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
 
    ModelAndView mav;
    checkRequest(request);
 
    // 判断当前是否需要支持在同一个session中只能线性地处理请求
    if (this.synchronizeOnSession) {
        // 获取当前请求的session对象
        HttpSession session = request.getSession(false);
        if (session != null) {
            // 为当前session生成一个唯一的可以用于锁定的key
            Object mutex = WebUtils.getSessionMutex(session);
            synchronized (mutex) {
                // 对HandlerMethod进行参数等的适配处理，并调用目标handler
                mav = invokeHandlerMethod(request, response, handlerMethod);
            }
        } else {
            // 如果当前不存在session，则直接对HandlerMethod进行适配
            mav = invokeHandlerMethod(request, response, handlerMethod);
        }
    } else {
        // 如果当前不需要对session进行同步处理，则直接对HandlerMethod进行适配
        mav = invokeHandlerMethod(request, response, handlerMethod);
    }
 
    // 判断当前请求头中是否包含Cache-Control请求头，如果不包含，则对当前response进行处理，为其设置过期时间
    if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
        // 如果当前SessionAttribute中存在配置的attributes，则为其设置过期时间。
        // 这里SessionAttribute主要是通过@SessionAttribute注解生成的
        if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
            applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
        } else {
            // 如果当前不存在SessionAttributes，则判断当前是否存在Cache-Control设置，
            // 如果存在，则按照该设置进行response处理，如果不存在，则设置response中的
            // Cache的过期时间为-1，即立即失效
            prepareResponse(response);
        }
    }
    return mav;
}

//核心处理流程
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
 
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        // 获取容器中全局配置的InitBinder和当前HandlerMethod所对应的Controller中配置的InitBinder，用于进行参数的绑定
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        // 获取容器中全局配置的ModelAttribute和当前HandlerMethod所对应的Controller
        // 中配置的ModelAttribute，这些配置的方法将会在目标方法调用之前进行调用
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
 
        // 将handlerMethod封装为一个ServletInvocableHandlerMethod对象，该对象用于对当前request的整体调用流程进行了封装
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            // 设置当前容器中配置的所有ArgumentResolver
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            // 设置当前容器中配置的所有ReturnValueHandler
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        // 将前面创建的WebDataBinderFactory设置到ServletInvocableHandlerMethod中
        invocableMethod.setDataBinderFactory(binderFactory);
        // 设置ParameterNameDiscoverer，该对象将按照一定的规则获取当前参数的名称
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
 
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        // 这里initModel()方法主要作用是调用前面获取到的@ModelAttribute标注的方法，
        // 从而达到@ModelAttribute标注的方法能够在目标Handler调用之前调用的目的
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
 
        // 获取当前的AsyncWebRequest，这里AsyncWebRequest的主要作用是用于判断目标
        // handler的返回值是否为WebAsyncTask或DefferredResult，如果是这两种中的一种，
        // 则说明当前请求的处理应该是异步的。所谓的异步，指的是当前请求会将Controller中
        // 封装的业务逻辑放到一个线程池中进行调用，待该调用有返回结果之后再返回到response中。
        // 这种处理的优点在于用于请求分发的线程能够解放出来，从而处理更多的请求，只有待目标任务
        // 完成之后才会回来将该异步任务的结果返回。
        AsyncWebRequest asyncWebRequest = WebAsyncUtils
            .createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);
 
        // 封装异步任务的线程池，request和interceptors到WebAsyncManager中
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
 
        // 这里就是用于判断当前请求是否有异步任务结果的，如果存在，则对异步任务结果进行封装
        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) 
                asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            if (logger.isDebugEnabled()) {
                logger.debug("Found concurrent result value [" + result + "]");
            }
            // 封装异步任务的处理结果，虽然封装的是一个HandlerMethod，但只是Spring简单的封装
            // 的一个Callable对象，该对象中直接将调用结果返回了。这样封装的目的在于能够统一的
            // 进行右面的ServletInvocableHandlerMethod.invokeAndHandle()方法的调用
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }
 
        // 对请求参数进行处理，调用目标HandlerMethod，并且将返回值封装为一个ModelAndView对象
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }
 
        // 对封装的ModelAndView进行处理，主要是判断当前请求是否进行了重定向，如果进行了重定向，
        // 还会判断是否需要将FlashAttributes封装到新的请求中
        return getModelAndView(mavContainer, modelFactory, webRequest);
    } finally {
        // 调用request destruction callbacks和对SessionAttributes进行处理
        webRequest.requestCompleted();
    }
}
```

- 获取当前容器中使用`@InitBinder`注解注册的属性转换器；
- 获取当前容器中使用`@ModelAttribute`标注但没有使用`@RequestMapping`标注的方法，并且在调用目标方法之前调用这些方法；
- 判断目标handler返回值是否使用了WebAsyncTask或DefferredResult封装，如果封装了，则按照异步任务的方式进行执行；
- 处理请求参数，调用目标方法和处理返回值。

![handleradapter](/img/in-post/post-springmvc/handleradapter.png)