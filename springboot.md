##SpringBoot
> 1.x **spring.config.location** 添加一个新的配置
> 2.x **spring.config.location** 覆盖默认配置 **spring.config.additional-location** 添加一个新的配置

* `EmbeddedServletContainer` -> WebServer
* `org.springframework.boot.context.embedded`->`org.springframework.boot.web.server`
* `EmbeddedServletContainerCustomizer`->`WebServerFactoryCustomizer`

* `ConfigurationPropertiesBeanRegistrar` 用来注册`EnableConfigurationProperties`指定的`value`的值作为Bean,即Properties配置文件(前提该配置文件Bean必须包含`ConfigurationProperties`注解,否则在注册该bean的时候抛异常)
* `ConfigurationPropertiesBindingPostProcessorRegistrar` 用来注册`ConfigurationPropertiesBindingPostProcessor`Bean,用于处理`ConfigurationProperties`数据绑定
* `@Validated`用在属性配置文件,可以通过JSR303进行非空及其他校验,或者实现`Validator`接口,参考`ConfigurationPropertiesBinder#bind`方法
