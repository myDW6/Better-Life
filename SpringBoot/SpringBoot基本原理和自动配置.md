#### @Configuration的区别

```java
ConfigurableApplicationContext context = SpringApplication.run(SpringbootexpApplication.class, args);
//返回IOC
context.getBeanDefinitionNames()  获取所有组件
```

@Configuration (proxyBeanMethods = true) 默认为true,标注的类本身就是被Cglib增强的对象,调用代理对象的方法, 如果为false,拿到的配置类本身就不是代理对象,里面@Bean拿到的组件就不是单实例

Full模式:proxyBeanMethods为true

Lite模式:proxyBeanMethods为false 解决场景:组件依赖

最佳实战:

+ 配置类组件之间无依赖关系用lite模式加快容器启动过程,减少判断
+ 有依赖关系,方法会被调用得到之前的单实例组件,用Full模式



#### @Conditional

![image-20210118211049541](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118211049541.png)



#### 配置绑定

@ConfigurationProperties(prefix), @Compoment

如果是第三方的,没法标注@Compoment,那么在 配置类 上@EnableConfigurationProperties(Bean.class)

​			使用 @ConfigurationProperties 注解的类生效。



#### 自动配置原理

@SpringBootApplication

```java
@SpringBootConfiguration  =>  @Configuration  只是说明是个配置类
@EnableAutoConfiguration
  =>    @AutoConfigurationPackage  
  						=>  @Import({Registrar.class})   (AutoConfigurationPackages的静态内部类)\
  利用Register给容器中导入一系列组件,注解元信息代表了注解标在哪儿,属性值等
  			@Import({AutoConfigurationImportSelector.class})
@ComponentScan(
    excludeFilters = {@Filter(
    type = FilterType.CUSTOM,
    classes = {TypeExcludeFilter.class}
), @Filter(
    type = FilterType.CUSTOM,
    classes = {AutoConfigurationExcludeFilter.class}
)}
  //自定义的两个扫描器
)
```

![image-20210118215114513](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118215114513.png)

```
AutoConfigurationPackages.register(registry, (String[])(new AutoConfigurationPackages.PackageImports(metadata)).getPackageNames().toArray(new String[0]));
//获取包名主类的包名,然后将该包下所有组件注册进容器
```

![image-20210118222124613](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118222124613.png)

AutoConfigurationImportSelector:

![image-20210118222646004](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118222646004.png)

![image-20210118222705994](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118222705994.png)

![image-20210118222922275](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118222922275.png)

获取候选的自动配置

这个方法点进去

![image-20210118223139974](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118223139974.png)

![image-20210118223212329](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118223212329.png)

在spring-boot-autoconfigure-2.4.1.jar/META-INF/spring.factories文件写死

springboot一启动就要加载上述文件中的所有配置类



#### 按需开启

上述场景一共130个自动配置类,实际上条件装配,以aop为例

![image-20210118223956058](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118223956058.png)

需要使用Advice了才会生效

也就是启动时默认加载全部,按照条件装配,最终按需配置



#### 部分场景源码

+ AOP

```java
@Configuration(
    proxyBeanMethods = false  //lite模式
)
@ConditionalOnProperty(
  //配置文件中是否存在spring.aop的配置 
    prefix = "spring.aop",
    name = {"auto"},
    havingValue = "true",
  	//如果存在spring.aop.auto并且值为true 就生效
    matchIfMissing = true //就算没有配 也认为配置了
)

//所以AopAutoConfiguration会生效   
//context.getBeanNamesForType(AopAutoConfiguration.class)可以得到
//org.springframework.boot.autoconfigure.aop.AopAutoConfiguration
public class AopAutoConfiguration {
    public AopAutoConfiguration() {
    }

    @Configuration(
        proxyBeanMethods = false
    ) 
  //当缺少了这个类时,会生效,正好没有org.aspectj.weaver.Advice,所以AspectJ的AOP代理没有生效
    @ConditionalOnMissingClass({"org.aspectj.weaver.Advice"})
  //配置文件是否配置了spring.aop
  //proxy-target-class值是否为true 为true则开启
    @ConditionalOnProperty(
        prefix = "spring.aop",
        name = {"proxy-target-class"},
        havingValue = "true",
        matchIfMissing = true //即使没有配 也认为你配置了 所以开启了AOP
    )
  //简单的AOP有接口有实现类
    static class ClassProxyingConfiguration {
        ClassProxyingConfiguration(BeanFactory beanFactory) {
            if (beanFactory instanceof BeanDefinitionRegistry) {
                BeanDefinitionRegistry registry = (BeanDefinitionRegistry)beanFactory;
                AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
                AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
            }

        }
    }

    @Configuration(
        proxyBeanMethods = false
    )
  //如果有aspectJ里面的Advice才会生效  下面这个类不会生效
    @ConditionalOnClass({Advice.class})
    static class AspectJAutoProxyingConfiguration {
        AspectJAutoProxyingConfiguration() {
        }

        @Configuration(
            proxyBeanMethods = false
        )
        @EnableAspectJAutoProxy(
            proxyTargetClass = true
        )
        @ConditionalOnProperty(
            prefix = "spring.aop",
            name = {"proxy-target-class"},
            havingValue = "true",
            matchIfMissing = true
        )
        static class CglibAutoProxyConfiguration {
            CglibAutoProxyConfiguration() {
            }
        }

        @Configuration(
            proxyBeanMethods = false
        )
        @EnableAspectJAutoProxy(
            proxyTargetClass = false
        )
        @ConditionalOnProperty(
            prefix = "spring.aop",
            name = {"proxy-target-class"},
            havingValue = "false",
            matchIfMissing = false
        )
        static class JdkDynamicAutoProxyConfiguration {
            JdkDynamicAutoProxyConfiguration() {
            }
        }
    }
}
```

![image-20210118232527855](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118232527855.png)

+ Cache

  ```java
  @Configuration(
      proxyBeanMethods = false
  )
  @ConditionalOnClass({CacheManager.class})
  //有没有这个类 Spring-context包下的缓存管理器 存在
  @ConditionalOnBean({CacheAspectSupport.class}) //有没有这个类的组件 从上面可知不存在 所以缓存自动配置类都不生效  下面配置不用考虑
  
  @ConditionalOnMissingBean(
      value = {CacheManager.class},
      name = {"cacheResolver"}
  )
  @EnableConfigurationProperties({CacheProperties.class}) //开启属性绑定
  @AutoConfigureAfter({CouchbaseDataAutoConfiguration.class, HazelcastAutoConfiguration.class, HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class})
  @Import({CacheAutoConfiguration.CacheConfigurationImportSelector.class, CacheAutoConfiguration.CacheManagerEntityManagerFactoryDependsOnPostProcessor.class})
  public class CacheAutoConfiguration {
      public CacheAutoConfiguration() {
      }
  
      @Bean
      @ConditionalOnMissingBean
      public CacheManagerCustomizers cacheManagerCustomizers(ObjectProvider<CacheManagerCustomizer<?>> customizers) {
          return new CacheManagerCustomizers((List)customizers.orderedStream().collect(Collectors.toList()));
      }
  
      @Bean
      public CacheAutoConfiguration.CacheManagerValidator cacheAutoConfigurationValidator(CacheProperties cacheProperties, ObjectProvider<CacheManager> cacheManager) {
          return new CacheAutoConfiguration.CacheManagerValidator(cacheProperties, cacheManager);
      }
  
      static class CacheConfigurationImportSelector implements ImportSelector {
          CacheConfigurationImportSelector() {
          }
  
          public String[] selectImports(AnnotationMetadata importingClassMetadata) {
              CacheType[] types = CacheType.values();
              String[] imports = new String[types.length];
  
              for(int i = 0; i < types.length; ++i) {
                  imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
              }
  
              return imports;
          }
      }
  
      static class CacheManagerValidator implements InitializingBean {
          private final CacheProperties cacheProperties;
          private final ObjectProvider<CacheManager> cacheManager;
  
          CacheManagerValidator(CacheProperties cacheProperties, ObjectProvider<CacheManager> cacheManager) {
              this.cacheProperties = cacheProperties;
              this.cacheManager = cacheManager;
          }
  
          public void afterPropertiesSet() {
              Assert.notNull(this.cacheManager.getIfAvailable(), () -> {
                  return "No cache manager could be auto-configured, check your configuration (caching type is '" + this.cacheProperties.getType() + "')";
              });
          }
      }
  
      @ConditionalOnClass({LocalContainerEntityManagerFactoryBean.class})
      @ConditionalOnBean({AbstractEntityManagerFactoryBean.class})
      static class CacheManagerEntityManagerFactoryDependsOnPostProcessor extends EntityManagerFactoryDependsOnPostProcessor {
          CacheManagerEntityManagerFactoryDependsOnPostProcessor() {
              super(new String[]{"cacheManager"});
          }
      }
  }
  ```

+ web

  ![image-20210118232724099](https://gitee.com/shao_dw/pic/raw/master/upload/image-20210118232724099.png)



+ web

  ```java
  @AutoConfigureOrder(-2147483648)  //配置顺序
  @Configuration(
      proxyBeanMethods = false
  )
  @ConditionalOnWebApplication(
      type = Type.SERVLET
    //原生servelt的web环境 (响应式导入starter-web-flux)  是
  )
  @ConditionalOnClass({DispatcherServlet.class}) //导了web-mvc就有这个类 配置类生效
  @AutoConfigureAfter({ServletWebServerFactoryAutoConfiguration.class}) 
  //在ServletWebServerFactoryAutoConfiguration自动配置完之后再配置
  public class DispatcherServletAutoConfiguration {
      public static final String DEFAULT_DISPATCHER_SERVLET_BEAN_NAME = "dispatcherServlet";
      public static final String DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME = "dispatcherServletRegistration";
  
      public DispatcherServletAutoConfiguration() {
      }
  
      @Order(2147483637)
      private static class DispatcherServletRegistrationCondition extends SpringBootCondition {
          private DispatcherServletRegistrationCondition() {
          }
  
          public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
              ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
              ConditionOutcome outcome = this.checkDefaultDispatcherName(beanFactory);
              return !outcome.isMatch() ? outcome : this.checkServletRegistration(beanFactory);
          }
  
          private ConditionOutcome checkDefaultDispatcherName(ConfigurableListableBeanFactory beanFactory) {
              boolean containsDispatcherBean = beanFactory.containsBean("dispatcherServlet");
              if (!containsDispatcherBean) {
                  return ConditionOutcome.match();
              } else {
                  List<String> servlets = Arrays.asList(beanFactory.getBeanNamesForType(DispatcherServlet.class, false, false));
                  return !servlets.contains("dispatcherServlet") ? ConditionOutcome.noMatch(this.startMessage().found("non dispatcher servlet").items(new Object[]{"dispatcherServlet"})) : ConditionOutcome.match();
              }
          }
  
          private ConditionOutcome checkServletRegistration(ConfigurableListableBeanFactory beanFactory) {
              Builder message = this.startMessage();
              List<String> registrations = Arrays.asList(beanFactory.getBeanNamesForType(ServletRegistrationBean.class, false, false));
              boolean containsDispatcherRegistrationBean = beanFactory.containsBean("dispatcherServletRegistration");
              if (registrations.isEmpty()) {
                  return containsDispatcherRegistrationBean ? ConditionOutcome.noMatch(message.found("non servlet registration bean").items(new Object[]{"dispatcherServletRegistration"})) : ConditionOutcome.match(message.didNotFind("servlet registration bean").atAll());
              } else if (registrations.contains("dispatcherServletRegistration")) {
                  return ConditionOutcome.noMatch(message.found("servlet registration bean").items(new Object[]{"dispatcherServletRegistration"}));
              } else {
                  return containsDispatcherRegistrationBean ? ConditionOutcome.noMatch(message.found("non servlet registration bean").items(new Object[]{"dispatcherServletRegistration"})) : ConditionOutcome.match(message.found("servlet registration beans").items(Style.QUOTE, registrations).append("and none is named dispatcherServletRegistration"));
              }
          }
  
          private Builder startMessage() {
              return ConditionMessage.forCondition("DispatcherServlet Registration", new Object[0]);
          }
      }
  
      @Order(2147483637)
      private static class DefaultDispatcherServletCondition extends SpringBootCondition {
          private DefaultDispatcherServletCondition() {
          }
  
        //条件进出规则
          public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMeta	data metadata) {
              Builder message = ConditionMessage.forCondition("Default DispatcherServlet", new Object[0]);
              ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
              List<String> dispatchServletBeans = Arrays.asList(beanFactory.getBeanNamesForType(DispatcherServlet.class, false, false));
              if (dispatchServletBeans.contains("dispatcherServlet")) {
                  return ConditionOutcome.noMatch(message.found("dispatcher servlet bean").items(new Object[]{"dispatcherServlet"}));
              } else if (beanFactory.containsBean("dispatcherServlet")) {
                  return ConditionOutcome.noMatch(message.found("non dispatcher servlet bean").items(new Object[]{"dispatcherServlet"}));
              } else {
                  return dispatchServletBeans.isEmpty() ? ConditionOutcome.match(message.didNotFind("dispatcher servlet beans").atAll()) : ConditionOutcome.match(message.found("dispatcher servlet bean", "dispatcher servlet beans").items(Style.QUOTE, dispatchServletBeans).append("and none is named dispatcherServlet"));
              }
          }
      }
  
      @Configuration(
          proxyBeanMethods = false
      )
      @Conditional({DispatcherServletAutoConfiguration.DispatcherServletRegistrationCondition.class})
      @ConditionalOnClass({ServletRegistration.class})
      @EnableConfigurationProperties({WebMvcProperties.class}) //开启属性绑定
      @Import({DispatcherServletAutoConfiguration.DispatcherServletConfiguration.class})
      protected static class DispatcherServletRegistrationConfiguration {
          protected DispatcherServletRegistrationConfiguration() {
          }
  
          @Bean(
              name = {"dispatcherServletRegistration"}
          )
          @ConditionalOnBean(
              value = {DispatcherServlet.class},
              name = {"dispatcherServlet"}
          )
          public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet, WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
              DispatcherServletRegistrationBean registration = new DispatcherServletRegistrationBean(dispatcherServlet, webMvcProperties.getServlet().getPath());
              registration.setName("dispatcherServlet");
              registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
              multipartConfig.ifAvailable(registration::setMultipartConfig);
              return registration;
          }
      }
  
      @Configuration(
          proxyBeanMethods = false
      )
      @Conditional({DispatcherServletAutoConfiguration.DefaultDispatcherServletCondition.class})
      @ConditionalOnClass({ServletRegistration.class})
      @EnableConfigurationProperties({WebMvcProperties.class}) //开启属性绑定
      protected static class DispatcherServletConfiguration {
          protected DispatcherServletConfiguration() {
          }
  
          @Bean(
              name = {"dispatcherServlet"}
          )
        //配置核心DispatcherServlet
          public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
              DispatcherServlet dispatcherServlet = new DispatcherServlet();//自己new 从webMvcProperties取值赋值
              dispatcherServlet.setDispatchOptionsRequest(webMvcProperties.isDispatchOptionsRequest());
              dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
              dispatcherServlet.setThrowExceptionIfNoHandlerFound(webMvcProperties.isThrowExceptionIfNoHandlerFound());
              dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
              dispatcherServlet.setEnableLoggingRequestDetails(webMvcProperties.isLogRequestDetails());
              return dispatcherServlet;
          }
  
          @Bean
          @ConditionalOnBean({MultipartResolver.class}) //容器中有这个类型的组件 目前有 
        //context.getBeanNamesForType(MultipartResolver.class) 可以返回
          @ConditionalOnMissingBean(
              name = {"multipartResolver"} //没有这个名字的组件    
            //表示什么意思?  容器中有了一个MultipartResolver但是名字不叫multipartResolver  我就从容器中找到
          )
        //此处 对@Bean标注的方法传入的参数 会从Spring容器中找
          public MultipartResolver multipartResolver(MultipartResolver resolver) {
            //配置springmvc配置文件上传解析器名字必须叫这个multipartResolver 但是很多人不知道这个默认行为 名字不叫这个multipartResolver, 防止用户配置的文件上传解析器不符合规范  把自定义的解析器命名为multipartResolver(方法名)
              return resolver;
          }
      }
  }
  ```

+ dsa

#### 总结

+ springboot先加载所有的自动配置类 xxxAutoConfiguration
+ 每个自动配置类,按照条件进行生效,默认都会绑定配置文件指定的值,从xxxProperties中拿,xxxProperties和配置文件进行了绑定,并且使用了@EnableConfigurationProperties标注
+ 生效的配置类就会给容器中装配很多组件
+ 只要容器中有了这些组件,相当于这些功能也就具备了
+ 定制化配置
  + 直接@Bean替换底层组件
  + 看这个组件对应的xxxProperties的值,对应修改



#### 最佳实践

+ 引入场景依赖

+ 查看自动配置了哪些

  + 上述方式分析
  + debug=true 自动配置报告

+ 是否需要修改

  + 参照文档 https://docs.spring.io/spring-boot/docs/current/reference/html/appendix-application-properties.html#common-application-properties

  + 上述方式找前缀

  + 自定义添加组件或者替换 @Bean @Component

  + 自定义器(待学习) xxxCustomizer

    

    

    

    

  