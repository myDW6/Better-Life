SpringBoot Web开发

![image-20210119000635348](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210119000635348.png)

https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features

#### Spring MVC Auto-configuration

Spring Boot provides auto-configuration for Spring MVC that works well with most applications.

The auto-configuration adds the following features on top of Spring’s defaults:

- Inclusion of `ContentNegotiatingViewResolver` and `BeanNameViewResolver` beans.

  内容协商视图解析器 beanName视图解析器

- Support for serving static resources, including support for WebJars (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-static-content))).

- Automatic registration of `Converter`, `GenericConverter`, and `Formatter` beans.

  自动注册`Converter  `GenericConverter` `Formatter`

- Support for `HttpMessageConverters` (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-message-converters)).

- Automatic registration of `MessageCodesResolver` (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-message-codes)).

- Static `index.html` support.

- Automatic use of a `ConfigurableWebBindingInitializer` bean (covered [later in this document](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-web-binding-initializer)). 

  DataBinder负责请求数据绑定到JavaBean

If you want to keep those Spring Boot MVC customizations and make more [MVC customizations](https://docs.spring.io/spring/docs/5.3.3/reference/html/web.html#mvc) (interceptors, formatters, view controllers, and other features), you can add your own `@Configuration` class of type `WebMvcConfigurer` but **without**`@EnableWebMvc`.

使用@Configuration + `WebMvcConfigurer自定义规则`而不是@EnableWebMvc`

If you want to provide custom instances of `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, or `ExceptionHandlerExceptionResolver`, and still keep the Spring Boot MVC customizations, you can declare a bean of type `WebMvcRegistrations` and use it to provide custom instances of those components.

使用`WebMvcRegistrations`改变底层默认组件

If you want to take complete control of Spring MVC, you can add your own `@Configuration` annotated with `@EnableWebMvc`, or alternatively add your own `@Configuration`-annotated `DelegatingWebMvcConfiguration` as described in the Javadoc of `@EnableWebMvc`.

使用@EnableWebMvc` + @Configuration + `DelegatingWebMvcConfiguration`全面接管SpringMVC



#### 简单功能

https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-spring-mvc-auto-configuration



#### 静态资源处理

目录:`/static`  `/public` `/resources`  `/META-INF/resources`)  访问:项目根路径/+静态资源名

原理:静态映射/**   请求进来之后先找controller,controller不能处理的所有请求又都交给静态资源处理 静态资源也找不到则404

为了拦截器配置方便,让所有的静态资源访问都带prefix,  配置spring.mvc.static-path-pattern: /res/**   

改变静态资源路径:spring.web.resources.static-locations



#### 静态资源配置原理

SpringBoot启动默认加载自动配置类,关于web开发,静态资源相关的   WebMvcAutoConfiguration

```java
@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnWebApplication(
    type = Type.SERVLET
)
@ConditionalOnClass({Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class})
@ConditionalOnMissingBean({WebMvcConfigurationSupport.class})  //全面接管SpringMVC了容器有这个bean 自动配置就会失效
@AutoConfigureOrder(-2147483638)
@AutoConfigureAfter({DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class, ValidationAutoConfiguration.class})
```

自动配置类生效,给容器中配置了:

```java
@Bean
//兼容Rest风格
@ConditionalOnMissingBean({HiddenHttpMethodFilter.class})
@ConditionalOnProperty(
    prefix = "spring.mvc.hiddenmethod.filter",
    name = {"enabled"},
    matchIfMissing = false
)
public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
    return new OrderedHiddenHttpMethodFilter();
}
```

还有:

```java
@Configuration(
    proxyBeanMethods = false
)
@Import({WebMvcAutoConfiguration.EnableWebMvcConfiguration.class})
//WebMvcProperties.class    prefix = "spring.mvc"   
//ResourceProperties.class   prefix = "spring.resources",
//WebProperties     spring.web
@EnableConfigurationProperties({WebMvcProperties.class, ResourceProperties.class, WebProperties.class})
@Order(0)
public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
    private static final Log logger = LogFactory.getLog(WebMvcConfigurer.class);
    private final Resources resourceProperties;
    private final WebMvcProperties mvcProperties;
    private final ListableBeanFactory beanFactory;
    private final ObjectProvider<HttpMessageConverters> messageConvertersProvider;
    private final ObjectProvider<DispatcherServletPath> dispatcherServletPath;
    private final ObjectProvider<ServletRegistrationBean<?>> servletRegistrations;
    final WebMvcAutoConfiguration.ResourceHandlerRegistrationCustomizer resourceHandlerRegistrationCustomizer;

  //一个配置类只有一个有参构造器 特性: 所有参数的值都会从容器中确定
    public WebMvcAutoConfigurationAdapter(
      ResourceProperties resourceProperties,    //获取spring.resources绑定的所有的值的对象
      WebProperties webProperties,  						//spring.web
      WebMvcProperties mvcProperties, 					//spring.mvc
      ListableBeanFactory beanFactory, 					//ioc
      ObjectProvider<HttpMessageConverters> messageConvertersProvider, 		//找到HttpMessageConverters	  
      ObjectProvider<WebMvcAutoConfiguration.ResourceHandlerRegistrationCustomizer> resourceHandlerRegistrationCustomizerProvider, //资源处理器自定义器
      ObjectProvider<DispatcherServletPath> dispatcherServletPath, 
      ObjectProvider<ServletRegistrationBean<?>> servletRegistrations) //ServletRegistrationBean注册原生Servelt组件使用
    { 
        this.resourceProperties = (Resources)(resourceProperties.hasBeenCustomized() ? resourceProperties : webProperties.getResources());
        this.mvcProperties = mvcProperties;
        this.beanFactory = beanFactory;
        this.messageConvertersProvider = messageConvertersProvider;
        this.resourceHandlerRegistrationCustomizer = (WebMvcAutoConfiguration.ResourceHandlerRegistrationCustomizer)resourceHandlerRegistrationCustomizerProvider.getIfAvailable();
        this.dispatcherServletPath = dispatcherServletPath;
        this.servletRegistrations = servletRegistrations;
        this.mvcProperties.checkConfiguration();
    }

  //添加资源处理器    
   public void addResourceHandlers(ResourceHandlerRegistry registry) {
            if (!this.resourceProperties.isAddMappings()) {
              //add-mappings=false 静态资源会被禁用
                logger.debug("Default resource handling disabled");
            } else {
              //获取静态资源的缓存时间  返回code:304  浏览器cache-control = cache.period
                Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
                CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
                if (!registry.hasMappingForPattern("/webjars/**")) {
                  //webjars/**下的所有请求 都走classpath:/META-INF/resources/webjars/这个路径
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{"/webjars/**"}).addResourceLocations(new String[]{"classpath:/META-INF/resources/webjars/"}).setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl).setUseLastModified(this.resourceProperties.getCache().isUseLastModified()));
                }

              //静态资源路径    默认映射/**
                String staticPathPattern = this.mvcProperties.getStaticPathPattern();  // /**
                if (!registry.hasMappingForPattern(staticPathPattern)) {
                    this.customizeResourceHandlerRegistration(registry.addResourceHandler(new String[]{staticPathPattern}).addResourceLocations(WebMvcAutoConfiguration.getResourceLocations(this.resourceProperties.getStaticLocations()))//  如下图所示
                                                              .setCachePeriod(this.getSeconds(cachePeriod)).setCacheControl(cacheControl).setUseLastModified(this.resourceProperties.getCache().isUseLastModified()));
                }

            }
        }
  
   

}
```

![image-20210119231531710](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210119231531710.png)

![image-20210119231604716](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210119231604716.png)

#### 表单Rest请求处理原理

对表单页面的Rest方式进行处理,逻辑如下

首先,表单的method只支持get和post,该怎么处理:

组件OrderedHiddenHttpMethodFilter便是处理Rest的   extends HiddenHttpMethodFilter  

这个HiddenHttpMethodFilter里面:有个_method  要求我们在表单时使用post提交来模拟,put delete,patch 使用隐藏域

![image-20210119233625397](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210119233625397.png)

核心在这个方法

![image-20210119233722388](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210119233722388.png)

 首先拿到原生httpservletrequest,如果是post请求 且没有异常=>  获取_method的值 这个值表单会传delete/put 转为大写DElETE,PUT,PATCH  然后将request和method传入一个包装器,包装器很简单,只是包装了请求方法并重写了getMethod()

包装模式 过滤器链放行的可是包装后的request

![image-20210119234015334](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210119234015334.png)

看HttpServletRequestWrapper =>  

```java
class HttpServletRequestWrapper extends ServletRequestWrapper implements
        HttpServletRequest
  //本身实现了HttpServletRequest  所以上面又强制转型 很妙
```

但是这样并没有生效,为什么?

![image-20210119234314883](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210119234314883.png)

必须手动开启

Rest客户端工具 其实没有上述这些问题,所以这项选择开启, 前后端分离没有这种烦恼



## 请求映射原理

![image-20210120211847785](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210120211847785.png)



HttpServlet  ->   FrameworkServlet  processRequest(request, response)   

![image-20210120212133955](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210120212133955.png)

doService是抽象方法  被DispatcherServelet重写

![image-20210120212249498](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210120212249498.png)

![image-20210120212426255](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210120212426255.png)

可见每个请求最终都要到doDispatch方法中处理,SpringMVC都从DispatcherServlet的doDispatch开始分析.

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
   HttpServletRequest processedRequest = request;
   HandlerExecutionChain mappedHandler = null;
   boolean multipartRequestParsed = false;

   WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

   try {
      ModelAndView mv = null;
      Exception dispatchException = null;

      try {
         processedRequest = checkMultipart(request);
         multipartRequestParsed = (processedRequest != request);

        //找到当前请求的处理器进行处理,找到Controller的对应方法
         // Determine handler for the current request.
         mappedHandler = getHandler(processedRequest);
         if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
         }

         // Determine handler adapter for the current request.
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

         if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return;
         }

         // Actually invoke the handler.
         mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

         if (asyncManager.isConcurrentHandlingStarted()) {
            return;
         }

         applyDefaultViewName(processedRequest, mv);
         mappedHandler.applyPostHandle(processedRequest, response, mv);
      }
      catch (Exception ex) {
         dispatchException = ex;
      }
      catch (Throwable err) {
         // As of 4.3, we're processing Errors thrown from handler methods as well,
         // making them available for @ExceptionHandler methods and other scenarios.
         dispatchException = new NestedServletException("Handler dispatch failed", err);
      }
      processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
   }
   catch (Exception ex) {
      triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
   }
   catch (Throwable err) {
      triggerAfterCompletion(processedRequest, response, mappedHandler,
            new NestedServletException("Handler processing failed", err));
   }
   finally {
      if (asyncManager.isConcurrentHandlingStarted()) {
         // Instead of postHandle and afterCompletion
         if (mappedHandler != null) {
            mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
         }
      }
      else {
         // Clean up any resources used by a multipart request.
         if (multipartRequestParsed) {
            cleanupMultipart(processedRequest);
         }
      }
   }
}



@Nullable
	protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    //handlerMappings保存了处理的映射
		if (this.handlerMappings != null) {
			for (HandlerMapping mapping : this.handlerMappings) {
				HandlerExecutionChain handler = mapping.getHandler(request);
				if (handler != null) {
					return handler;
				}
			}
		}
		return null;
	}
```

![image-20210120214022504](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210120214022504.png)

RequestMappingHandlerMapping  保存了所有@RequestMapping和Handler的映射规则

springmvc启动会扫描所有的@requestmapping  将信息保存到handlerMappins里面

![image-20210120214936206](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210120214936206.png)

所有的请求映射都保存到handlerMapping中,

+ springboot自动配置欢迎页的handlermapping 访问index.html
+ 请求进来 挨个尝试所有的handlermapping看是否有请求信息 
  + 如果有就找这个请求对应的handler
  + 如果没有就找下一个handlerMapping
+ springboot默认配置了哪些
+ ![image-20210120220816421](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210120220816421.png)
  + 需要一些自定义的映射处理,可以自己给容器中放handlermapping,自定义handlermapping



#### 普通参数与基本注解

+ 注解

  @PathVariable @RequestHeader @ModelAttribute @RequestParam @MatrixVariable @RequestBody

+ Servlet API

  WebRequest  ServletRequest  MultipartRequest  HttpSession  javax.servlet.http.PushBuilder  Principal  

  InputStream  Reader  HttpMethod  Locale   TimeZone   ZoneId

+ 复杂参数

  Map  Errors/BindingResult  Model   RedirectAttributes  ServeltResponse   SessionStatus   UriComponentBuilder  ServeltUriComponentBuilder  

+ 自定义对象参数  :  可以自动类型装换与格式化  可以级联封装