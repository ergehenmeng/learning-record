#### generic=true 泛化
> 消费端泛化

```
GenericService.$invoke("method",new String[]{paramType},new Object[]{})
```
> 服务端泛化

```
GenericServiceImpl.$invke(String method){ if(method.equals("say")){...} }
```

>服务端 Protocol injvm导出

QosProtocolWrapper(统计拦截次数)-ProtocolListenerWrapper(导出监听器)--ProtocolFilterWrapper(包装拦截器)---InjvmProtocol(本地协议导出)

> 服务端 Protocol 注册中心导出

QosProtocolWrapper(统计拦截次数)-ProtocolListenerWrapper(导出监听器)--ProtocolFilterWrapper(包装拦截器)-RegistryProtocol(注册中心)-DubboProtocol(dubbo协议注册中心导出)

> 消费端 Protocol 引用

ProtocolListenerWrapper(导出监听器)-ProtocolFilterWrapper(包装拦截器)-QosProtocolWrapper(统计拦截次数)-RegistryProtocol(注册中心)-DubboProtocol(dubbo协议注册中心导出)




	