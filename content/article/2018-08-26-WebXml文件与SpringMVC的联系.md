---
layout: post
title: WebXml文件与SpringMVC的联系
date: 2018-08-26
tags: 
- SpringMVC
- Servlet
categories: 
- 技术
---

无论采用何种框架来进行Java Web的开发，只要是Web项目必须在WEB-INF下有web.xml，这是java规范。 当然，我们最早接触到Java Web容器通常是tomcat，但这并不意味着web.xml是属于Tomcat的，同样，Servlet本身也不属于Tomcat，它与JSP等是Java Web的基础规范。而Servlet的运行需要有Servlet容器的支持，常见的容器有Tomcat、Jetty、JBoss等。

<!-- more -->

对于一个web.xml文件，比较重要的节点有context-param、listener、filter以及servlet：
```xml
<!-- context-param元素声明应用范围内的初始化参数 -->
<context-param></context-param>
<!-- filter 过滤器元素将一个名字与一个实现javax.servlet.Filter接口的类相关联。 -->
<filter></filter>
<!-- filter-mapping 一旦命名了一个过滤器，就要利用filter-mapping元素把它与一个或多个servlet或JSP页面相关联。 -->
<filter-mapping></filter-mapping>
<!-- listener 对事件监听程序的支持，事件监听程序在建立、修改和删除会话或servlet环境时得到通知。Listener元素指出事件监听程序类。 -->
<listener></listener>
<!-- servlet 在向servlet或JSP页面制定初始化参数或定制URL时，必须首先命名servlet或JSP页面。Servlet元素就是用来完成此项任务的。 -->
<servlet></servlet>
```
其他关于web.xml解释可以参考这里，本文不再详细介绍。本文主要是在整理关于Java Web使用SpringMVC过程的心得。

我们知道一个最基本的Spring MVC项目需要在web.xml中区配置如下信息：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://java.sun.com/xml/ns/javaee" xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd" id="WebApp_ID" version="2.5">
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:application-context.xml</param-value>
    </context-param>
    
    <listener>
        <listener-class>
            org.springframework.web.context.ContextLoaderListener
        </listener-class>
    </listener>

    <servlet>
        <servlet-name>web-app</servlet-name>
        <servlet-class>
            org.springframework.web.servlet.DispatcherServlet
        </servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:context-mvc.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>web-app</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```
对于这样一份配置，我们不仅要知其然还要知其所以然。这样不仅能够加深对Java web以及web.xml的理解，对于以后排错也会有很大的帮助。

#### 回到一个普通的Java Web项目

为什么要这么配置，首先我们要**回到**web.xml加载顺序介绍。web.xml基础节点的加载顺序是context-param > listener > filter > serlvet。

##### context-param节点

context-param节点中的键值对首先会被容器读取并存放到ServletContext对象中，这里面的键值对信息被整个web项目共享。

如何读取context-param中的值呢？我们只要获得ServletContext对象，然后再调用它的getInitParameter提供键名来获得方法就可以获得。在哪些地方能获得SerlvetContext对象呢？我们继承HttpServlet的Serlvet中去调用getServletContext就能获取ServletContext对象；这里我首先来编写一个简单Java web项目来读取相关的参数，首先是配置文件：
```XML
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
    <context-param>
        <param-name>test-param</param-name>
        <param-value>this is a test-param string</param-value>
    </context-param>

   <servlet>
        <servlet-name>simple-web-app</servlet-name>
        <servlet-class>
            servlet.SimpleServlet
        </servlet-class>
        </servlet>
    <servlet-mapping>
        <servlet-name>simple-web-app</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
```
在这个配置文件中，我只定义了最简单的两个节点context-param与servlet，对于context-param节点，我定义了"test-param" : "this is a test-param string"键值对，然后定义了servlet相关配置（这一部分我认为你已经明白了）。最后，SimpleServlet中的代码如下：
```JAVA
public class SimpleServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
        throws ServletException, IOException {
        doPost(req, resp); // 将所有的Get请求转移到Post请求方法
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) 
        throws ServletException, IOException {
        ServletContext context = this.getServletContext();
        String param = context.getInitParameter("test-param");
        resp.getWriter().print(param);
    }
}
```
在doPost方法中，我们按照前面说的方式获取ServletContext对象，然后再通过getInitParameter方法获取"test-param"键对应的字符串，然后输出到网页上：
![print.png](https://res.zhen.blog/images/post/2018-08-26-JavaWeb/print.png)

##### listener节点

web.xml第二步加载就是listener节点，该节点的形式如下：
```XML
<listener>
    <listener-class>
        listener.MyInitListener
    </listener-class>
</listener>
```
而MyInitListener类需要实现ServletContextListener接口，并重写其中的方法：
```JAVA
public class MyInitListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println(new Date() + " contextInitialized....");
    }

    @Override
    public void contextDestroyed(ServletContextEvent sce) {

    }
}
```
当web.xml在加载监听器节点之后，会创建我们定义的对应的MyInitListener对象，并且会执行其中的contextInitialized方法，在整个项目关闭的时候会执行contextDestroyed方法。上面我们编写了contextInitialized方法中的内容，然后启动项目，会在控制台看到输出：
```
Sun Aug 26 13:02:19 CST 2018 contextInitialized....
```

当然，我们也可以ServletContextEvent.getServletContext().getInitParam("xxx")来获取context-param了：
```JAVA
@Override
public void contextInitialized(ServletContextEvent sce) {
    System.out.println(new Date() + "contextInitialized....");
    System.out.println(sce.getServletContext().getInitParameter("test-param") + " in contextInitialized()");
    // OUTPUT
    // Sun Aug 26 13:19:16 CST 2018 contextInitialized....
    // this is a test-param string in contextInitialized()
}
```

#### 回到Spring MVC配置

回到Spring。为什么想要使用spring mvc，需要设置param-name=contextConfigurationLocation，param-value=xxxx-spring（非mvc部分配置）.xml；然后设置listener节点中监听器类是org.springframework.web.context.ContextLoaderListener。其实就比较好理解了，因为在这一步通过首先将spring基础上下文的配置文件通过ContextLoaderListener监听器去加载，然后读取出基础spring IOC以及AOP部分配置，将一些基础组件加载到spring的bean容器中，在接下来的无论是filter节点中的对象还是serlvet节点中的对象需要依赖注入的部分，都已经通过加载context-param中对于spring基础配置文件定位加载好了，后续spring相关的bean的创建以及注入等都交给了spring来进行管理。

同时在加载serlvet时候，因为所有的请求都交给了DispatcherSevlet，且指定了mvc.xml配置文件的路径参数，所以，我们在这个mvc.xml中去设置静态资源的处理规则以及试图和控制器等处理规则。



