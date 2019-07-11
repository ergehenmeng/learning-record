##SpringBoot
> 1.x **spring.config.location** 添加一个新的配置
> 2.x **spring.config.location** 覆盖默认配置 **spring.config.additional-location** 添加一个新的配置

* `EmbeddedServletContainer` -> WebServer
* `org.springframework.boot.context.embedded`->`org.springframework.boot.web.server`
* `EmbeddedServletContainerCustomizer`->`WebServerFactoryCustomizer`

* `ConfigurationPropertiesBeanRegistrar` 用来注册`EnableConfigurationProperties`指定的`value`的值作为Bean,即Properties配置文件(前提该配置文件Bean必须包含`ConfigurationProperties`注解,否则在注册该bean的时候抛异常)
* `ConfigurationPropertiesBindingPostProcessorRegistrar` 用来注册`ConfigurationPropertiesBindingPostProcessor`Bean,用于处理`ConfigurationProperties`数据绑定
* `@Validated`用在属性配置文件,可以通过JSR303进行非空及其他校验,或者实现`Validator`接口,参考`ConfigurationPropertiesBinder#bind`方法

##Aop增强介绍

* `AnnotationAwareAspectJAutoProxyCreator` 为核心对Bean进行增强的类,实现了`BeanPostProcessor`接口
    * 查找实现`Advisor`的bean
    * 通过`BeanFactoryAspectJAdvisorsBuilder`查找 包含`@AspectJ`注解的bean
    * 根据方法进行匹配,如果类有一个方法匹配上Advisor即可
    * 为AspectJ增强的`Advisor`列表前面添加一个`ExposeInvocationInterceptor` 用来获取`MethodInvocation`信息
    * 将所有与指定类匹配的`Advisor`
    * 在`#postProcessAfterInitialization`方法中对已经初始化后的类进行增强操作
    * 内部通过创建`ProxyFactory`并调用`ProxyFactory#getProxy` 生成增强代理类
    * 内部通过DefaultAopProxyFactory创建AopProxy对象(`JdkDynamicAopProxy`或者`ObjenesisCglibAopProxy`)调用getProxy()
	

* `CglibAopProxy.getProxy()` 用于生成代理对象,在该方法内通过`Enhancer#setCallbackTypes`来设置一组Aop拦截类型,同时设置`Enhancer#setCallbackFilter(ProxyCallbackFilter)`来设置采用哪个拦截类型进行拦截
*  在setCallbackTypes中增强拦截有如下:
    * `DynamicAdvisedInterceptor` 下标为0
    * `StaticUnadvisedExposedInterceptor` `DynamicUnadvisedExposedInterceptor` (静态方法增强)下标 1
    * `SerializableNoOp` 下标 2
    * `StaticDispatcher` `SerializableNoOp` 下标 3
    * `AdvisedDispatcher` 下标 4
    * `EqualsInterceptor` 下标 5
    * `HashCodeInterceptor` 下标 6
* `ProxyCallbackFilter` 用来确定方法调用时应该采用上述那个增强拦截器进行操作
* `DynamicAdvisedInterceptor` 包装拦截器 将方法调用包装成`CglibMethodInvocation` 并调用其proceed()方法
* `CglibMethodInvocation`的父类`ReflectiveMethodInvocation`包含调用链`Advisor`,并且在proceed()方法中进行链式递归调用
