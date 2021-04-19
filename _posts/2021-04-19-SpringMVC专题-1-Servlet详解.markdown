---
layout:     post
title:      "SpringMVC专题-1.Servlet详解"
subtitle:   " \"Servlet详解\""
date:       2021-04-19 15:25:00 +0800
author:     "snowball"
header-img: "img/home-bg-o.jpg"
tags:
    - SpringMVC
    - Servlet
    - Spring学习笔记
    - 技术
typora-root-url: ..
---

<!--  “Spring MVC. ” -->

一般来说，浏览器通过url访问资源后台程序处理有三个步骤：

1. 接收请求
2. 处理请求
3. 响应请求

web服务器：将某个主机上的资源映射为一个URL供外界访问，完成接收和响应请求

servlet容器：存放着servlet对象（由程序员编程提供），处理请求

## Servlet

```
public interface Servlet {
    //tomcat反射创建servlet之后，调用init方法传入ServletConfig
    void init(ServletConfig var1) throws ServletException;
    ServletConfig getServletConfig();
    //tomcat解析http请求，封装成对象传入
    void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;
    String getServletInfo();
    void destroy();
}
```

ServletConfig：封装了servlet的参数信息，从web.xml中获取，init-param

ServletRequest：http请求到了tomcat后，tomcat通过字符串解析，把各个请求头（header），请求地址（URL），请求参数（queryString）都封装进Request。

ServletResponse：Response在tomcat传给servlet时还是空的对象，servlet逻辑处理后，最终通过response.write()方法，将结果写入response内部的缓冲区,tomcat会在servlet处理结束后拿到response，获取里面的信息，组装成http响应给客户端

## GenericServlet

改良版的servlet，抽象类

```
public abstract class GenericServlet implements Servlet, ServletConfig, Serializable {
    private static final long serialVersionUID = 1L;
    private transient ServletConfig config;

    public GenericServlet() {
    }

    //并不是销毁servlet的方法，而是销毁servlet前一定会调用的方法。默认空实现,可以借此关闭一些资源
    public void destroy() {
    }

    public String getInitParameter(String name) {
        return this.getServletConfig().getInitParameter(name);
    }

    public Enumeration<String> getInitParameterNames() {
        return this.getServletConfig().getInitParameterNames();
    }

    public ServletConfig getServletConfig() {
        return this.config;//初始化时已被赋值
    }

    public ServletContext getServletContext() {
        //通过ServletConfig获取ServletContext
        return this.getServletConfig().getServletContext();
    }

    public String getServletInfo() {
        return "";
    }

    public void init(ServletConfig config) throws ServletException {
        this.config = config;//提升ServletConfig作用域，由局部变量变成全局变量
        this.init();//提供给子类覆盖
    }

    public void init() throws ServletException {
    }

    public void log(String message) {
        this.getServletContext().log(this.getServletName() + ": " + message);
    }

    public void log(String message, Throwable t) {
        this.getServletContext().log(this.getServletName() + ": " + message, t);
    }

    //空实现
    public abstract void service(ServletRequest var1, ServletResponse var2) throws ServletException, IOException;

    public String getServletName() {
        return this.config.getServletName();
    }
}
```

## HttpServlet

GenericServlet的升级版，针对http请求所定制，在GenericServlet的基础上增加了service方法的实现，完成请求方法的判断抽象类，用来被子类继承，得到匹配http请求的处理，子类必须重写以下方法中的一个doGet，doPost，doPut，doDelete 未重写会报错（400,405），service方法不可以重写，采用模板模式实现

```
public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
    HttpServletRequest request;
    HttpServletResponse response;
    try {
        request = (HttpServletRequest)req;//强转成http类型，功能更强大
        response = (HttpServletResponse)res;
    } catch (ClassCastException var6) {
        throw new ServletException(lStrings.getString("http.non_http"));
    }

    this.service(request, response);//每次都调
}

protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String method = req.getMethod();//获取请求方式
    long lastModified;
    if (method.equals("GET")) {//判断逻辑，调用不同的处理方法
        lastModified = this.getLastModified(req);
        if (lastModified == -1L) {
            //本来业务逻辑应该直接写在这里，但是父类无法知道子类具体的业务逻辑，所以抽成方法让子类重写，父类的默认实现输出405，没有意义
            this.doGet(req, resp);
        } else {
            long ifModifiedSince;
            try {
                ifModifiedSince = req.getDateHeader("If-Modified-Since");
            } catch (IllegalArgumentException var9) {
                ifModifiedSince = -1L;
            }

            if (ifModifiedSince < lastModified / 1000L * 1000L) {
                this.maybeSetLastModified(resp, lastModified);
                this.doGet(req, resp);
            } else {
                resp.setStatus(304);
            }
        }
    } else if (method.equals("HEAD")) {
        lastModified = this.getLastModified(req);
        this.maybeSetLastModified(resp, lastModified);
        this.doHead(req, resp);
    } else if (method.equals("POST")) {
        this.doPost(req, resp);
    } else if (method.equals("PUT")) {
        this.doPut(req, resp);
    } else if (method.equals("DELETE")) {
        this.doDelete(req, resp);
    } else if (method.equals("OPTIONS")) {
        this.doOptions(req, resp);
    } else if (method.equals("TRACE")) {
        this.doTrace(req, resp);
    } else {
        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[]{method};
        errMsg = MessageFormat.format(errMsg, errArgs);
        resp.sendError(501, errMsg);
    }

}
```

一个类被声明为抽象的，一般有两个原因：

- 有抽象方法需要被实现
- 没有抽象方法，但是不希望被实例化

## ServletContext

servlet上下文，代表web.xml文件，其实就是一个map，服务器会为每个应用创建一个servletContext对象：

- 创建是在服务器启动时完成
- 销毁是在服务器关闭时完成

javaWeb中的四个域对象：都可以看做是map，都有getAttribute()/setAttribute()方法。

- ServletContext域（Servlet间共享数据）
- Session域（一次会话间共享数据，也可以理解为多次请求间共享数据）
- Request域（同一次请求共享数据）
- Page域（JSP页面内共享数据）

servletConfig对象持有ServletContext的引用，Session域和Request域也可以得到ServletContext

五种方法获取：

\* ServletConfig#getServletContext();

\* GenericServlet#getServletContext();

\* HttpSession#getServletContext();

\* HttpServletRequest#getServletContext();

\* ServletContextEvent#getServletContext();//创建ioc容器时的监听

## Filter

不仅仅是拦截Request

拦截方式有四种：

![Filter](/img/in-post/post-springmvc/filter.jpg)

Redirect和REQUEST/FORWARD/INCLUDE/ERROR最大区别在于：

- 重定向会导致浏览器发送**2**次请求，FORWARD们是服务器内部的**1**次请求

因为FORWARD/INCLUDE等请求的分发是服务器内部的流程，不涉及浏览器，REQUEST/FORWARD/INCLUDE/ERROR和Request有关，Redirect通过Response发起

通过配置，Filter可以过滤服务器内部转发的请求

## Servlet映射器原理

每一个url要交给哪个servlet处理，由映射器决定

![servlet映射器](/img/in-post/post-springmvc/servlet映射器.png)

映射器在tomcat中就是org.apache.catalina.mapper.Mapper类：

internalMapWrapper方法定义了七种映射规则

```
private final void internalMapWrapper(ContextVersion contextVersion,
                                          CharChunk path,
                                          MappingData mappingData) throws IOException {

    int pathOffset = path.getOffset();
    int pathEnd = path.getEnd();
    boolean noServletPath = false;

    int length = contextVersion.path.length();
    if (length == (pathEnd - pathOffset)) {
        noServletPath = true;
    }
    int servletPath = pathOffset + length;
    path.setOffset(servletPath);

    // Rule 1 -- 精确匹配
    MappedWrapper[] exactWrappers = contextVersion.exactWrappers;
    internalMapExactWrapper(exactWrappers, path, mappingData);

    // Rule 2 -- 前缀匹配
    boolean checkJspWelcomeFiles = false;
    MappedWrapper[] wildcardWrappers = contextVersion.wildcardWrappers;
    if (mappingData.wrapper == null) {
        internalMapWildcardWrapper(wildcardWrappers, contextVersion.nesting,
                                   path, mappingData);
        if (mappingData.wrapper != null && mappingData.jspWildCard) {
            char[] buf = path.getBuffer();
            if (buf[pathEnd - 1] == '/') {
                mappingData.wrapper = null;
                checkJspWelcomeFiles = true;
            } else {
                // See Bugzilla 27704
                mappingData.wrapperPath.setChars(buf, path.getStart(),
                                                 path.getLength());
                mappingData.pathInfo.recycle();
            }
        }
    }

    if(mappingData.wrapper == null && noServletPath &&
       contextVersion.object.getMapperContextRootRedirectEnabled()) {
        // The path is empty, redirect to "/"
        path.append('/');
        pathEnd = path.getEnd();
        mappingData.redirectPath.setChars
            (path.getBuffer(), pathOffset, pathEnd - pathOffset);
        path.setEnd(pathEnd - 1);
        return;
    }

    // Rule 3 -- 扩展名匹配
    MappedWrapper[] extensionWrappers = contextVersion.extensionWrappers;
    if (mappingData.wrapper == null && !checkJspWelcomeFiles) {
        internalMapExtensionWrapper(extensionWrappers, path, mappingData,
                                    true);
    }

    ...
```

上面都不匹配，则交给DefaultServlet，就是简单地用IO流读取静态资源并响应给浏览器。如果资源找不到，报404错误

对于静态资源，Tomcat最后会交由一个叫做DefaultServlet的类来处理对于Servlet ，Tomcat最后会交由一个叫做 InvokerServlet的类来处理对于JSP，Tomcat最后会交由一个叫做JspServlet的类来处理

![servlet映射器访问](/img/in-post/post-springmvc/servlet映射器访问.png)

也就是说，servlet,/*这种配置，相当于把DefaultServlet、JspServlet以及我们自己写的其他Servlet都“短路”了，它们都失效了。

这会导致两个问题：

- JSP无法被编译成Servlet输出HTML片段（JspServlet短路）
- HTML/CSS/JS/PNG等资源无法获取（DefaultServlet短路）

DispatcherServlet配置/，会和DefaultServlet产生路径冲突，从而覆盖DefaultServlet。此时，所有对静态资源的请求，映射器都会分发给我们自己写的DispatcherServlet处理。遗憾的是，它只写了业务代码，并不能IO读取并返回静态资源。JspServlet的映射路径没有被覆盖，所以动态资源照常响应。

DispatcherServlet配置/*，虽然JspServlet和DefaultServlet拦截路径还是.jsp和/，没有被覆盖，但无奈的是在到达它们之前，请求已经被DispatcherServlet抢去，所以最终不仅无法处理JSP，也无法处理静态资源。

tomcat中conf/web.xml

```
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    <init-param>
        <param-name>debug</param-name>
        <param-value>0</param-value>
    </init-param>
    <init-param>
        <param-name>listings</param-name>
        <param-value>false</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>


<servlet>
    <servlet-name>jsp</servlet-name>
    <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
    <init-param>
        <param-name>fork</param-name>
        <param-value>false</param-value>
    </init-param>
    <init-param>
        <param-name>xpoweredBy</param-name>
        <param-value>false</param-value>
    </init-param>
    <load-on-startup>3</load-on-startup>
</servlet>

<!-- The mappings for the JSP servlet -->
<servlet-mapping>
    <servlet-name>jsp</servlet-name>
    <url-pattern>*.jsp</url-pattern>
    <url-pattern>*.jspx</url-pattern>
</servlet-mapping>
```

相当于每个应用默认都配置了JSPServlet和DefaultServlet处理JSP和静态资源。

### Tomca容器初始化

监听器模式   

addWebapp

1、Server  --->  Service  -->  Connector  Engine addChild--->  context(servlet容器)

2、LifecycleListener监听器：ContextConfig  放在 context里面

向内引爆

StandardServer.startInternal  -->StandardService.startInternal-->StandardContext

Container-->Engine

模板类：LifecycleBase

startInternal：容器启动

webconfig()

fireLifecycleEvent发布监听，通过监听器完成servlet包装成了wrapper  context

![tomcat容器初始化](/img/in-post/post-springmvc/tomcat容器初始化.png)

### Servlet初始化

![servlet初始化](/img/in-post/post-springmvc/servlet初始化.png)

### Servlet调用

![servlet调用](/img/in-post/post-springmvc/servlet调用.png)

