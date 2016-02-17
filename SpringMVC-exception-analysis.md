# 前言

本章将分析SpringMVC的异常处理内容，让读者了解SpringMVC异常处理的设计原理。

# 重要接口和类介绍

1.**HandlerExceptionResolver接口**

SpringMVC异常处理核心接口。该接口定义了1个解析异常的方法：

    ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception;

2.**AbstractHandlerExceptionResolver抽象类**

实现了HandlerExceptionResolver和Ordered接口的抽象类。

先看下属性：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis01.png)

再看下接口的实现：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis02.png)

3.**AbstractHandlerMethodExceptionResolver抽象类**

继承AbstractHandlerExceptionResolver抽象类的抽象类。 该类主要就是为HandlerMethod类服务，既handler参数是HandlerMethod类型。

该类重写了shouldApplyTo方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis03.png)

doResolveException抽象方法的实现中调用了doResolveHandlerMethodException方法，该方法也是1个抽象方法。

4.**ExceptionHandlerExceptionResolver类**

继承自AbstractHandlerMethodExceptionResolver，**该类主要处理Controller中用@ExceptionHandler注解定义的方法。该类也是annotation-driven标签配置中定义的HandlerExceptionResolver实现类之一，大多数异常处理都是由该类操作。**

该类比较重要，我们来详细讲解一下。

首先我们看下这个类的属性：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis04.png)

再来看下该类的doResolveHandlerMethodException抽象方法的实现：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis05.png)

默认的HandlerMethodArgumentResolver集合：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis06.png)

默认的HandlerMethodReturnValueHandler集合：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis07.png)

其中关于HandlerMethodArgumentResolver、HandlerMethodReturnValueHandler接口，HandlerMethodArgumentResolverComposite、HandlerMethodReturnValueHandlerComposite类，HandlerMethod、ServletInvocableHandlerMethod类等相关知识的请参考[解析SpringMVC中Controller的方法中参数的工作原理](SpringMVC-request-param-analysis.md)


我们进入getExceptionHandlerMethod方法看看是如何得到ServletInvocableHandlerMethod的：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis08.png)

我们看到getExceptionHandlerMethod中会实例化ExceptionHandlerMethodResolver，我们看看这个类到底是什么东西？

**ExceptionHandlerMethodResolver是一个会在Class及Class的父类集合中找出带有@ExceptionHandler注解的类，该类带有key为Throwable，value为Method的缓存属性。**

ExceptionHandlerMethodResolver的构造过程：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis09.png)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis10.png)

**ExceptionHandlerExceptionResolver处理过程总结一下：根据用户调用Controller中相应的方法得到HandlerMethod，之后构造ExceptionHandlerMethodResolver，构造ExceptionHandlerMethodResolver有2种选择**

1.通过HandlerMethod拿到Controller，找出Controller中带有@ExceptionHandler注解的方法(局部)

2.找到@ControllerAdvice注解配置的类中的@ExceptionHandler注解的方法(全局)。

**这2种方式构造的ExceptionHandlerMethodResolver中都有1个key为Throwable，value为Method的缓存。之后通过发生的异常找出对应的Method，然后调用这个方法进行处理。这里异常还有个优先级的问题，比如发生的是NullPointerException，但是声明的异常有Throwable和Exception，这时候ExceptionHandlerMethodResolver找Method的时候会根据异常的最近继承关系找到继承深度最浅的那个异常，即Exception。**

5.**DefaultHandlerExceptionResolver类**

继承自AbstractHandlerExceptionResolver抽象类。**annotation-driven标签配置中定义的HandlerExceptionResolver实现类之一。**

该类的doResolveException方法中主要对一些特殊的异常进行处理，比如NoSuchRequestHandlingMethodException、HttpRequestMethodNotSupportedException、HttpMediaTypeNotSupportedException、HttpMediaTypeNotAcceptableException等。

6.**ResponseStatusExceptionResolver类**

继承自AbstractHandlerExceptionResolver抽象类。**annotation-driven标签配置中定义的HandlerExceptionResolver实现类之一。**

该类的doResolveException方法主要在异常及异常父类中找到@ResponseStatus注解，然后使用这个注解的属性进行处理。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis11.png)

说明一下为什么ExceptionHandlerExceptionResolver、DefaultHandlerExceptionResolver、ResponseStatusExceptionResolver是annotation-driven标签配置中定义的HandlerExceptionResolver实现类。

我们看下annotation-driven标签配置解析类AnnotationDrivenBeanDefinitionParser中部分代码片段：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis12.png)

这3个ExceptionResolver最终被会加入到DispatcherServlet中的handlerExceptionResolvers集合中。

其中ExceptionHandlerExceptionResolver优先级最高，ResponseStatusExceptionResolver第二，DefaultHandlerExceptionResolver第三。

为什么ExceptionHandlerExceptionResolver优先级最高，因为order属性值最低，这部分的知识请参考：[Spring中Ordered接口简介](spring/Spring-Ordered-interface.md)

7.**@ResponseStatus注解**

让1个方法或异常有状态码(status)和理由(reason)返回。这个状态码是http响应的状态码。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis13.png)

8.**SimpleMappingExceptionResolver类**

继承自AbstractHandlerExceptionResolver抽象类，是1个根据配置进行解析异常的类，包括配置异常类型，默认的错误视图，默认的响应码，异常映射等配置属性。 本章不分析，有兴趣的读者可自行查看源码。

# 源码分析

下面我们来分析SpringMVC处理异常的源码。

SpringMVC在处理请求的时候，通过RequestMappingHandlerMapping得到HandlerExecutionChain，然后通过RequestMappingHandlerAdapter得到1个ModelAndView对象，这在之前发生的异常都会被catch到，然后得到这个异常并作为参数传入到processDispatchResult方法处理。

processDispatchResult方法如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis14.png)

processHandlerException方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis15.png)

# 实例讲解

接下里讲常用ExceptionResolver的实例。

**1.ExceptionHandlerExceptionResolver**

    @Controller
    @RequestMapping(value = "/error")
    public class TestErrorController {

        @RequestMapping("/exception")
        public ModelAndView exception(ModelAndView view) throws ClassNotFoundException {
            view.setViewName("index");
            throw new ClassNotFoundException("class not found");
        }

        @RequestMapping("/nullpointer")
        public ModelAndView nullpointer(ModelAndView view) {
            view.setViewName("index");
            String str = null;
            str.length();
            return view;
        }  

        @ExceptionHandler(RuntimeException.class)
        public ModelAndView error(RuntimeException error, HttpServletRequest request) {
            ModelAndView mav = new ModelAndView();
            mav.setViewName("error");
            mav.addObject("param", "Runtime error");
            return mav;
        }  

        @ExceptionHandler()
        public ModelAndView error(Exception error, HttpServletRequest request, HttpServletResponse response) {
            ModelAndView mav = new ModelAndView();
            mav.setViewName("error");
            mav.addObject("param", "Exception error");
            return mav;
        }  

        /**
        @ExceptionHandler(NullPointerException.class)
        public ModelAndView error(ModelAndView mav) {
            mav.setViewName("error");
    　　　　mav.addObject("param", "NullPointer error");
            return mav;
        }*/

    }

分析一下：

如果用户进入nullpointer方法，str对象还未初始化，会发生NullPointerException。如果去掉最后1个注释掉的error方法，那么会报错。因为ExceptionHandlerExceptionResolver的默认HandlerMethodArgumentResolver中只有ServletRequestMethodArgumentResolver和ServletResponseMethodArgumentResolver(所以其他2个error方法中的request和response参数没问题)。 所以我们给最后1个error方法加了注释。

由于TestErrorController控制器中有2个带有@ExceptionHandler注解的方法，之前分析的ExceptionHandlerMethodResolver构造过程中，会构造ExceptionHandlerMethodResolver，ExceptionHandlerMethodResolver内部会有1个key分别为RuntimeException和Exception，value分别为第一个和第二个error方法的缓存。由于NullPointerException的继承关系离RuntimeException比Exception近，因此最终进入了第一个error方法。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis16.png)

如果用户进入exception方法，同理。ClassNotFoundException继承自Exception，跟RuntimeException没关系，那么进入第二个error方法。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis17.png)

说明一下，两个error方法返回值都是ModelAndView，这是因为ExceptionHandlerMethodResolver的默认HandlerMethodReturnValueHandler中有ModelAndViewMethodReturnValueHandler。还有其他的比如ModelMethodProcessor、ViewMethodReturnValueHandler和ViewNameMethodReturnValueHandler等。这3个分别代表返回值Model，View和字符串。 有兴趣的读者可自行查看源码。

上个例子是基于Controller的@ExceptionHandler注解方法，每个Controller都需要写@ExceptionHandler注解方法(写个BaseController可不用每个Controller都写单独的@ExceptionHandler注解方法)。

ExceptionHandlerMethodResolver内部找不到Controller的@ExceptionHandler注解的话，会找@ControllerAdvice中的@ExceptionHandler注解方法。

因此，我们也可以写1个ControllerAdvice。

    @ControllerAdvice
    public class ExceptionControllerAdvice {

        @ExceptionHandler(Throwable.class)
        @ResponseBody
        public Map<String, Object> ajaxError(Throwable error, HttpServletRequest request, HttpServletResponse response) {
            Map<String, Object> map = new HashMap<String, Object>();
            map.put("error", error.getMessage());
            map.put("result", "error");
            return map;
        }

    }

此类中的error方法代表着全局异常处理方法。

该方法可对ajax操作进行异常处理，我们返回值使用了@ResponseBody进行了处理，然后配置json消息转换器即可，这样该方法响应给客户端的数据就变成了json数据。

这方面的知识请参考：[解析SpringMVC关于json、xml自动转换的原理](SpringMVC-xml-json-convert.md)

**2.ResponseStatusExceptionResolver**

先定义1个自定义异常：

    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public class UnauthorizedException extends RuntimeException {

    }

Controller代码：

    @Controller
    @RequestMapping(value = "/error")
    public class TestErrorController {

        @RequestMapping("/unauth")
        public ModelAndView unauth(ModelAndView view) {
            view.setViewName("index");
            throw new UnauthorizedException();
        }

    }

由于该类没有写@ExceptionHandler注解，因此ExceptionHandlerExceptionResolver不能解析unauth触发的异常。接下来由ResponseStatusExceptionResolver进行解析，由于触发的异常UnauthorizedException带有@ResponseStatus注解。因此会被ResponseStatusExceptionResolver解析到。最后响应HttpStatus.UNAUTHORIZED代码给客户端。HttpStatus.UNAUTHORIZED代表响应码401，无权限。 关于其他的响应码请参考HttpStatus枚举类型源码。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis18.png)

**3.DefaultHandlerExceptionResolver**

直接上代码：

    @Controller
    @RequestMapping(value = "/error")
    public class TestErrorController {

        @RequestMapping("/noHandleMethod")
        public ModelAndView noHandleMethod(ModelAndView view, HttpServletRequest request) throws NoSuchRequestHandlingMethodException {
            view.setViewName("index");
            throw new NoSuchRequestHandlingMethodException(request);
        }

    }

用户进入noHandleMethod方法触发NoSuchRequestHandlingMethodException异常，由于没配置@ExceptionHandler以及该异常没有@ResponseStatus注解，最终由DefaultHandlerExceptionResolver解析，由于NoSuchRequestHandlingMethodException属于DefaultHandlerExceptionResolver解析的异常，因此被DefaultHandlerExceptionResolver解析。NoSuchRequestHandlingMethodException会发生404错误。

关于DefaultHandlerExceptionResolver可以处理的其他异常，请参考DefaultHandlerExceptionResolver源码。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis19.png)

# 扩展ExceptionHandlerExceptionResolver功能

SpringMVC提供的HandlerExceptionResolver基本上都能满足我们的开发要求，因此本章就不准备写自定义的HandlerExceptionResolver。

既然不写自定义的HandlerExceptionResolver，我们就来扩展ExceptionHandlerExceptionResolver来吧，让它支持更多的功能。

比如为ExceptionHandlerExceptionResolver添加更多的HandlerMethodArgumentResolver，ExceptionHandlerExceptionResolver默认只能有2个HandlerMethodArgumentResolver和ServletRequestMethodArgumentResolver(处理ServletRequest、WebRequest、MultipartRequest、HttpSession等参数)和ServletResponseMethodArgumentResolver(处理ServletResponse、OutputStream或Writer参数)。

　　ModelAndView这种类型的参数会被ServletModelAttributeMethodProcessor处理。因此我们需要给ExceptionHandlerExceptionResolver添加ServletModelAttributeMethodProcessor这个HandlerMethodArgumentResolver。由于ServletModelAttributeMethodProcessor处理ModelAndView参数会使用WebDataBinderFactory参数，因此我们得重写doResolveHandlerMethodException方法，所以新写了1个类继承ExceptionHandlerExceptionResolver。

    public class MyExceptionHandlerExceptionResolver extends ExceptionHandlerExceptionResolver{

        public MyExceptionHandlerExceptionResolver() {
            List<HandlerMethodArgumentResolver> list = new ArrayList<HandlerMethodArgumentResolver>();
            list.add(new ServletModelAttributeMethodProcessor(true));
            this.setCustomArgumentResolvers(list);
        }

        @Override
        protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request,
                                                               HttpServletResponse response, HandlerMethod handlerMethod, Exception exception) {
            ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
            if (exceptionHandlerMethod == null) {
                return null;
            }

            //ServletModelAttributeMethodProcessor 内部会使用传递进来的WebDataBinderFactory参数，该参数由ServletInvocableHandlerMethod提供
            exceptionHandlerMethod.setDataBinderFactory(new ServletRequestDataBinderFactory(null, null));

            exceptionHandlerMethod.setHandlerMethodArgumentResolvers(getArgumentResolvers());
            exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(getReturnValueHandlers());

            ServletWebRequest webRequest = new ServletWebRequest(request, response);
            ModelAndViewContainer mavContainer = new ModelAndViewContainer();

            try {
                if (logger.isDebugEnabled()) {
                    logger.debug("Invoking @ExceptionHandler method: " + exceptionHandlerMethod);
                }
                exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception);
            }
            catch (Exception invocationEx) {
                logger.error("Failed to invoke @ExceptionHandler method: " + exceptionHandlerMethod, invocationEx);
                return null;
            }

            if (mavContainer.isRequestHandled()) {
                return new ModelAndView();
            }
            else {
                ModelAndView mav = new ModelAndView().addAllObjects(mavContainer.getModel());
                mav.setViewName(mavContainer.getViewName());
                if (!mavContainer.isViewReference()) {
                    mav.setView((View) mavContainer.getView());
                }
                return mav;
            }
        }

    }

配置：

    <bean class="org.format.demo.support.exceptionResolver.MyExceptionHandlerExceptionResolver">
      <property name="order" value="-1"/>
    </bean>

配置完成之后，然后去掉本章实例讲解中ExceptionHandlerExceptionResolver的代码，并去掉支持NullPointerException异常的那个方法的注释。

测试如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-exception-analysis20.png)

读者可根据需求自己实现其他的扩展功能。 或者实现HandlerExceptionResolver接口新写1个HandlerExceptionResolver实现类。

新的的HandlerExceptionResolver实现类只需在配置文件中定义即可，然后配置优先级。DispatcherServlet初始化HandlerExceptionResolver的时候会自动寻找容器中实现了HandlerExceptionResolver接口的类，然后添加进来。

# 总结

分析了SpringMVC的异常处理机制并介绍了几个重要的接口和类，并分析了在annotation-driven标签中定义的3个常用的HandlerExceptionResolver。

之后又编写了1个继承自ExceptionHandlerExceptionResolver类的异常解析类，巩固了之前分析的知识。

希望这篇文章能帮助读者了解SpringMVC异常机制。

# 参考资料

[http://docs.spring.io/spring/docs/4.0.5.RELEASE/spring-framework-reference/htmlsingle/#mvc-exceptionhandlers](http://docs.spring.io/spring/docs/4.0.5.RELEASE/spring-framework-reference/htmlsingle/#mvc-exceptionhandlers)
