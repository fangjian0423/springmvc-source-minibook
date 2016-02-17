# DispatcherServlet初始化过程

在分析DispatcherServlet之前，我们先看下DispatcherServlet的继承关系。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet01.png)

HttpSerlvetBean继承自HttpServlet。

HttpServletBean覆写了init方法，对初始化过程做了一些处理。 我们来看下init方法到底做了什么：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet02.png)


    <servlet>
      <servlet-name>dispatcher</servlet-name>  
      <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>  
      <load-on-startup>1</load-on-startup>  
      <init-param>
        <param-name>contextConfigLocation</param-name>  
        <param-value>classpath:springConfig/dispatcher-servlet.xml</param-value>  
      </init-param>
    </servlet>

    <servlet-mapping>
      <servlet-name>dispatcher</servlet-name>  
      <url-pattern>/</url-pattern>  
    </servlet-mapping>

比如上面这段配置，传递了contextConfigLocation参数，之后构造BeanWrapper，这里使用BeanWrapper，有2个理由：

1.contextConfigLocation属性在FrameworkServlet中定义，HttpServletBean中未定义       
2.利用Spring的注入特性，只需要调用setPropertyValues方法就可将contextConfigLocation属性设置到对应实例中，也就是以依赖注入的方式初始化属性。

然后设置DispatcherServlet中的contextConfigLocation属性(FrameworkServlet中定义)为web.xml中读取的contextConfigLocation参数，该参数用于构造SpringMVC容器上下文。

下面看下FrameworkServlet这个类，FrameworkServlet继承自HttpServletBean。

首先来看下该类覆写的initServletBean方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet03.png)

接下来看下initWebApplicationContext方法的具体实现逻辑：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet04.png)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet05.png)

这里的根上下文是web.xml中配置的ContextLoaderListener监听器中根据contextConfigLocation路径生成的上下文。

    <context-param>
      <param-name>contextConfigLocation</param-name>  
      <param-value>classpath:springConfig/applicationContext.xml</param-value>  
    </context-param>
    <listener>
      <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>  
    </listener>

比如这段配置文件中根据classpath:springConfig/applicationContext.xml下的xml文件生成的根上下文。



最后看下DispatcherServlet。

DispatcherServlet覆写了FrameworkServlet中的onRefresh方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet06.png)

很明显，initStrategies方法内部会初始化各个策略接口的实现类。

比如异常处理初始化initHandlerExceptionResolvers方法：[SpringMVC异常处理机制详解](SpringMVC-exception-analysis.md)

视图处理初始化initViewResolvers方法：[SpringMVC视图机制详解](SpringMVC-view-viewResolver.md)

请求映射处理初始化initHandlerMappings方法：[详解SpringMVC请求的时候是如何找到正确的Controller](SpringMVC-request-mapping.md)

总结一下各个Servlet的作用：

1.HttpServletBean

**主要做一些初始化的工作，将web.xml中配置的参数设置到Servlet中。比如servlet标签的子标签init-param标签中配置的参数。**

2.FrameworkServlet
　**将Servlet与Spring容器上下文关联。其实也就是初始化FrameworkServlet的属性webApplicationContext，这个属性代表SpringMVC上下文，它有个父类上下文，既web.xml中配置的ContextLoaderListener监听器初始化的容器上下文。**

3.DispatcherServlet

**初始化各个功能的实现类。比如异常处理、视图处理、请求映射处理等。**


#DispatcherServlet处理请求过程

在分析DispatcherServlet处理请求过程之前，我们回顾一下Servlet对于请求的处理。

HttpServlet提供了service方法用于处理请求，service使用了模板设计模式，在内部对于httpget方法会调用doGet方法，httppost方法调用doPost方法...........

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet07.png)

进入processRequest方法看下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet08.png)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet09.png)

其中注册的监听器类型为ApplicationListener接口类型。

继续看DispatcherServlet覆写的doService方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-dispatcherServlet10.png)

最终就是doDispatch方法。

doDispatch方法功能简单描述一下：

首先根据请求的路径找到HandlerMethod(带有Method反射属性，也就是对应Controller中的方法)，然后匹配路径对应的拦截器，有了HandlerMethod和拦截器构造个HandlerExecutionChain对象。HandlerExecutionChain对象的获取是通过HandlerMapping接口提供的方法中得到。有了HandlerExecutionChain之后，通过HandlerAdapter对象进行处理得到ModelAndView对象，HandlerMethod内部handle的时候，使用各种HandlerMethodArgumentResolver实现类处理HandlerMethod的参数，使用各种HandlerMethodReturnValueHandler实现类处理返回值。 最终返回值被处理成ModelAndView对象，这期间发生的异常会被HandlerExceptionResolver接口实现类进行处理。

# 总结

本章分析了SpringMVC入口Servlet -> DispatcherServlet的作用，其中分析了父类HttpServletBean以及FrameworkServlet的作用。

SpringMVC的设计与Struts2完全不同，Struts2采取的是一种完全和Web容器隔离和解耦的机制，而SpringMVC就是基于最基本的request和response进行设计。

# 参考资料

[http://my.oschina.net/lichhao/blog/102315](http://my.oschina.net/lichhao/blog/102315)
[http://my.oschina.net/lichhao/blog/104943](http://my.oschina.net/lichhao/blog/104943)
[http://jinnianshilongnian.iteye.com/blog/1602617](http://jinnianshilongnian.iteye.com/blog/1602617)
