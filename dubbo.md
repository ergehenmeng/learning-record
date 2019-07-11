#### generic=true 泛化
> 消费端泛化

```
GenericService.$invoke("method",new String[]{paramType},new Object[]{})
```
> 服务端泛化

```
GenericServiceImpl.$invke(String method){ if(method.equals("say")){...} }
```

> 服务端 Protocol injvm导出

QosProtocolWrapper(统计拦截次数)-ProtocolListenerWrapper(导出监听器)--ProtocolFilterWrapper(包装拦截器)---InjvmProtocol(本地协议导出)

> 服务端 Protocol 注册中心导出

* Protocol导出Invoker对象过程
   * 动态生成Protocol$Adaptive 通过javassit库
   * 在调用export方法时,会生成动态生成RegistryProtocol对象,并通过set方法设置一些属性,例如RegistryFactory,ProxyFactory,Cluster,Protocol(Protocol$Adaptive)
   * 将RegitryProtocol作为构造入参包装为QosProtocolWrapper,ProtocolListenerWrapper,ProtocolFilterWrapper
   * 注意上述三个Wrapper顺序会随机
   * 在RegistryProtocol.export方法中会将protocol协议由registry改为dubbo,并通过Protocol$Adaptive.export进行二次export()
   * 此时会动态生成DubboProtocol,依旧由QosProtocolWrapper,ProtocolListenerWrapper,ProtocolFilterWrapper包装,注意此时只会不包含Protocol$Adaptive
   * DubboProtocol会将Invoker包装为DubboExporter
   * 启动服务暴露端口
   * Netty处理器调用过程MultiMessageHandler->HeartbeatHandler->Dispatcher#dispatch->DecodeHandler->HeaderExchangeHandler->ExchangeHandlerAdapter
> 消费端 Protocol 引用

ProtocolListenerWrapper(导出监听器)-ProtocolFilterWrapper(包装拦截器)-QosProtocolWrapper(统计拦截次数)-RegistryProtocol(注册中心)-DubboProtocol(dubbo协议注册中心导出)

