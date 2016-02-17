# SpringMVC源码分析迷你书

[GITHUB仓库](https://github.com/fangjian0423/springmvc-source-minibook)

[在线阅读](https://fangjian0423.gitbooks.io/springmvc-source-minibook/content)

一本分析SpringMVC部分源码的mini书，基本上是从之前的博客中拷贝过来的，博客地址：http://www.cnblogs.com/fangjian0423/p/springMVC-directory-summary.html

这个mini书比较适合想深入学习SpringMVC的读者。

这本书的文章都是基于Spring4.0.2版本的。

文章阅读顺序如下：

1. [SpringMVC入门](SpringMVC-introduction.md)

  SpringMVC的入门文章， 对于某些没接触过SpringMVC的同学来说，可以阅读以下，了解这个框架的结构以及使用，以入门的同学可以选择不看~

2. [核心分发器DispatcherServlet分析](SpringMVC-dispatcherServlet.md)

  SpringMVC入口Servlet -> DispatcherServlet的分析，解释了DispatcherServlet的父类FrameworkServlet以及FrameworkServlet的父类HttpServletBean的作用

3. [解析SpringMVC针对请求是如何找到正确的Controller](SpringMVC-request-mapping.md)

  想知道http请求是如何找到对应Controller中的方法的吗，这个过程当中到底发生了什么，你知道吗？ 本篇将为你一一解答。

4. [解析SpringMVC中Controller的方法中参数的工作原理](SpringMVC-request-param-analysis.md)

  本文分析Controller中方法的参数是如何被注入进来的

5. [解析SpringMVC关于json、xml自动转换的原理](SpringMVC-xml-json-convert.md)

  通过json、xml的自动转换巩固第四篇文章的知识，自动转换由RequestResponseBodyMethodProcessor处理，该类实现了HandlerMethodArgumentResolver和HandlerMethodReturnValueHandler接口。

6. [解析SpringMVC类型转换、数据绑定](SpringMVC-databind-typeconvert.md)

  介绍了属性编辑器的概念以及Spring对属性编辑器的支持，本文知识消化之后可以回过头看第4篇中FormObjArgumentResolver的实现。

7. [解析SpringMVC拦截器](SpringMVC-interceptor.md)

  解释了SpringMVC拦截器的设计原理。

8. [解析SpringMVC视图机制](SpringMVC-view-viewResolver.md)

  分析了SpringMVC的视图机制，主要也就是讲解View和ViewResolver这两个接口的作用。

9. [解析SpringMVC异常处理机制](SpringMVC-exception-analysis.md)

  解释了SpringMVC异常机制的设计原理。

SpringMVC是Spring大框架中的一个子框架，所以内部肯定用到了Spring框架的一些依赖，所以本书也会将一些Spring框架的内容，目前只写了1篇文章。

  1. [Spring中Ordered接口简介](spring/Spring-Ordered-interface.md)

文中难免会有错误，欢迎读者指出。
