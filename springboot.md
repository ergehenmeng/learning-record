# SpringBoot

## 常用说明

> 1.x **spring.config.location** 为添加一个新的配置
> 2.x **spring.config.location** 覆盖默认配置 **spring.config.additional-location** 添加一个新的配置

* EmbeddedServletContainer->WebServer
* org.springframework.boot.context.embedded->org.springframework.boot.web.server
* EmbeddedServletContainerCustomizer->WebServerFactoryCustomizer



## SpringBean大致生命周期

* `DefaultListableBeanFactory#registerBeanDefinition` 注册BeanDefinition

* `DefaultListableBeanFactory#getMergedBeanDefinition` 合并BeanDefinition (GenericBeanDefinition -> RootBeanDefinition)

* `DefaultListableBeanFactory#resolveBeforeInstantiation` 实例化之前操作 

  * (`InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`) 可以手动实例化bean(bean的生命周期不会继续)

* `DefaultListableBeanFactory#createBeanInstance` 实例化阶段 (构造方法,反射,cglib)

* `DefaultListableBeanFactory#populateBean` 实例化后阶段(属性填充)

  * `InstantiationAwareBeanPostProcessor#postProcessProperties` 对属性进行修改

* `DefaultListableBeanFactory#initializeBean` 初始化前中后

  * `Aware` 接口回调
  * `BeanPostProcessor#postProcessBeforeInitialization` 实例化前操作 
    * `ApplicationContextAwareProcessor`等上下文Aware接口的回调
    * `InitDestroyAnnotationBeanPostProcessor` 中调用`@PostConstruct`方法
  * `InitializingBean#afterPropertiesSet` 接口方法调用
  * 自定义init方法调用
  
* `DefaultListableBeanFactory#preInstantiateSingletons` 初始化所有的单例bean

  * `SmartInitializingSingleton#afterSingletonsInstantiated` 完全初始化后调用 

* `DefaultListableBeanFactory#destoryBean` 销毁前中(单个Bean)

  * `DestructionAwareBeanPostProcessor#postProcessBeforeDestruction` 销毁前操作 `@PreDestroy`
  * `DisposableBean#destroy` 接口方法
  * 自定义的销毁方法

  

  

  

  

  

  

  

  

  





