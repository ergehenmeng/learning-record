####SpringMVC执行流程

* 如果是文件上传,则会将`HttpServletRequest`包装为`MultipartHttpServletRequest`
* 通过`HandlerMapping`生成`HandlerExecutionChain`
   * `DispatcherServlet`注册了多个`HandlerMapping`,核心类`RequestMappingHandlerMapping`
   * 在`RequestMappingHandlerMapping`中通过解析request中的url匹配最合适的`HandlerMethod`
       * `HandlerMethod`由`MappingRegistry`维护,在初始化完成后会扫描Controller中`RequestMapping`注解,组装成url与`HandlerMethod`映射关系
       * 将url与HandlerMethod对应关系由Map进行维护
       * `HandlerMethod`是`@RequestMapping`注解标示的方法的信息类
   * 将得到`HandlerMethod`与匹配成功`HandlerInterceptor`包装为`HandlerExecutionChain`并返回
* 通过`HandlerExecutionChain`得到`HandlerAdapter`
   * `DispatcherServlet`注册了多个`HandlerAdapter`
   * 通过HandlerAdapter#supports`来确定该Adapter是否支持处理本次请求
   * 核心实现类`RequestMappingHandlerAdapter` 用于处理`HandlerMethod`类型的请求
* 执行`HandlerExecutionChain`中拦截器的前置处理
* 执行`HandlerAdapter#handle`核心调用方法,并返回`ModelAndView`
   * 根据`HandlerMethod` 创建`WebDataBinderFactory`(数据绑定)
   * 创建`ModelFactory` 处理`@ModelAttribute`注解
   * 将`HandlerMethod`包装为`ServletInvocableHandlerMethod`类
   * 解析参数调用真实的方法并返回`ModelAndView`
* 如果`ModelAndView` 中不包含视图时,通过`DefaultRequestToViewNameTranslator`来解析request生成默认的viewName
* 执行`HandlerExecutionChain`中拦截器的后置处理
* 执行最终处理(在上述接口调用之后)
   * 如果有异常产生,则通过`HandlerExceptionResolver`异常解析器解析并返回`ModelAndView`,核心类`ExceptionHandlerExceptionResolver`
       * `ExceptionHandlerExceptionResolver`在实例化后会扫描有所有的`@ControllerAdvice`类,
       * 扫描这些类中包含`@ExceptionHandler`的方法,并将其包装为一个`ExceptionHandlerMethodResolver`类
       * 初始化`HandlerMethodArgumentResolver`参数解析器
       * 初始化`HandlerMethodReturnValueHandler`返回值处理器
       * 解析异常信息并包装为`ServletInvocableHandlerMethod`
       * 解析参数后通过反射调用要处理异常的方法
       * 解析返回值信息
   * 如果`ModelAndView`中包含视图信息则进行渲染
       * 如果视图名不为空,则通过是`ViewResolver`视图解析器解析出相应的`View`
       * 渲染View
   * 执行`HandlerExecutionChain`中拦截器最终方法
* 清空上传的临时文件(如果有)