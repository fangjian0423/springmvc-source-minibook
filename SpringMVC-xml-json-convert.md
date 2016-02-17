# 现象

本章使用的demo基于maven，是根据入门blog的例子继续写下去的。

我们先来看一看对应的现象。 我们这里的配置文件 *-dispatcher.xml中的关键配置如下(其他常规的配置文件不在讲解，可参考本书一开始提到的入门章节)：

(视图配置省略)

    <mvc:resources location="/static/" mapping="/static/**"/>
    <mvc:annotation-driven/>
    <context:component-scan base-package="org.format.demo.controller"/>

pom中需要有以下依赖(Spring依赖及其他依赖不显示)：

    <dependency>
      <groupId>org.codehaus.jackson</groupId>
      <artifactId>jackson-core-asl</artifactId>
      <version>1.9.13</version>
    </dependency>
    <dependency>
      <groupId>org.codehaus.jackson</groupId>
      <artifactId>jackson-mapper-asl</artifactId>
      <version>1.9.13</version>
    </dependency>

这个依赖是json序列化的依赖。

ok。我们在Controller中添加一个method：

    @RequestMapping("/xmlOrJson")
    @ResponseBody
    public Map<String, Object> xmlOrJson() {
        Map<String, Object> map = new HashMap<String, Object>();
        map.put("list", employeeService.list());
        return map;
    }

直接访问地址：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert01.png)

我们看到，短短几行配置。使用@ResponseBody注解之后，Controller返回的对象 自动被转换成对应的json数据，在这里不得不感叹SpringMVC的强大。

我们好像也没看到具体的配置，唯一看到的就是*-dispatcher.xml中的一句配置：<mvc:annotation-driven/>。其实就是这个配置，导致了java对象自动转换成json对象的现象。

那么spring到底是如何实现java对象到json对象的自动转换的呢？ 为什么转换成了json数据，如果想转换成xml数据，那该怎么办？


# 源码分析

在讲解<mvc:annotation-driven/>这个配置之前，我们先了解下Spring的消息转换机制。@ResponseBody这个注解就是使用消息转换机制，最终通过json的转换器转换成json数据的。

HttpMessageConverter接口就是Spring提供的http消息转换接口。有关这方面的知识大家可以参考"参考资料"中的第二条链接，里面讲的很清楚。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert02.png)

下面开始分析<mvc:annotation-driven/>这句配置:

这句代码在spring中的解析类是：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert03.png)

在AnnotationDrivenBeanDefinitionParser源码的152行parse方法中：

分别实例化了RequestMappingHandlerMapping，ConfigurableWebBindingInitializer，RequestMappingHandlerAdapter等诸多类。

其中**RequestMappingHandlerMapping**和**RequestMappingHandlerAdapter**这两个类比较重要。

RequestMappingHandlerMapping处理请求映射的，处理@RequestMapping跟请求地址之间的关系。

RequestMappingHandlerAdapter是请求处理的适配器，也就是请求之后处理具体逻辑的执行，关系到哪个类的哪个方法以及转换器等工作，这个类是我们讲的重点，其中它的属性messageConverters是本章要讲的重点。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert04.png)

私有方法:getMessageConverters

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert05.png)

从代码中我们可以，RequestMappingHandlerAdapter设置messageConverters的逻辑：

1.如果<mvc:annotation-driven>节点有子节点message-converters，那么它的转换器属性messageConverters也由这些子节点组成。

message-converters的子节点配置如下：

    <mvc:annotation-driven>
      <mvc:message-converters>
        <bean class="org.example.MyHttpMessageConverter"/>
        <bean class="org.example.MyOtherHttpMessageConverter"/>
      </mvc:message-converters>
    </mvc:annotation-driven>

2.message-converters子节点不存在或它的属性register-defaults为true的话，加入其他的转换器：ByteArrayHttpMessageConverter、StringHttpMessageConverter、ResourceHttpMessageConverter等。

我们看到这么一段：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert06.png)

这些boolean属性是哪里来的呢，它们是AnnotationDrivenBeanDefinitionParser的静态变量。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert07.png)

其中ClassUtils中的isPresent方法如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert08.png)

看到这里，读者应该明白了为什么本章一开始在pom文件中需要加入对应的jackson依赖，为了让json转换器jackson成为默认转换器之一。

mvc:annotation-driven标签的作用读者也明白了。

下面我们看如何通过消息转换器将java对象进行转换的。

RequestMappingHandlerAdapter在进行handle的时候，会委托给HandlerMethod（具体由子类ServletInvocableHandlerMethod处理）的invokeAndHandle方法进行处理，这个方法又转接给HandlerMethodReturnValueHandlerComposite处理。

HandlerMethodReturnValueHandlerComposite维护了一个HandlerMethodReturnValueHandler列表。

**HandlerMethodReturnValueHandler是一个对返回值进行处理的策略接口，这个接口非常重要。关于这个接口的细节**请参考[解析SpringMVC中Controller的方法中参数的工作原理](SpringMVC-request-param-analysis.md)章节。然后找到对应的HandlerMethodReturnValueHandler对结果值进行处理。

最终找到RequestResponseBodyMethodProcessor这个Handler（由于使用了@ResponseBody注解）。

RequestResponseBodyMethodProcessor的supportsReturnType方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert09.png)

然后使用handleReturnValue方法进行处理：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert10.png)

我们看到，这里使用了转换器。　　

具体的转换方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert11.png)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert12.png)

至于为何是请求头部的**Accept**数据，读者可以进去debug这个**getAcceptableMediaTypes**方法看看。 我就不罗嗦了～～～

ok。至此，我们走遍了所有的流程。

现在，回过头来看。为什么一开始的demo输出了json数据？

我们来分析吧。


由于我们只配置了<mvc:annotation-driven>，因此使用spring默认的那些转换器。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert13.png)

很明显，我们看到了2个xml和1个json转换器。 **要看能不能转换，得看HttpMessageConverter接口的public boolean canWrite(Class<?> clazz, MediaType mediaType)方法是否返回true来决定的。**

我们先分析SourceHttpMessageConverter：

它的canWrite方法被父类AbstractHttpMessageConverter重写了。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert14.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert15.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert16.png)

发现SUPPORTED_CLASSES中没有Map类(本章demo返回的是Map类)，因此不支持。

下面看Jaxb2RootElementHttpMessageConverter：

这个类直接重写了canWrite方法。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert17.png)

需要有XmlRootElement注解。 很明显，Map类当然没有。

最终MappingJackson2HttpMessageConverter匹配，进行json转换。（为何匹配，请读者自行查看源码）

# 实例讲解

我们分析了转换器的转换过程之后，下面就通过实例来验证我们的结论吧。

首先，我们先把xml转换器实现。

之前已经分析，默认的转换器中是支持xml的。下面我们加上注解试试吧。

由于Map是jdk源码中的部分，因此我们用Employee来做demo。

因此，Controller加上一个方法：

    @RequestMapping("/xmlOrJsonSimple")
    @ResponseBody
    public Employee xmlOrJsonSimple() {
        return employeeService.getById(1);
    }

实体中加上@XmlRootElement注解

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert18.png)

结果如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert19.png)

我们发现，解析成了xml。

这里为什么解析成xml，而不解析成json呢？



之前分析过，消息转换器是根据class和mediaType决定的。

我们使用firebug看到：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert20.png)

我们发现Accept有xml，没有json。因此解析成xml了。



我们再来验证，同一地址，HTTP头部不同Accept。看是否正确。


    $.ajax({
        url: "${request.contextPath}/employee/xmlOrJsonSimple",
        success: function(res) {
            console.log(res);
        },
        headers: {
            "Accept": "application/xml"
        }
    });

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert21.png)

    $.ajax({
        url: "${request.contextPath}/employee/xmlOrJsonSimple",
        success: function(res) {
            console.log(res);
        },
        headers: {
            "Accept": "application/json"
        }
    });

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert22.png)

验证成功。

# 关于配置

如果不想使用mvc:annotation-driven标签中默认的RequestMappingHandlerAdapter的话，我们可以在重新定义这个bean，spring会覆盖掉默认的RequestMappingHandlerAdapter。

为何会覆盖，请参考[Spring中Ordered接口简介](spring/Spring-Ordered-interface.md)

    <bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
      <property name="messageConverters">
        <list>
          <bean class="org.springframework.http.converter.ByteArrayHttpMessageConverter"/>
          <bean class="org.springframework.http.converter.StringHttpMessageConverter"/>
          <bean class="org.springframework.http.converter.ResourceHttpMessageConverter"/>
        </list>
      </property>
    </bean>

或者如果只想换messageConverters的话。

    <mvc:annotation-driven>
      <mvc:message-converters>
        <bean class="org.example.MyHttpMessageConverter"/>
        <bean class="org.example.MyOtherHttpMessageConverter"/>
      </mvc:message-converters>
    </mvc:annotation-driven>

如果还想用其他converters的话。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert23.png)

以上是spring-mvc jar包中的converters。

这里我们使用转换xml的MarshallingHttpMessageConverter。

这个converter里面使用了marshaller进行转换

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert24.png)

我们这里使用XStreamMarshaller。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert25.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-xml-json-convert26.png)

json没有转换器，返回406.

至于xml格式的问题，大家自行解决吧。 这里用的是XStream～。

使用这种方式，pom别忘记了加入xstream的依赖：


    <dependency>
      <groupId>com.thoughtworks.xstream</groupId>
      <artifactId>xstream</artifactId>
      <version>1.4.7</version>
    </dependency>

# 总结

写了这么多，可能读者觉得有点罗嗦。 毕竟这也是自己的一些心得，希望都能说出来与读者共享。

刚接触SpringMVC的时候，发现这种自动转换机制很牛逼，但是一直没有研究它的原理，目前，算是了了一个小小心愿吧，SpringMVC还有很多内容，以后自己研究其他内容的时候还会与大家一起共享的。

# 参考资料

[http://my.oschina.net/HeliosFly/blog/205343](http://my.oschina.net/HeliosFly/blog/205343)

[http://my.oschina.net/lichhao/blog/172562](http://my.oschina.net/lichhao/blog/172562)

[http://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html](http://docs.spring.io/spring/docs/current/spring-framework-reference/html/mvc.html)
