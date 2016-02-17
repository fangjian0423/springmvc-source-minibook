# 前言

本章将分析SpringMVC的视图这部分内容，让读者了解SpringMVC视图的设计原理。

# 重要接口和类介绍

1.**View接口**

视图基础接口，它的各种实现类是无状态的，因此是线程安全的。 该接口定义了两个方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver01.png)

2.**AbstractView抽象类**

View接口的基础实现类。我们稍微介绍一下这个抽象类。

首先看下这个类的属性：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver02.png)

再看下抽象类中接口方法的实现：

getContentType方法直接返回contentType属性即可。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver03.png)

render方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver04.png)


3.**AbstractUrlBasedView抽象类**

继承自AbstractView抽象类，增加了1个类型为String的url参数。

4.**InternalResourceView类**

继承自AbstractUrlBasedView抽象类的类，表示JSP视图。

我们看下这个类的renderMergedOutputModel方法(AbstractView抽象类定义的抽象方法，为View接口提供的render方法服务)。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver05.png)

5.**JstlView类**

JSTL视图，继承自InternalResourceView，该类大致上与InternalResourceView类一致。

6.**AbstractTemplateView抽象类**


继承自AbstractUrlBasedView抽象类，重写了renderMergedOutputModel方法，在该方法中会调用renderMergedTemplateModel方法，renderMergedTemplateModel方法为新定义的抽象方法。

该抽象类有几个boolean属性exposeSessionAttributes，exposeRequestAttributes。设置为true的话会将request和session中的键值和值丢入到renderMergedTemplateModel方法中的model这个Map参数中。

这个类是某些模板引擎视图类的父类。 比如FreemarkerView，VelocityView。

7.**FreeMarkerView类**

继承自AbstractTemplateView抽象类。

直接看renderMergedTemplateModel方法，renderMergedTemplateModel内部会调用doRender方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver06.png)

8.**RedirectView类**

继承自AbstractUrlBasedView，并实现SmartView接口。SmartView接口定义了1个boolean isRedirectView();方法。

该视图的renderMergedOutputModel方法主要就是通过response.sendRedirect进行重定向。

9.**ViewResolver接口**

视图解释器，用来解析视图View，与View接口配合使用。

该接口只有1个方法，通过视图名称viewName和Locale对象得到View接口实现类：

    View resolveViewName(String viewName, Locale locale) throws Exception;

10.**AbstractCachingViewResolver抽象类**

带有缓存功能的ViewResolver接口基础实现抽象类，该类有个属性名为viewAccessCache的以 "viewName_locale" 为key， View接口为value的Map。

该抽象类实现的resolveViewName方法内部会调用createView方法，方法内部会调用loadView抽象方法。

11.**UrlBasedViewResolver类**

继承自AbstractCachingViewResolver抽象类、并实现Ordered接口的类，是ViewResolver接口简单的实现类。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver07.png)

该类复写了createView方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver08.png)

父类(AbstractCachingViewResolver)的createView方法内部会调用loadView抽象方法，UrlBasedViewResolver实现了这个抽象方法：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver09.png)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver10.png)

下面对UrlBasedViewResolver做1个test，配置如下：

    <bean class="org.springframework.web.servlet.view.UrlBasedViewResolver">
      <property name="prefix" value="/WEB-INF/view/"/>
      <property name="suffix" value=".jsp"/>
      <property name="viewClass" value="org.springframework.web.servlet.view.InternalResourceView"/>
      <property name="viewNames">
        <array>  
          <value type="java.lang.String">*</value>  
        </array>    
      </property>  
      <property name="contentType" value="text/html;charset=utf-8"/>
      <property name="attributesMap">
        <map>
          <entry key="mytest" value="mytestvalue"/>
        </map>
      </property>
      <property name="attributes">
        <props>
          <prop key="test">testvalue</prop>
        </props>
      </property>
    </bean>

我们看到：以InternalResourceView这个JSP视图作为视图；viewNames我们设置了*，这里的*代表全部视图名(这个viewNames属性不设置也可以，代表全部视图名都处理)；http响应头部contentType信息：text/html;charset=utf-8；attributesMap和attributes传入的Map和Properties参数都会被丢入到staticAttributes属性中，这个staticAttributes会被设置成AbstractView的staticAttributes属性，也就是request域中的参数。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver11.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver12.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver13.png)

我们看到request域中没有设置mytest和testvalue值。但是页面中会显示，因为我们配置了attributesMap和attributes参数。

如果我们把viewNames中的"*"改成"index1"。那么就报错了，因为处理视图名的时候index匹配不上index1。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver14.png)

12.**InternalResourceViewResolver类**

继承自UrlBasedViewResolver，以InternalResourceView作为视图，若项目中存在“javax.servlet.jsp.jstl.core.Config”该类，那么会以JstlView作为视图。重写了buildView方法，主要就是为了给InternalResourceView视图设置属性。

13.**AbstractTemplateViewResolver类**

继承自UrlBasedViewResolver，重写了buildView方法，主要就是构造AbstractTemplateView以及为它设置相应的属性。

14.**FreeMarkerViewResolver类**

继承自AbstractTemplateViewResolver，将视图设置为FreeMarkerView。

15.**ModelAndView对象**

顾名思义，带有视图和Model属性的一个模型和视图类。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver15.png)

**值得注意的是，这个视图属性是一个Object类型的数据，可以直接是View接口的实现类或者视图名(字符串)。**

# 源码分析

下面我们来分析SpringMVC处理视图的源码。

SpringMVC在处理请求的时候，通过RequestMappingHandlerMapping得到HandlerExecutionChain，然后通过RequestMappingHandlerAdapter得到1个ModelAndView对象，之后通过processDispatchResult方法处理。

processDispatchResult方法如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver16.png)

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver17.png)

如果配置的ViewResolver如下：

    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
      <property name="prefix" value="/WEB-INF/view/"/>
      <property name="suffix" value=".jsp"/>
    </bean>

那么就是使用InternalResourceViewResolver来解析视图。

之前分析过，InternalResourceViewResolver重写了UrlBasedViewResolver的buildView方法。但是还是会调用UrlBasedViewResolver的buildView方法。

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver18.png)

最终得到InternalResourceView或JstlView视图。这两个视图的render方法本章介绍重要接口及类的时候已分析。



PS：DispathcerServlet中的viewResolvers属性是个集合，如果存在多个ViewResolver对象，必定会有优先级的问题，这部分的内容请参考[Spring中Ordered接口简介](spring/Spring-Ordered-interface.md)

# 编码自定义的ViewResolver

下面，我们就来编写自定义的ViewResolver。

自定义的ViewResolver处理视图名的时候，遇到 "jsp:" 开头的会找jsp页面，遇到 "freemarker:" 开头的找freemarker页面。

    public class CustomViewResolver extends UrlBasedViewResolver {

      public static final String JSP_URL_PREFIX = "jsp:";
      public static final String FTL_URL_PREFIX = "freemarker:";

      private static final boolean jstlPresent = ClassUtils.isPresent(
                "javax.servlet.jsp.jstl.core.Config", CustomViewResolver.class.getClassLoader());

      private Boolean exposePathVariables = false;

      private boolean exposeRequestAttributes = false;

      private boolean allowRequestOverride = false;

      private boolean exposeSessionAttributes = false;

      private boolean allowSessionOverride = false;

      private boolean exposeSpringMacroHelpers = true;

      public CustomViewResolver() {
          this.setViewClass(FreeMarkerView.class);
      }  

      @Override
      protected AbstractUrlBasedView buildView(String viewName) throws Exception {
        if(viewName.startsWith(FTL_URL_PREFIX)) {
            return buildFreemarkerView(viewName.substring(FTL_URL_PREFIX.length()));
        } else if(viewName.startsWith(JSP_URL_PREFIX)) {
            Class viewCls = jstlPresent ? JstlView.class : InternalResourceView.class;
            return buildView(viewCls, viewName.substring(JSP_URL_PREFIX.length()), getPrefix(), ".jsp");
        } else {
            //默认以freemarker处理
            return buildFreemarkerView(viewName);
        }
      }

      private AbstractUrlBasedView build(Class viewClass, String viewName, String prefix, String suffix) {
        AbstractUrlBasedView view = (AbstractUrlBasedView) BeanUtils.instantiateClass(viewClass);
        view.setUrl(prefix + viewName + suffix);
        String contentType = getContentType();
        if (contentType != null) {
            view.setContentType(contentType);
        }
        view.setRequestContextAttribute(getRequestContextAttribute());
        view.setAttributesMap(getAttributesMap());
        if (this.exposePathVariables != null) {
            view.setExposePathVariables(exposePathVariables);
        }
        return view;    
      }

      private AbstractUrlBasedView buildFreemarkerView(String viewName) throws Exception {
        AbstractTemplateView view = (AbstractTemplateView) build(FreeMarkerView.class, viewName, "", getSuffix());
        view.setExposeRequestAttributes(this.exposeRequestAttributes);
        view.setAllowRequestOverride(this.allowRequestOverride);
        view.setExposeSessionAttributes(this.exposeSessionAttributes);
        view.setAllowSessionOverride(this.allowSessionOverride);
        view.setExposeSpringMacroHelpers(this.exposeSpringMacroHelpers);
        return view;      
      }

      //get set方法省略

    }

xml配置：

    <bean class="org.format.demo.support.viewResolver.CustomViewResolver">
      <property name="prefix" value="/WEB-INF/view/"/>
      <property name="suffix" value=".ftl"/>
      <property name="contentType" value="text/html;charset=utf-8"/>
      <property name="exposeRequestAttributes" value="true"/>
      <property name="exposeSessionAttributes" value="true"/>
      <property name="exposeSpringMacroHelpers" value="true"/>
      <property name="requestContextAttribute" value="request"/>
    </bean>
    <bean id="freemarkerConfig" class="org.springframework.web.servlet.view.freemarker.FreeMarkerConfigurer">
      <property name="templateLoaderPath" value="/WEB-INF/view/"/>
      <property name="defaultEncoding" value="utf-8"/>
      <property name="freemarkerSettings">
        <props>
          <prop key="template_update_delay">10</prop>
          <prop key="locale">zh_CN</prop>
          <prop key="datetime_format">yyyy-MM-dd</prop>
          <prop key="date_format">yyyy-MM-dd</prop>
          <prop key="number_format">#.##</prop>
        </props>
      </property>
    </bean>

简单解释一下：CustomViewResolver解析视图名的时候，判断 "jsp:" 和 "freemarker:" 开头的名字，如果是 "jsp:" 开头的，如果有JSTL依赖，构造JSTLView视图，否则构造InternalResourceView视图。如果是 "freemarker:" 构造FreemarkerView。在构造视图之前分别会设置一些属性。

xml配置：配置prefix是为了给jsp视图用的，freemarker视图不需要prefix，因为FreemarkerView内部会使用配置的FreeMarkerConfigurer，并用FreeMarkerConfigurer内部的templateLoaderPath属性作为前缀，配置的suffix是为了让FreemarkerView使用，当后缀。

最后附上Controller代码：

    @Controller
    @RequestMapping(value = "/tvrc")
    public class TestViewResolverController {

        @RequestMapping("jsp")
        public ModelAndView jsp(ModelAndView view) {
            view.setViewName("jsp:trvc/index");
            return view;
        }

        @RequestMapping("/ftl")
        public ModelAndView freemarker(ModelAndView view) {
            view.setViewName("freemarker:trvc/index");
            return view;
        }

    }

视图 /WEB-INF/view/trvc/index.jsp 中的的内容是输出This is jsp page

视图 /WEB-INF/view/trvc/index.ftl 中的的内容是输出This is freemarker page

![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver19.png)
![](http://7x2wh6.com1.z0.glb.clouddn.com/springmvc-view-viewResolver20.png)

# 总结

本章分析了SpringMVC中的视图机制，View和ViewResolver这两个接口是视图机制的核心，并分析了几个重要的View和ViewResolver接口实现类，最终写了一个区别jsp和freemarker视图的ViewResolver实现类，让读者更加理解视图机制。

希望这篇文章能帮助读者了解SpringMVC视图机制。

# 参考资料

[http://my.oschina.net/HeliosFly/blog/221392](http://my.oschina.net/HeliosFly/blog/221392)
