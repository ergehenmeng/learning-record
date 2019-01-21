* **入参枚举说明**

``` 
EnumOrdinalTypeHandler 
```

```
EnumTypeHandler
```
> 在入参使用枚举时,parameterType没有指定枚举类型,需要在```#{menu,javaType=com.fanyin...,typeHanlder=org.apache.ibatis.type.EnumOrdinalTypeHandler}```中指定否则注册枚举处理类型时,枚举类型为空会抛异常

* **自定义枚举**

继承 ```BaseTypeHandler```类,如果需要处理的枚举类型一致,只需要使用@MappedTypes传入多个需要解析的枚举类即可
