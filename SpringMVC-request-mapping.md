# 前言

我们使用浏览器通过地址 http://ip:port/contextPath/path 进行访问，SpringMVC是如何得知用户到底是访问哪个Controller中的方法，这期间到底发生了什么。

本章将分析SpringMVC是如何处理请求与Controller之间的映射关系的，让读者知道这个过程中到底发生了什么事情。

# 源码分析

在分析源码之前，我们先了解一下几个东西。

1.这个过程中重要的接口和类。

**HandlerMethod**类：

Spring3.1版本之后引入的。 是一个封装了方法参数、方法注解，方法返回值等众多元素的类。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping01.jpg)

它的子类InvocableHandlerMethod有两个重要的属性WebDataBinderFactory和HandlerMethodArgumentResolverComposite，很明显是对请求进行处理的。

InvocableHandlerMethod的子类ServletInvocableHandlerMethod有个重要的属性HandlerMethodReturnValueHandlerComposite，很明显是对响应进行处理的。

ServletInvocableHandlerMethod这个类在HandlerAdapter对每个请求处理过程中，都会实例化一个出来(上面提到的属性由HandlerAdapter进行设置)，分别对请求和返回进行处理。(RequestMappingHandlerAdapter源码，实例化ServletInvocableHandlerMethod的时候分别set了上面提到的重要属性)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping02.png)

**MethodParameter**类：

HandlerMethod类中的parameters属性类型，是一个MethodParameter数组。MethodParameter是一个封装了方法参数具体信息的工具类，包括参数的的索引位置，类型，注解，参数名等信息。

HandlerMethod在实例化的时候，构造函数中会初始化这个数组，这时只初始化了部分数据，在HandlerAdapter对请求处理过程中会完善其他属性，之后交予合适的HandlerMethodArgumentResolver接口处理。

以类DeptController为例：

    @Controller
    @RequestMapping(value = "/dept")
    public class DeptController {

      @Autowired
      private IDeptService deptService;

      @RequestMapping("/update")
      @ResponseBody
      public String update(Dept dept) {
        deptService.saveOrUpdate(dept);
        return "success";
      }

    }

(刚初始化时的数据)　　

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping03.png)

(HandlerAdapter处理后的数据)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping04.png)

**RequestCondition**接口：

Spring3.1版本之后引入的。 **是SpringMVC的映射基础中的请求条件，可以进行combine,compareTo，getMatchingCondition操作。这个接口是映射匹配的关键接口，其中getMatchingCondition方法关乎是否能找到合适的映射。**

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping05.png)

**RequestMappingInfo类**：

Spring3.1版本之后引入的。 是一个封装了各种请求映射条件并实现了RequestCondition接口的类。

有各种RequestCondition实现类属性，patternsCondition，methodsCondition，paramsCondition，headersCondition，consumesCondition以及producesCondition，这个请求条件看属性名也了解，分别代表http请求的路径模式、方法、参数、头部等信息。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping06.png)

**RequestMappingHandlerMapping类**：

处理请求与HandlerMethod映射关系的一个类。

2.Web服务器启动的时候，SpringMVC到底做了什么。

先看AbstractHandlerMethodMapping的initHandlerMethods方法中。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping07.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping08.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping09.png)

我们进入createRequestMappingInfo方法看下是如何构造RequestMappingInfo对象的。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping10.png)

PatternsRequestCondition构造函数：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping11.png)

类对应的RequestMappingInfo存在的话，跟方法对应的RequestMappingInfo进行combine操作。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping12.png)

然后使用符合条件的method来注册各种HandlerMethod。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping13.png)

下面我们来看下各种RequestCondition接口的实现类的combine操作。

PatternsRequestCondition：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping14.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping15.png)

RequestMethodsRequestCondition：

方法的请求条件，用个set直接add即可。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping16.png)

其他相关的RequestConditon实现类读者可自行查看源码。

最终，RequestMappingHandlerMapping中两个比较重要的属性

    private final Map<T, HandlerMethod> handlerMethods = new LinkedHashMap<T, HandlerMethod>();

    private final MultiValueMap<String, T> urlMap = new LinkedMultiValueMap<String, T>();

T为RequestMappingInfo。

构造完成。

我们知道，SpringMVC的分发器DispatcherServlet会根据浏览器的请求地址获得HandlerExecutionChain。

这个过程我们看是如何实现的。

首先看HandlerMethod的获得(直接看关键代码了)：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping17.png)

这里的比较器是使用RequestMappingInfo的compareTo方法(RequestCondition接口定义的)。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping18.png)

然后构造HandlerExecutionChain加上拦截器

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping19.png)

# 实例

写了这么多，来点例子让我们验证一下吧。

    @Controller
    @RequestMapping(value = "/wildcard")
    public class TestWildcardController {

      @RequestMapping("/test/**")
      @ResponseBody
      public String test1(ModelAndView view) {
        view.setViewName("/test/test");
        view.addObject("attr", "TestWildcardController -> /test/**");
        return view;
      }

      @RequestMapping("/test/*")
      @ResponseBody
      public String test2(ModelAndView view) {
        view.setViewName("/test/test");
        view.addObject("attr", "TestWildcardController -> /test*");
        return view;
      }

      @RequestMapping("test?")
      @ResponseBody
      public String test3(ModelAndView view) {
        view.setViewName("/test/test");
        view.addObject("attr", "TestWildcardController -> test?");
        return view;
      }

      @RequestMapping("test/*")
      @ResponseBody
      public String test4(ModelAndView view) {
        view.setViewName("/test/test");
        view.addObject("attr", "TestWildcardController -> test/*");
        return view;
      }

    }

由于这里的每个pattern都带了*因此，都不会加入到urlMap中，但是handlerMethods还是有的。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping20.png)

当我们访问：http://localhost:8888/SpringMVCDemo/wildcard/test1 的时候。

会先根据 "/wildcard/test1"找urlMap对应的RequestMappingInfo集合，找不到的话取handlerMethods集合中所有的key集合(也就是RequestMappingInfo集合)。

然后进行匹配，匹配根据RequestCondition的getMatchingCondition方法。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping21.png)

最终匹配到2个RequestMappingInfo：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping22.png)

然后会使用比较器进行排序。

之前也分析过，比较器是有优先级的。

我们看到，RequestMappingInfo除了pattern，其他属性都是一样的。

我们看下PatternsRequestCondition比较的逻辑：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping23.png)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping24.png)

因此，/test*的通配符比/test?的多，因此，最终选择了/test?

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping25.png)

直接比较优先于通配符。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping26.png)

    @Controller
    @RequestMapping(value = "/priority")
    public class TestPriorityController {

      @RequestMapping(method = RequestMethod.GET)
      @ResponseBody
      public String test1(ModelAndView view) {
        view.setViewName("/test/test");
        view.addObject("attr", "其他condition相同，带有method属性的优先级高");
        return view;
      }

      @RequestMapping()
      @ResponseBody
      public String test2(ModelAndView view) {
        view.setViewName("/test/test");
        view.addObject("attr", "其他condition相同，不带method属性的优先级高");
        return view;
      }

    }

这里例子，其他requestCondition都一样，只有RequestMethodCondition不一样。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping27.png)

看出，方法多的优先级越多。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping28.png)

至于其他的RequestCondition，大家自行查看源码吧。

# 资源文件映射

以上分析均是基于Controller方法的映射(RequestMappingHandlerMapping)。

SpringMVC中还有静态文件的映射，SimpleUrlHandlerMapping。

DispatcherServlet找对应的HandlerExecutionChain的时候会遍历属性handlerMappings，这个一个实现了HandlerMapping接口的集合。

由于我们在*-dispatcher.xml中加入了以下配置：

    <mvc:resources location="/static/" mapping="/static/**"/>

Spring解析配置文件会使用ResourcesBeanDefinitionParser进行解析的时候，会实例化出SimpleUrlHandlerMapping。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping29.jpg)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping30.jpg)

其中注册的HandlerMethod为ResourceHttpRequestHandler。

访问地址：http://localhost:8888/SpringMVCDemo/static/js/jquery-1.11.0.js

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping31.jpg)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-request-mapping32.jpg)

地址匹配到/static/**。

最终SimpleUrlHandlerMapping找到对应的Handler -> ResourceHttpRequestHandler。

ResourceHttpRequestHandler进行handleRequest的时候，直接输出资源文件的文本内容。

# 总结

大致上整理了一下SpringMVC对请求的处理，包括其中比较关键的类和接口，希望对读者有帮助。

也让读者对SpringMVC有了更深入的认识，也为之后分析数据绑定，拦截器、HandlerAdapter等打下基础。

# 参考资料

[http://jinnianshilongnian.iteye.com/blog/1684403](http://jinnianshilongnian.iteye.com/blog/1684403)

[http://my.oschina.net/HeliosFly/blog/212329](http://my.oschina.net/HeliosFly/blog/212329)
