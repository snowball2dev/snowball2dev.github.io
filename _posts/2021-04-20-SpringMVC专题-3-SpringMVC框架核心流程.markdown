---
layout:     post
title:      "SpringMVC专题-2.SpringMVC框架核心流程"
subtitle:   " \"SpringMVC框架核心流程\""
date:       2021-04-20 16:30:00 +0800
author:     "snowball"
header-img: "img/home-bg-o.jpg"
tags:
    - SpringMVC核心流程
    - SpringMVC容器初始化
    - Spring学习笔记
    - 技术
typora-root-url: ..
---

<!--  “Spring MVC. ” -->

## **容器初始化，请求处理**

web.xml配置

```
<web-app>
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>

    <servlet>
        <servlet-name>app</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:springmvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>app</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
</web-app>
```

## ContextLoaderListener

ContextLoaderListener通过实现ServletContextListener接口，将spring容器融入web容器当中。这个可以分两个角度来理解：

web项目自身：接收web容器启动web应用的通知，开始自身配置的解析加载，创建bean实例，通过一个WebApplicationContext来维护spring项目的主容器相关的bean，以及其他一些组件。

web容器：web容器使用ServletContext来维护每一个web应用，ContextLoaderListener将spring容器，即WebApplicationContext，作为ServletContext的一个attribute，key为，保存在ServletContext中，从而web容器和spring项目可以通过ServletContext来交互。

### 创建父容器

作为一个中间层来建立spring容器和web容器的关联关系

```
//监听servlet容器的启动
@Override
public void contextInitialized(ServletContextEvent event) {
    initWebApplicationContext(event.getServletContext());//在父类ContextLoader中实现
}
```

**ContextLoader**

具体完成监听逻辑，初始化web应用上下文，即root ApplicationContext，他的主要流程就是创建一个IOC容器，并将创建的IOC容器存到servletContext中

1.contextId：当前容器的id，主要给底层所使用的BeanFactory，在进行序列化时使用。

2.contextConfigLocation：配置文件的位置，默认为WEB-INF/applicationContext.xml，可以通过在web.xml使用context-param标签来指定其他位置，其他名字或者用逗号分隔指定多个。在配置文件中通过beans作为主标签来定义bean。这样底层的BeanFactory会解析beans标签以及里面的bean，从而来创建BeanDefinitions集合，即bean的元数据内存数据库。

3.contextClass：当前所使用的WebApplicationContext的类型，如果是在WEB-INF/applicationContext.xml中指定beans，则使用XmlWebApplicationContext，如果是通过注解，如@Configuration，@Component等，则是AnnotationConfigWebApplicationContext，通过扫描basePackages指定的包来创建bean。

4.contextInitializerClasses：ApplicationContextInitializer的实现类，即在调用ApplicationContext的refresh加载beanDefinition和创建bean之前，对WebApplicationContext进行一些初始化。

```
//创建和初始化spring主容器对应的WebApplicationContext对象实例并调用refresh方法完成从contextConfigLocation指定的配置中，加载BeanDefinitions和创建bean实例
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    //判断是否已经有Root  WebApplicationContext，已经有则抛出异常
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        throw new IllegalStateException(
            "Cannot initialize context because there is already a root application context present - " +
            "check whether you have multiple ContextLoader* definitions in your web.xml!");
    }

    servletContext.log("Initializing Spring root WebApplicationContext");
    Log logger = LogFactory.getLog(ContextLoader.class);
    if (logger.isInfoEnabled()) {
        logger.info("Root WebApplicationContext: initialization started");
    }
    long startTime = System.currentTimeMillis();

    try {
        if (this.context == null) {
            //创建上下文对象  XmlWebApplicationContext(静态方法中从ContextLoader.properties文件中读取)  并赋值给全局变量context
            this.context = createWebApplicationContext(servletContext);
        }
        if (this.context instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
            if (!cwac.isActive()) {
                if (cwac.getParent() == null) {
                    // 设置父容器（如果有）
                    ApplicationContext parent = loadParentContext(servletContext);
                    cwac.setParent(parent);
                }
                //核心方法，完成配置加载，BeanDefinition定义和bean对象创建
                configureAndRefreshWebApplicationContext(cwac, servletContext);
            }
        }
        //ioc容器上下文设置到servlet上下文servletContext
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

        ClassLoader ccl = Thread.currentThread().getContextClassLoader();
        if (ccl == ContextLoader.class.getClassLoader()) {
            currentContext = this.context;
        }
        else if (ccl != null) {
            //将当前类加载器和上下文绑定
            currentContextPerThread.put(ccl, this.context);
        }

        if (logger.isInfoEnabled()) {
            long elapsedTime = System.currentTimeMillis() - startTime;
            logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
        }

        return this.context;
    }
    catch (RuntimeException | Error ex) {
        logger.error("Context initialization failed", ex);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
        throw ex;
    }
}

protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        }
        else {
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                      ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }

    wac.setServletContext(sc);
    //获取web.xml中的配置contextConfigLocation
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }

    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }
    // 使用ApplicationContextInitializer对ApplicationContext进行初始化
    customizeContext(sc, wac);
    //ApplicationContext的核心方法
    wac.refresh();
}
```

ContextLoaderListener监听器的作用就是启动Web容器时，自动装配ApplicationContext的配置信息。因为它实现了ServletContextListener这个接口，在web.xml配置了这个监听器，启动容器时，就会默认执行它实现的contextInitialized()方法初始化WebApplicationContext实例（XmlWebApplicationContext），并放入到ServletContext中。由于在ContextLoaderListener中关联了ContextLoader这个类，所以整个加载配置过程由ContextLoader来完成。

如果web.xml中加入了ContextLoaderListener则初始化SpringIOC容器，即root ApplicationContext，springmvc维护一个子容器，通过DispachterServlet来初始化，并通过parent变量持有父容器的引用

## DispatcherServlet

```
public class DispatcherServlet extends FrameworkServlet {
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
public abstract class HttpServletBean extends HttpServlet implements EnvironmentCapable, EnvironmentAware {
```

DispatcherServlet就是一个servlet，生命周期（实例化(无参构造器)，初始化（init），调用service(doService)，销毁(distory)）

### 初始化

#### HttpServletBean

继承HttpServlet，实现了EnvironmentAware（注入Environment对象）和EnvironmentCapable（访问Environment对象）接口，其中Environment主要从类路径的属性文件，运行参数，@PropertySource注解等获取应用的相关属性值，以提供给spring容器相关组件访问，或者写入属性到Environment来给其他组件访问。HttpServletBean的主要作用就是将于该servlet相关的init-param，封装成bean属性，然后保存到Environment当中，从而可以在spring容器中被其他bean访问。

```
//DispatcherServlet第一次加载时调用init方法
@Override
public final void init() throws ServletException {
    ...
    try {
         //加载web.xml文件中的servlet标签中的init-param，其中含有springMVC的配置文件的名字和路径若没有，则默认为（servlet-name）-servlet.xml，默认路径为WEF—INF下，设置到DispatcherServlet中
         PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
         //创建BeanWrapper实例，为DispatcherServlet设置属性
         BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
         ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
         bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
         initBeanWrapper(bw);
         //把init-param中的参数设置到DispatcherServlet里面去
         bw.setPropertyValues(pvs, true);
    }catch (BeansException ex) {
        logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
        throw ex;
    }

    //调用子类（FrameworkServlet）进行初始化
    // 模版方法，此方法在HttpServletBean本身是空的，但是因为调用方法的对象是DispatcherServlet,所以优先在DispatcherServlet找，找不到再去父类找，最后在FrameworkServlet找到
    initServletBean();

    if (logger.isDebugEnabled()) {
        logger.debug("Servlet '" + getServletName() + "' configured successfully");
    }
}
```

- 获取web.xml配置DispatcherServlet的初始化参数，存放到一个参数容器ServletConfigPropertyValues中
- 根据传进来的this创建BeanWrapper，本质上它就是DispatcherServlet
- 通过bw.setPropertyValues(pvs, true)，把参数设置到bw(即DispatcherServlet)里面去
- 最后调用子类的initServletBean()

#### FrameworkServlet

继承于HttpServletBean，因为DispatcherServlet通常包含一个独立的WebApplication,而普通的servlet则只是通过servletContext获取spring容器的root WebApplicationContext,从而从中获取相关bean，所以在DispatcherServlet类层次结构中,增加FrameworkServlet这层设计。作用就是用于获取，创建与管理，DispatcherServlet所绑定的WebApplicationContext对象,即完成WebApplicationContext的创建相关的：从contextConfigLocation获取xml或WebApplicationInitializer配置信息,根据contextClass创建WebApplicationContext，以及获取ApplicationContextInitializer来对WebApplicationContext进行初始化，最后调用refresh完成DispatcherServlet绑定的这个WebApplicationContext的创建。

重写了HttpServletBean的initServletBean()方法。

```
@Override
protected final void initServletBean() throws ServletException {
    ...
    try {
        //创建springmvc的ioc容器实例,初始化WebApplicationContext并调用子类（DispatcherServlet）的onRefresh(wac)方法
        this.webApplicationContext = initWebApplicationContext();
        initFrameworkServlet();
    }
    ...
}

protected WebApplicationContext initWebApplicationContext() {
    //通过ServletContext获得spring容器,获取root WebApplicationContext，即web.xml中配置的listener（ContextLoaderListener）
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    //定义springMVC容器wac
    WebApplicationContext wac = null;


    //判断容器是否由编程式传入（即是否已经存在了容器实例），存在的话直接赋值给wac，给springMVC容器设置父容器
    //最后调用刷新函数configureAndRefreshWebApplicationContext(wac)，作用是把springMVC的配置信息加载到容器中去（之前已经将配置信息的路径设置到了bw中）
    if (this.webApplicationContext != null) {
        // context上下文在构造时注入
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                // context没有被refreshed，设置父context、设置应用context id等服务
                if (cwac.getParent() == null) {
                    //将spring ioc设置为springMVC ioc的父容器
                    cwac.setParent(rootContext);
                }
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    if (wac == null) {
        // 在ServletContext中寻找是否有springMVC容器，初次运行是没有的，springMVC初始化完毕ServletContext就有了springMVC容器
        wac = findWebApplicationContext();
    }

    //当wac既没有没被编程式注册到容器中的，也没在ServletContext找得到，此时就要新建一个springMVC容器
    if (wac == null) {
        // 创建springMVC容器
        wac = createWebApplicationContext(rootContext);//会加载并触发监听  执行onRefresh，refreshEventReceived设置为true
    }

    if (!this.refreshEventReceived) {//如果监听器未接收到事件
        //到这里mvc的容器已经创建完毕，接着才是真正调用DispatcherServlet的初始化方法onRefresh(wac),模板模式
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }

    if (this.publishContext) {
        //将springMVC容器存放到ServletContext中去，方便下次取出来
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }
    return wac;
}


protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {
    Class<?> contextClass = getContextClass();
    if (this.logger.isDebugEnabled()) {
        this.logger.debug("Servlet with name '" + getServletName() +
                          "' will try to create custom WebApplicationContext context of class '" +
                          contextClass.getName() + "'" + ", using parent context [" + parent + "]");
    }
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException(
            "Fatal initialization error in servlet with name '" + getServletName() +
            "': custom WebApplicationContext class [" + contextClass.getName() +
            "] is not of type ConfigurableWebApplicationContext");
    }
    //实例化空白的ioc容器
    ConfigurableWebApplicationContext wac =
        (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
    //给容器设置环境
    wac.setEnvironment(getEnvironment());
    //给容器设置父容器(就是spring容器)，两个ioc容器关联在一起了
    wac.setParent(parent);
    //给容器加载springMVC的配置信息，之前已经通过bw将配置文件路径写入到了DispatcherServlet中
    wac.setConfigLocation(getContextConfigLocation());
    //上面提到过这方法，刷新容器，根据springMVC配置文件完成初始化操作，此时springMVC容器创建完成
    configureAndRefreshWebApplicationContext(wac);

    return wac;
}

protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        if (this.contextId != null) {
            wac.setId(this.contextId);
        }
        else {
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                      ObjectUtils.getDisplayString(getServletContext().getContextPath()) + '/' + getServletName());
        }
    }

    wac.setServletContext(getServletContext());
    wac.setServletConfig(getServletConfig());
    wac.setNamespace(getNamespace());
    wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
    }

    postProcessWebApplicationContext(wac);
    applyInitializers(wac);
    wac.refresh();//registerListeners会注册ContextRefreshListener，finishRefresh中发布事件（publishEvent(new ContextRefreshedEvent(this))）并触发监听逻辑,调用DispatcherServlet的onRefresh
}
```

- 创建Spring MVC的容器，根据配置文件实例化里面各种bean，并将之与spring的容器进行关联
- 把创建出来的Spring MVC容器存放到ServletContext中
- 通过模板方法模式调用子类DispatcherServlet的onRefresh()方法

#### DispatcherServlet

从FrameworkServlet中获取WebApplicationContext，然后从WebApplicationContext中获取DispatcherServlet的相关功能子组件bean，然后在自身维护一个引用。实现doService方法并使用这些功能子组件来完成请求的处理和生成响应。

```
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

//初始化九大核心组件
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);//文件上传解析
    initLocaleResolver(context);//本地解析
    initThemeResolver(context);//主题解析
    initHandlerMappings(context);//url请求映射
    initHandlerAdapters(context);//初始化真正调用controloler方法的类
    initHandlerExceptionResolvers(context);//异常解析
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);//视图解析
    initFlashMapManager(context);
}
```

### 请求处理

```
HttpServlet
protected void service(HttpServletRequest req, HttpServletResponse resp)
    throws ServletException, IOException {

    String method = req.getMethod();

    if (method.equals(METHOD_GET)) {
        long lastModified = getLastModified(req);
        if (lastModified == -1) {
            // servlet doesn't support if-modified-since, no reason
            // to go through further expensive logic
            doGet(req, resp);    
            ...
            doPost
            ...
            doHead
                
FrameworkServlet doGet、doPost。。。实现统一调用processRequest
protected final void doGet(HttpServletRequest request, HttpServletResponse response)
                throws ServletException, IOException {
   processRequest(request, response);
}                

            
processRequest的实现
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

    long startTime = System.currentTimeMillis();
    Throwable failureCause = null;
    //previousLocaleContext获取和当前线程相关的LocaleContext,根据已有请求构造一个新的和当前线程相关的LocaleContext
    LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
    LocaleContext localeContext = buildLocaleContext(request);

    //previousAttributes获取和当前线程绑定的RequestAttributes
    RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
    //为已有请求构造新的ServletRequestAttributes，加入预绑定属性
    ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);//异步请求处理
    asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

    //initContextHolders让新构造的RequestAttributes和ServletRequestAttributes和当前线程绑定，加入到ThreadLocal，完成绑定
    initContextHolders(request, localeContext, requestAttributes);

    try {
        //抽象方法doService由FrameworkServlet子类DispatcherServlet重写
        doService(request, response);
    }catch (ServletException | IOException ex) {
        failureCause = ex;
        throw ex;
    }catch (Throwable ex) {
        failureCause = ex;
        throw new NestedServletException("Request processing failed", ex);
    }finally {
        //解除RequestAttributes,ServletRequestAttributes和当前线程的绑定
        resetContextHolders(request, previousLocaleContext, previousAttributes);
        if (requestAttributes != null) {
            requestAttributes.requestCompleted();
        }
        logResult(request, response, failureCause, asyncManager);
        //注册监听事件ServletRequestHandledEvent,在调用上下文的时候产生Event
        publishRequestHandledEvent(request, response, startTime, failureCause);
    }
}
```

DispatcherServlet

```
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    logRequest(request);

    // Keep a snapshot of the request attributes in case of an include,
    // to be able to restore the original attributes after the include.
    Map<String, Object> attributesSnapshot = null;
    if (WebUtils.isIncludeRequest(request)) {
        attributesSnapshot = new HashMap<>();//保存request域中的数据,存一份快照
        Enumeration<?> attrNames = request.getAttributeNames();
        while (attrNames.hasMoreElements()) {
            String attrName = (String) attrNames.nextElement();
            if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
                attributesSnapshot.put(attrName, request.getAttribute(attrName));
            }
        }
    }

    //设置web应用上下文
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    //国际化本地
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    //样式
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    //设置样式资源
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());

    //请求刷新时保存属性
    if (this.flashMapManager != null) {
        FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
        if (inputFlashMap != null) {
            request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
        }
        //Flash attributes 在对请求的重定向生效之前被临时存储（通常是在session)中，并且在重定向之后被立即移除
        request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        //FlashMap 被用来管理 flash attributes 而 FlashMapManager 则被用来存储，获取和管理 FlashMap 实体
        request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
    }

    try {
        doDispatch(request, response);//核心方法
    }finally {
        if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
            // Restore the original attribute snapshot, in case of an include.
            if (attributesSnapshot != null) {
                restoreAttributesAfterInclude(request, attributesSnapshot);//将快照覆盖回去
            }
        }
    }
}

protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;

    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            //将request转换成multipartRequest，并检查是否解析成功(判断是否有文件上传)
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            //根据请求信息获取handler（包含了拦截器）
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                noHandlerFound(processedRequest, response);
                return;
            }

            //根据handler获取adapter
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
            //拦截器逻辑
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            //执行业务处理，返回视图模型
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
            //给视图模型设置viewName
            applyDefaultViewName(processedRequest, mv);
            //拦截器逻辑
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }catch (Exception ex) {
            dispatchException = ex;
        }catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        //处理请求结果,使用了组件LocaleResolver, ViewResolver和ThemeResolver(view#render)
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

```
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
        @Nullable Exception exception) throws Exception {

    boolean errorView = false;

    //异常处理
    if (exception != null) {
        //ModelAndViewDefiningException类型，会携带对应的ModelAndView
        if (exception instanceof ModelAndViewDefiningException) {
            logger.debug("ModelAndViewDefiningException encountered", exception);
            mv = ((ModelAndViewDefiningException) exception).getModelAndView();
        }else {
            //由对应的处理器handler进行异常处理，返回ModelAndView。其中使用了HandlerExceptionResolver。
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
            errorView = (mv != null);
        }
    }
    ....
protected ModelAndView processHandlerException(HttpServletRequest request, HttpServletResponse response,
            @Nullable Object handler, Exception ex) throws Exception {

        // Success and error responses may use different content types
        request.removeAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);

        // Check registered HandlerExceptionResolvers...
        ModelAndView exMv = null;
        if (this.handlerExceptionResolvers != null) {
            for (HandlerExceptionResolver resolver : this.handlerExceptionResolvers) {
                exMv = resolver.resolveException(request, response, handler, ex);
                if (exMv != null) {
                    break;
                }
            }
        }
        ......
            
            
```

创建一个自己异常处理类MyHandlerExceptionResolver，只要实现HandlerExceptionResolver即可，实现的MyHandlerExceptionResolver注入到容器中

### 配置MVC

JavaConfig方式配置webMvc：

1. webmvcconfigurer接口   

2. 继承WebMvcConfigurerAdapter

3. 继承webmvcconfigurationsupport  核心类   有默认的实现（非空）   

@EnableWebMvc包含了webmvcconfigurationsupport，WebMvcConfigurerComposite  webmvcconfigurer接口 的聚合类

自定义使用方法

  @EnableWebMvc  +  实现webmvcconfigurer接口

  继承webmvcconfigurationsupport  +  @Configuarion



![Dispatcher](/img/in-post/post-springmvc/Dispatcher.png)