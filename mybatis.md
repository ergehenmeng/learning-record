
* `ConfigurationCustomizer` 自定义mybatis配置信息
* `SqlSourceBuilder` 创建#{}解析对象,内部通过`GenericTokenParser`解析,并将解析后的信息保存为`ParameterMapping`对象
* `GenericTokenParser` 解析SQL中#{}
* `ParameterMapping` sql中占位符描述类#{}
* `MappedTypes`接口上注解`@MappedTypes`用来表示该类处理哪些java类型,如果接口上注解`@MappedJdbcTypes` 用来表示该类处理哪些数据库类型
* `MapperProxyFactory` Mapper接口方法描述对象,每个Mapper接口都会生成一个该对象,并交由Configuration#mapperRegistry#knownMappers成员变量来维护
* `SqlSessionTemplate` 基于SqlSession的模板类,内部采用动态代理方式对原始SqlSession进行拦截,具体代理类`SqlSessionInterceptor`
* `SqlSessionInterceptor` 再次创建代理对象SqlSession,如果该SqlSession代理对象由Spring管理则,则释放事务,如果不是由spring管理的则提交事务并关闭连接
* `TypeHandlerRegistry` 用于保存TypeHandler对象
```java

private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //调用方法前会真正创建SqlSession对象,SqlSession对象由sqlSessionFactory的默认实现类DefaultSqlSessionFactory实现
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }


``` 
* `MapperFactoryBean` 创建基于Mapper接口的工厂Bean,内部通过SqlSessionTemplate.getMapper(Class<T> cls)获取真正的Mapper,而getMapper方法内部通过动态代理生成接口的实现类
* `MapperProxy` 该类实现InvocationHandler接口,用于拦截Mapper代理类的所有方法
* `MappedStatement` 用于保存sql相关所有属性,该类通过Mapper.methodName作为key,保存到Configuration中的mappedStatements里面
* `MapperMethod` 在`MapperProxy#invoke`中创建,每个方法均包含一个`MapperMethod`并缓存到`MapperProxy`中,主要保存方法的调用类型(`SqlCommand`)入参,出参,分页(`MethodSignature`)等信息及真实调用逻辑.同时确认方式是insert,update,delete,select等类型

* `ConnectionLogger` 创建Connection打印日志,当调用prepareStatement方法时,会打印sql日志,同时创建`PreparedStatementLogger`对象,当调用createStatement方法时,会创建`StatementLogger`对象
* `PreparedStatementLogger` 执行PreparedStatement打印入参或者执行日志(动态代理)
* `StatementLogger` 创建Statement打印SQL日志(动态代理)
* `ResultSetLogger` 结果集日志打印(动态代理)
* `SpringManagedTransaction` spring事务管理器(针对mybatis)
* `SimpleExecutor` SQL执行器,执行增删改查
* `DefaultParameterHandler` setParameter(index,value)
* `SelectKeyGenerator` selectKey标签 内部直接执行一次查询并通过MetaObject反射设置id
* `PreparedStatementHandler` 用来执行PreparedStatement增删改查并返回结果集
* `DefaultResultSetHandler` 将ResultSet转换为相应的集合或类


```
EnumTypeHandler
```
> 在入参使用枚举时,parameterType没有指定枚举类型,需要在```#{menu,javaType=com.fanyin...,typeHanlder=org.apache.ibatis.type.EnumOrdinalTypeHandler}```中指定否则注册枚举处理类型时,枚举类型为空会抛异常

* **自定义枚举**

继承 ```BaseTypeHandler```类,如果需要处理的枚举类型一致,只需要使用@MappedTypes传入多个需要解析的枚举类即可


* 与spring进行整合时,basePackage支持 **,** **;** **空格** **回车符** 进行分割多个包
* `MapperFactoryBean` 在spring中用来获取单个Mapper的类
* `MapperScannerConfigurer` 实现BeanDefinitionRegistryPostProcessor接口在postProcessBeanDefinitionRegistry方法中通过`ClassPathMapperScanner`来扫描Mapper的,并将扫描到的Mapper接口生成基于MapperFactoryBean类代理类,交由spring管理,`MapperScan`就是采用此对象进行处理的
* pojo类采用别名 可以在类名添加`@Alias` 如果在配置文件中指定了mybatis.type-aliases-package(application.properties),则会扫描对应包下的pojo,如果发现有`@Alias`则使用此别名,否则使用类名的simpleName作为别名
* spring与mybatis集成时,扫描xml中的namespace并将其作为类加载到内存,同时将Mapper添加到`MapperRegistry`中(如果不存在该类,则忽略,因为namespace不是必填的,mybatis会通过Mapper接口再次添加一次)
* 在springboot中集成mybatis时,`MybatisAutoConfiguration` 通过`SqlSessionFactoryBean`创建`SqlSessionFactory`并且在getObject()中调用afterPropertiesSet()方法来解析mybatis配置,扫描Mapper.xml文件等操作,并将xml中update,insert,delete,select等标签转换为`MappedStatement`对象并缓存起来, id描述为Mapper类的全路径`.`标签ID
* `@MapperScan` 扫描Mapper类,并将其转换为`MapperFactoryBean`对象,通过getObject()调用configuration.getMapper得到代理后的Mapper类
* `ParamNameResolver`用来解析@Param参数并生成Mapper的方法参数数组
* `MapperMethod` 根据标签类型的不同,调用sqlSession.insert,sqlSession.update等,传入方法名称与参数
* `MapperRegistry` 通过扫描xml为每个Mapper生成`MapperProxyFactory`对象, `MapperProxyFactory`中创建Mapper的代理对象`MapperProxy`,在invoke方法中调用SqlSession方法进行查询sql处理
* **存储过程** {call my_product(#{inName,jdbcType=VARCHAR,mode=IN})}
* **存储过程游标** {call my_product(#{inName,jdbcType=VARCHAR,mode=IN},#{count,jdbcType=CURSOR,mode=OUT})}

* **settting** 
  * `cacheEnabled` 映射器中是否开启全局缓存 true
  * `lazyLoadingEnabled` 是否开启延迟加载(关联查询时使用),特定关联时可以通过fetchType来覆盖全局设置 false
  * `aggressiveLazyLoading` 当启用时,对任意延迟属性的调用会使带有延迟加载属性的对象全部加载,反之,每种每种属性都会按需加载 true
  * `multipeResultSetsEnabled` 是否允许单一语句返回多结果集(需要驱动支持) true
  * `useColumnLabel` 使用列标签代替列表(不同驱动结果不一样) true
  * `useGeneratedKey` 允许jdbc支持自动生成主键 false
  * `autoMapperBehavior` 指定mybatis应如何自动映射列到字段或属性 NONE:取消自动映射,PARTIAL:只会自动映射没有定义嵌套结果集映射的结果集 FULL:会自动映射任意复杂的结果集(无论是否嵌套) PARTIAL
  * `defaultExecutorType` 配置默认执行器 SIMPLE:普通执行器 REUSE:执行器会重用预处理语句,BATCH:执行器重用语句并执行批量更新
  * `defaultStatementTimeout` 超时时间,默认驱动的超时时间
  * `safeRowBoundEnabled` 允许嵌套语句中使用分页 false 
  * `mapUndersocreToCamelCase` 是否开启驼峰映射 true
  * `localCacheScope` mybatis利用本地缓存机制防止循环引用和加速重复嵌套查询,默认SESSION,这种情况下会缓存一个会话中所有执行的查询,若设置为STATEMENT,本地会话仅用在语句执行上,对相同的SqlSession的不同调用将不会共享数据
  * `jdbcTypeForNull` 当没有为参数提供特定JDBC类型时,为空值指定jdbc类型,NULL,VARCHAR,OTHER 默认OTHER
  * `lazyLoadTriggerMethods` 指定对象的方法触发一次缓存延迟,如果是列表以逗号分割,默认:equals,clone,hashCode,toString
  * `defaultScriptingLanguage` 指定动态SQL的默认语言 XMLDynamicLanguageDriver
  * `callSetterOnNulls` 当结果集为Null的时候调用映射对象的setter(map的put)方法,注意int,boolean 不能设置为null false
  * `logPrefix` 指定mybatis增加到日志名称的前缀
  * `logImpl` 指定mybatis日志的具体实现,未指定时,自动查询,SLF4J,LOG4J,LOG4J2,JDK_LOGGING,COMMONS_LOGGIN,STDOUT_LOGGING NO_LOGGING 
  * proxyFactory 指定mybatis创建具有延迟能力的对象所用的代理工具,CGLIB,JAVASSIST,3.3.0及以上JAVASSIST 以下CGLIB
* **TypeHandler** 将javaType转换为jdbcType,或者将jdbcType转换为javaType,自定义可配置为全局也可针对某个Mapper标签
* 枚举类型映射
  * `EnumTypeHandler` 字符串映射到数据库,通过调用name()方法和valueOf方法进行数据转换
  * `EnumOrdinalTypeHandler` 通过下标映射到数据库,通过ordinal()方法进行数据转换
  * 自定义映射同样实现`TypeHandler`接口即可
* TransactionManager事务分三种
  * JDBC 采用jdbc方式管理事务,通过编码控制
  * MANAGED 容器式事务,在JNDI中使用
  * 自定义事务 特殊情况应用
* DataSource链接方式配置
  * UNPOOLED 非连接池数据库 UnpooledDataSource
  * POOLED 连接池数据库 PooledDataSource
  * JNDI JNDI数据源 JndiDataSourceFactory
  * 其他 例如使用DBCP数据源等 需要自定义
* **association** 一对一关系
* **collection** 一对多关系
* **discriminator** 鉴别器,根据实际选择的那个类作为实例,允许特定条件关联不通的结果集
#### cache标签
* `eviction`：缓存的回收策略 默认LRU
  * LRU - 最近最少使用，移除最长时间不被使用的对象
  * FIFO - 先进先出，按对象进入缓存的顺序来移除它们
  * SOFT - 软引用，移除基于垃圾回收器状态和软引用规则的对象
  * WEAK - 弱引用，更积极地移除基于垃圾收集器和弱引用规则的对象
* `flushInterval`：缓存刷新间隔 缓存多长时间清空一次，默认不清空，设置一个毫秒值
* `readOnly` 是否只读 
  * true：mybatis认为所有从缓存中获取数据的操作都是只读操作,不会修改数据。mybatis 会将缓冲数据中的引用直接交给用户,速度快,不安全
  * false: 读写(默认)：mybatis觉得获取的数据可能会被修改,mybatis会利用序列化&反序列化的技术克隆一份新的数据给你.安全，速度相对慢
* `size` 缓存存放多少个元素
* `type` 指定自定义缓存的全类名(实现Cache接口即可)
* `flushCache` 是否刷新缓存 一般用在insert,delete,update
* `useCache` 是否开启缓存 一般用在select

